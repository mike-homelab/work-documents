# Enterprise AI Assistant — Architecture & Implementation Plan

**Version:** 6.0 (Comprehensive — Merged Architecture, Resilience Patterns & Distributed Tracing)  
**Target:** Azure AI Foundry Agentic Platform with Next.js Frontend  
**Deployment:** Azure APIM → Azure App Service (FastAPI, auto-scaled) → Azure Service Bus → Azure AI Foundry → Azure Cosmos DB → Azure Web PubSub → Azure AI Search  

---

## 1. What We Are Building

An enterprise chat assistant for **Product Design Engineers** that helps look up specific product parts (by part number) and ask general product questions. The system:

*   Authenticates users via **Azure Entra ID (MSAL)**, validated at **Azure API Management (APIM)** layer
*   Runs a **deterministic pre-processing layer** in the Worker — regex-based part number detection, normalization, and direct API calls *before* LLM involvement (faster, cheaper, more predictable)
*   Detects whether the user is asking about a specific part number or a general product question
*   Normalizes part numbers (handles typos, spacing, casing)
*   Looks up parts via the **Product Description API** (existing REST API)
*   Falls back to the **Product GPT API** (existing external AI app) when no part is found or the query is general
*   Grounds Product GPT answers with **Azure AI Search** data sourced from a web crawler of the company website
*   Summarizes verbose responses through an **Editor Model (GPT-4o-mini)** — the orchestrator decides when summarization is needed based on the user's question
*   Streams real-time status updates to the frontend via **Azure Web PubSub** (managed WebSocket — survives instance scale-down)
*   Creates AI Foundry threads **lazily** — only when the first message requiring LLM reasoning arrives (not at session creation)
*   Handles follow-up questions, topic switches, disambiguation, and N queued messages (FIFO)
*   Enforces **resilience patterns** — explicit timeouts, retry policies, and circuit breakers for all external dependencies
*   Implements **end-to-end distributed tracing** via OpenTelemetry with Azure Monitor integration
*   Auto-scales horizontally: 1 new App Service instance per 10 queued messages
*   Expires sessions after 10 minutes of inactivity

---

## 2. System Architecture Diagram

```text
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND (Next.js)                                  │
│  ┌──────────────┐  ┌────────────────────────┐  ┌──────────────────────────────┐ │
│  │ MSAL Auth    │  │ REST: POST https://    │  │ WebSocket: wss://            │ │
│  │ (Entra ID)   │  │ apim.company.com/api/  │  │ pubsub.company.com/ws/{sid}  │ │
│  │              │  │ v1/chat/send           │  │ ← status updates, responses  │ │
│  └──────────────┘  └────────────────────────┘  └──────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
         │                    │                          ▲
         │ JWT                │ HTTPS                    │ WebSocket (managed)
         ▼                    ▼                          │
┌─────────────────────────────────────────┐    ┌────────────────────────────┐
│   AZURE API MANAGEMENT (APIM)           │    │ AZURE WEB PUBSUB           │
│   • validate-jwt policy (Entra ID)      │    │ (Managed WebSocket Service) │
│   • Rate limiting (per user/IP)         │    │ • Survives instance         │
│   • API versioning (/api/v1/*)          │    │   scale-down                │
│   • Request/response logging            │    │ • Hub: "chat"               │
│   • Subscription key injection          │    │ • Group per session_id      │
│   • CORS policy                         │    └────────────────────────────┘
│   • Backend: App Service pool           │               ▲
└─────────────────────────────────────────┘               │ Worker pushes
         │                                                │ status/response
         │ HTTPS + Ocp-Apim-Subscription-Key              │ via SDK
         ▼                                                │
┌──────────────────────────────────────────────────────────┴──────────────────────┐
│         AZURE APP SERVICE — AUTO-SCALED POOL (FastAPI)                           │
│         (Scale rule: +1 instance per 10 Service Bus queued messages)             │
│         (OpenTelemetry SDK instrumented — all spans exported to Azure Monitor)   │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │ Gateway Layer (runs in main process — receives HTTP from APIM)             │  │
│  │  • Validates Ocp-Apim-Subscription-Key header (verifies request came      │  │
│  │    through APIM — lightweight string check, NOT full JWT parsing)         │  │
│  │  • Extracts user_id from X-User-Id header (injected by APIM policy       │  │
│  │    after JWT validation)                                                   │  │
│  │  • Generates correlation_id (UUID v4) for distributed tracing             │  │
│  │  • Session lifecycle management (create / validate / expire)              │  │
│  │  • Enqueue message → Azure Service Bus (SDK call, not HTTP)               │  │
│  │  • REST endpoints: /api/v1/chat/send, /api/v1/session/create, etc.       │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                              │                                                   │
│                   ┌──────────┴──────────┐                                        │
│                   │ IN-PROCESS BOUNDARY  │  ← NOT an HTTP call!                  │
│                   │ Gateway enqueues to  │  ← Gateway and Worker share the same  │
│                   │ Service Bus via SDK  │     Python process. Session validation,│
│                   │ Worker dequeues via  │     Cosmos DB reads, Service Bus ops   │
│                   │ Service Bus via SDK  │     are all internal SDK/function calls│
│                   └──────────┬──────────┘                                        │
│                              │                                                   │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │ Worker Layer (background asyncio task in same process)                     │  │
│  │  • Dequeues messages from Service Bus (SDK call)                          │  │
│  │  • Deterministic Pre-Processing:                                          │  │
│  │    - Regex part number detection → normalize → direct API call            │  │
│  │    - Off-topic / greeting short-circuit                                   │  │
│  │  • Loads session context from Cosmos DB (SDK call)                        │  │
│  │  • Lazy Thread Creation: creates AI Foundry thread on first LLM-bound    │  │
│  │    message only (not at session creation)                                 │  │
│  │  • Calls Azure AI Foundry Agent Service (SDK call)                       │  │
│  │  • Circuit breakers + timeout/retry on all external calls                 │  │
│  │  • Pushes status/response to Azure Web PubSub (SDK call)                 │  │
│  │  • Saves conversation + context to Cosmos DB (SDK call)                  │  │
│  │  • On SIGTERM: stops dequeuing, finishes current message, exits cleanly  │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────┘
         │                              │                          │
         ▼                              ▼                          ▼
┌──────────────┐    ┌─────────────────────────────────────┐  ┌──────────────────┐
│Azure Service │    │ Azure AI Foundry Agent Service       │  │ Azure Cosmos DB  │
│Bus (Standard)│    │  ┌─────────────────────────────┐     │  │  • conversations │
│  • chat-queue│    │  │ Orchestrator Agent (GPT-4o)  │     │  │  • session_context│
│  • Sessions  │    │  │                              │     │  └──────────────────┘
│    (FIFO per │    │  │  Tools:                      │     │
│    session)  │    │  │  ├─ product_description_api  │     │
│  • DLQ       │    │  │  ├─ product_gpt_api          │     │
└──────────────┘    │  │  └─ search_knowledge_base    │     │
                    │  │      (Azure AI Search)        │     │
                    │  └─────────────────────────────┘     │
                    │                                       │
                    │  ┌─────────────────────────────┐     │
                    │  │ Editor Agent (GPT-4o-mini)   │     │
                    │  │  • Summarizes verbose output  │     │
                    │  │  • Filters for design engineer│     │
                    │  │    audience                   │     │
                    │  │  • Activated by orchestrator   │     │
                    │  │    when needed                │     │
                    │  └─────────────────────────────┘     │
                    └─────────────────────────────────────┘
                                 │         │
                    ┌────────────┘         └──────────────┐
                    ▼                                      ▼
        ┌───────────────────┐               ┌─────────────────────────┐
        │Product Description│               │  Product GPT API        │
        │API (Existing REST)│               │  (Existing External AI) │
        └───────────────────┘               └─────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    OFFLINE / SCHEDULED PIPELINE                              │
│  Azure AI Search Indexer (built-in web crawler)                             │
│    → Crawls company website on schedule                                     │
│    → Chunks HTML pages into semantic segments                               │
│    → Creates vector embeddings (Azure OpenAI text-embedding-3-large)        │
│    → Stores in Azure AI Search Index: "product-knowledge-index"             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY LAYER                                        │
│  OpenTelemetry SDK (auto + custom instrumentation)                          │
│    → Traces: Gateway → Service Bus → Worker → Pre-Process → AI Foundry     │
│    → Metrics: token usage, circuit breaker state, routing path stats        │
│    → Exporter: Azure Monitor (Application Insights)                         │
│    → Correlation ID propagated end-to-end                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.1 — Internal vs. External Calls: What Is and Isn't an HTTP Call

| Call | Type | Explanation |
|------|------|-------------|
| APIM → App Service | External HTTP | This is the ONLY HTTP hop for user requests. APIM forwards to the App Service backend pool. |
| Gateway → Session Validation | Internal function call | `session_service.validate(session_id)` — Python function in the same process. Zero network. |
| Gateway → Service Bus Enqueue | External SDK call | Uses `azure-servicebus` SDK. This is an AMQP connection, not HTTP per-request. The SDK maintains a persistent connection. |
| Worker → Service Bus Dequeue | External SDK call | Same persistent AMQP connection. Worker pulls messages as they arrive. |
| Worker → Cosmos DB | External SDK call | Uses `azure-cosmos` SDK. HTTPS under the hood, but connection is pooled and reused. |
| Worker → AI Foundry | External SDK call | Uses `azure-ai-projects` SDK. HTTPS under the hood, but connection is pooled. |
| Worker → Web PubSub | External SDK call | Uses `azure-messaging-webpubsubservice` SDK. Single REST call to push a message to a group. |
| Gateway ↔ Worker | NOT a call at all | They share the same Python process. Gateway enqueues to Service Bus, Worker dequeues. There is no direct Gateway→Worker call. They are decoupled by the queue. |

> **Authentication is NOT an HTTP call to your own service.** APIM validates the JWT using its built-in `validate-jwt` policy (cached JWKS keys from Entra ID). By the time the request reaches your FastAPI app, auth is already done. FastAPI only checks the `Ocp-Apim-Subscription-Key` header (a simple string comparison) to verify the request came through APIM.

---

## 3. Authentication & Authorization

Users authenticate via **Azure Entra ID (formerly Azure Active Directory)** using the MSAL flow. JWT validation happens at the **APIM layer**, not in FastAPI.

### Authentication Flow

1.  **Frontend (Next.js):** Integrates `@azure/msal-react` to initiate the login flow. User signs in via Azure Entra ID and receives an access token (JWT).
2.  **Azure APIM:** Every incoming API request includes the JWT in the `Authorization: Bearer <token>` header. APIM's `validate-jwt` policy validates the token signature, audience, expiry, and tenant using cached JWKS keys from Entra ID.
3.  **Header Injection:** After validation, APIM extracts `user_id` (oid claim) from the JWT and injects:
    *   `X-User-Id: <entra_object_id>` — the authenticated user's identity
    *   `Ocp-Apim-Subscription-Key: <internal_key>` — proves the request came through APIM
4.  **FastAPI Gateway:** Validates the `Ocp-Apim-Subscription-Key` (a lightweight string comparison — NOT full JWT parsing) and reads `X-User-Id` from the header. Trusts that APIM already validated the JWT.
5.  **No Custom Auth:** Zero custom user databases or password management — fully delegated to Azure Entra ID and APIM.

### APIM Policy Configuration

```xml
<!-- Inbound policy on all /api/v1/* operations -->
<inbound>
    <validate-jwt header-name="Authorization" 
                  failed-validation-httpcode="401"
                  failed-validation-error-message="Unauthorized">
        <openid-config url="https://login.microsoftonline.com/{tenant_id}/v2.0/.well-known/openid-configuration" />
        <audiences>
            <audience>{app_client_id}</audience>
        </audiences>
        <issuers>
            <issuer>https://login.microsoftonline.com/{tenant_id}/v2.0</issuer>
        </issuers>
    </validate-jwt>
    
    <!-- Extract user_id from validated JWT and pass to backend -->
    <set-header name="X-User-Id" exists-action="override">
        <value>@(context.Request.Headers.GetValueOrDefault("Authorization","").AsJwt()?.Claims["oid"]?.FirstOrDefault())</value>
    </set-header>
    
    <!-- Rate limiting: 30 requests per minute per user -->
    <rate-limit-by-key calls="30" 
                       renewal-period="60" 
                       counter-key="@(context.Request.Headers.GetValueOrDefault("Authorization","").AsJwt()?.Claims["oid"]?.FirstOrDefault())" />
</inbound>
```

### Azure Configuration
*   Register the application in **Azure Entra ID → App Registrations**.
*   Configure API permissions and expose scopes as needed.
*   Use **Managed Identity** for all service-to-service communication (Gateway → Service Bus, Workers → Cosmos DB, Workers → AI Foundry, Workers → Web PubSub).
*   Store external API keys in **Azure Key Vault** (Product Description API, Product GPT API).
*   Configure **CORS** on APIM to allow only the Next.js frontend domain.

---

## 4. End-to-End Message Flow (First Message to Final Response)

This section walks through the complete lifecycle of a user interaction, from the moment the user opens the app to the final response delivery. Every step is numbered for Copilot traceability.

### 4.1 — User Opens the App & Authenticates

```
STEP 1: User navigates to the Next.js app
STEP 2: Next.js app checks if user has a valid MSAL token in browser storage
STEP 3: If no token → redirect to Azure Entra ID login page
STEP 4: User signs in → Entra ID returns JWT access token to the browser
STEP 5: Next.js stores the token via @azure/msal-react and includes it 
        in all subsequent API calls as: Authorization: Bearer <jwt>
```

### 4.2 — Session Creation (Lazy — No AI Foundry Thread Yet)

```
STEP 6:  Next.js calls POST https://apim.company.com/api/v1/session/create 
         with the JWT in Authorization header
STEP 7:  APIM receives the request:
         a) validate-jwt policy checks token signature, audience, expiry, tenant
         b) APIM extracts user_id (oid claim) from the JWT
         c) APIM injects headers: 
            X-User-Id: <entra_object_id>
            Ocp-Apim-Subscription-Key: <internal_key>
         d) APIM forwards to App Service backend pool
STEP 8:  FastAPI Gateway receives the request:
         a) Validates Ocp-Apim-Subscription-Key matches expected value
            (lightweight string check — confirms request came through APIM)
         b) Reads X-User-Id header (trusts APIM already validated the JWT)
STEP 9:  Gateway generates a new session_id (UUID v4)
STEP 10: Gateway generates a correlation_id (UUID v4) for distributed tracing
STEP 11: Gateway writes a new session_context document to Cosmos DB:
         {
           "id": "ctx_<session_id>",
           "session_id": "<session_id>",
           "user_id": "<entra_object_id>",
           "current_product": null,
           "session_entities": [],
           "status": "active",
           "created_at": "<timestamp>",
           "last_active_at": "<timestamp>",
           "expires_at": "<timestamp + 10 minutes>",
           "turn_count": 0,
           "pending_messages": 0,
           "ai_foundry_thread_id": null
         }
```

> **IMPORTANT: No AI Foundry Thread is created here.** The `ai_foundry_thread_id` is `null`. The thread will be created lazily when the first message that requires LLM reasoning arrives (see §4.4). This avoids creating thousands of empty threads for sessions where users never type anything or where all queries are resolved deterministically.

```
STEP 12: Gateway negotiates a Web PubSub client access URL for the session:
         → Calls Web PubSub SDK: generate_client_token(user_id, groups=[session_id])
         → Gets back a wss:// URL with embedded access token
STEP 13: Gateway returns to frontend:
         { 
           "session_id": "<id>", 
           "ws_url": "wss://pubsub.company.com/client/hubs/chat?access_token=<token>" 
         }
STEP 14: Frontend opens WebSocket connection to the ws_url
         → Connection is to Azure Web PubSub (NOT to the App Service)
         → Frontend joins the group for this session_id
         → If the App Service instance scales down, this connection SURVIVES
           because it's on the managed Web PubSub service
```

### 4.3 — User Sends Their First Chat Message

```
STEP 15: User types: "What is the spec for LS - 12345?"
STEP 16: Next.js sends POST https://apim.company.com/api/v1/chat/send
         Body: { "session_id": "<id>", "message": "What is the spec for LS - 12345?" }
         Header: Authorization: Bearer <jwt>
STEP 17: APIM validates JWT, injects X-User-Id + subscription key, forwards to App Service
STEP 18: Gateway validates subscription key + session (exists, not expired, belongs to user)
STEP 19: Gateway checks if session has pending_messages > 0
         → If YES: this message is QUEUED (see §4.9 for multi-message handling)
         → If NO: continue
STEP 20: Gateway increments pending_messages to 1 in session_context
STEP 21: Gateway generates a message_id (UUID v4) and correlation_id (UUID v4)
STEP 22: Gateway pushes status to frontend via Web PubSub SDK:
         → client.send_to_group(session_id, {
             "type": "status", "message_id": "<id>", "status": "queued"
           })
STEP 23: Gateway enqueues the message to Azure Service Bus queue "chat-queue":
         {
           "message_id": "<msg_id>",
           "session_id": "<session_id>",
           "user_id": "<entra_object_id>",
           "correlation_id": "<correlation_id>",
           "content": "What is the spec for LS - 12345?",
           "timestamp": "<now>"
         }
         → Uses Service Bus SESSION feature: session_id = "<session_id>"
           (ensures FIFO ordering per user session)
STEP 24: Gateway returns HTTP 202 Accepted:
         { "message_id": "<msg_id>", "status": "queued" }
```

### 4.4 — Worker Picks Up the Message: Deterministic Pre-Processing & Lazy Thread Creation

```
STEP 25: Worker (background asyncio task) dequeues the message from Service Bus
         → Worker holds a SESSION LOCK on this session_id
         → No other App Service instance can process messages for this session
           until this worker releases the lock
         → Worker sets the correlation_id from the message as the active 
           OpenTelemetry trace context
STEP 26: Worker pushes status via Web PubSub:
         → { "type": "status", "message_id": "<id>", "status": "processing",
             "display_text": "Analyzing your question..." }
STEP 27: Worker loads session_context from Cosmos DB to get:
         - current_product (null for first message)
         - session_entities (empty for first message)
         - ai_foundry_thread_id (null for first message)
```

**=== DETERMINISTIC PRE-PROCESSING (Before LLM) ===**

```
STEP 28: Worker runs regex scan for part number patterns:
         Pattern: /\bLS[\s\-_\.]*\d+/i
         Input: "What is the spec for LS - 12345?"
         Match: "LS - 12345"

         If MATCH found:
           → Normalize: uppercase prefix, replace separators with underscore
           → "LS - 12345" → "LS_12345"
           → Continue to STEP 29 (direct API call)
         
         If NO MATCH:
           → Check if clearly off-topic (pattern match against known categories)
           → If off-topic → short-circuit with polite redirect (no LLM needed)
           → Otherwise → skip to STEP 33 (route to AI Foundry Agent)
```

**For simple single-intent part number queries (the common case):**

```
STEP 29: Worker calls Product Description API DIRECTLY (no LLM involved):
         → GET https://api.company.com/v1/products/LS_12345
         → With timeout (10s), retry (3x, exponential backoff), circuit breaker
         → Pushes status via Web PubSub:
           { "type": "status", "display_text": "Searching for product LS-12345..." }

STEP 30: Handle API result:
         
         IF exact match (1 result):
           → Product data available
           → Worker formats response using brief LLM call for natural language,
             OR uses template-based formatting for simple specs
           → Skip to STEP 39 (Response Delivery)
         
         IF multiple matches:
           → Worker formats disambiguation question (template-based, no LLM needed):
             "I found multiple matches for LS_12345:
              1) LS_12345_V1 — Hydraulic Pump
              2) LS_12345_V2 — Hydraulic Motor
              Which one are you asking about?"
           → Skip to STEP 39 (Response Delivery)
         
         IF zero results:
           → Generate fallback candidate by stripping ALL separators: "LS12345"
           → Retry Product Description API with "LS12345"
           → If found → handle as exact match / multiple matches above
           → If still zero → continue to STEP 31 (fallback to AI Foundry)

STEP 31: Zero results from both candidates → route to AI Foundry for 
         Product GPT + AI Search grounding (see §4.6)
```

**For compound queries, ambiguous messages, or fallback scenarios — Lazy Thread Creation:**

```
STEP 32: Worker determines that this message needs LLM reasoning
         (compound query, ambiguous reference, fallback from zero API results,
          or message the pre-processor cannot confidently classify)

STEP 33: Worker checks ai_foundry_thread_id from session_context:

         IF ai_foundry_thread_id is null (first time needing LLM for this session):
           → Worker creates a new AI Foundry Thread via SDK:
             thread = client.agents.create_thread()
           → Worker persists thread_id back to Cosmos DB session_context:
             update session_context.ai_foundry_thread_id = thread.id
           → This is the ONLY time a thread is created
         
         IF ai_foundry_thread_id exists:
           → Worker retrieves the existing thread

STEP 34: Worker saves the user message to Cosmos DB conversations container:
         {
           "id": "<msg_id>",
           "session_id": "<session_id>",
           "user_id": "<entra_object_id>",
           "correlation_id": "<correlation_id>",
           "role": "user",
           "content": "What is the spec for LS - 12345?",
           "timestamp": "<now>",
           "metadata": {
             "routing_path": "ai_foundry"
           }
         }
```

> **How the Worker communicates with Azure AI Foundry (step by step):**
>
> The Worker uses the `azure-ai-projects` Python SDK. Here is the exact communication pattern:

```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.ai.projects.models import FunctionTool, ToolSet, AzureAISearchTool

# 1. Initialize the client (once at app startup)
client = AIProjectClient.from_connection_string(
    credential=DefaultAzureCredential(),
    conn_str="<your_ai_foundry_connection_string>"
)

# 2. Define tools — these are Python functions the agent can call
product_description_api = FunctionTool(functions=[lookup_product_description])
product_gpt_api = FunctionTool(functions=[query_product_gpt])

# Azure AI Search is added as a connected resource in AI Foundry
search_tool = AzureAISearchTool(
    index_connection_id="<ai_search_connection_id>",
    index_name="product-knowledge-index"
)

tool_set = ToolSet()
tool_set.add(product_description_api)
tool_set.add(product_gpt_api)
tool_set.add(search_tool)

# 3. Create the agent (once, reuse across sessions)
agent = client.agents.create_agent(
    model="gpt-4o",
    name="product-orchestrator",
    instructions=SYSTEM_PROMPT,  # See Section 10
    toolset=tool_set
)

# 4. Per-session: create a thread LAZILY (done on first LLM-bound message — Step 33)
thread = client.agents.create_thread()

# 5. Per-message: add the user message to the thread
client.agents.create_message(
    thread_id=thread.id,
    role="user",
    content="What is the spec for LS - 12345?"
)

# 6. Start a Run — the agent now processes the message
run = client.agents.create_and_process_run(
    thread_id=thread.id,
    agent_id=agent.id
)
# This call BLOCKS until the agent finishes (including all tool calls)
# OR returns with status "requires_action" if it needs tool results

# 7. Handle tool calls (if the agent decided to call a tool)
while run.status == "requires_action":
    tool_calls = run.required_action.submit_tool_outputs.tool_calls
    tool_outputs = []
    for tool_call in tool_calls:
        # Execute the function locally and collect results
        result = execute_tool(tool_call.function.name, tool_call.function.arguments)
        tool_outputs.append({"tool_call_id": tool_call.id, "output": str(result)})
    
    # Submit results back to the agent
    run = client.agents.submit_tool_outputs_and_process(
        thread_id=thread.id,
        run_id=run.id,
        tool_outputs=tool_outputs
    )

# 8. Get the final response
messages = client.agents.list_messages(thread_id=thread.id)
assistant_reply = messages.get_last_text_message_by_role("assistant")
```

> **Key concept:** The Azure AI Foundry Thread maintains ALL conversation history automatically. You add a message, start a Run, the agent reads the full thread, decides what tools to call, and produces a response. You don't manually manage context — the Thread IS the context.

### 4.5 — Agent Processes: Part Number Detection Path

> **Note:** For simple single-intent part number queries, the deterministic pre-processor (Steps 28-30) handles this path directly. The flow below applies when the LLM is involved (compound queries, ambiguous references, or pre-processor fallback).

```
STEP 35: Worker pushes status via Web PubSub:
         { "type": "status", "message_id": "<id>", "status": "processing",
           "display_text": "Searching for product LS-12345..." }
STEP 36: The agent (inside the Run) analyzes the user message and detects:
         - Intent: part_lookup
         - Raw part number: "LS - 12345"
         - Normalized: "LS_12345" (uppercase prefix, replace separators with _)
STEP 37: Agent decides to call the `product_description_api` tool
         The Run pauses with status = "requires_action"
         The tool_call contains:
         {
           "function": {
             "name": "product_description_api",
             "arguments": "{\"part_number\": \"LS_12345\"}"
           }
         }
STEP 38: Worker intercepts the tool call and executes it:
         → Makes HTTP GET to the existing Product Description API:
           GET https://api.company.com/v1/products/LS_12345
         → With timeout (10s), retry (3x), circuit breaker
         → Receives JSON response with product details
         → Pushes status via Web PubSub:
           { "type": "status", "display_text": "Found product details, preparing response..." }
STEP 38b: Worker submits the tool output back to the agent:
         tool_outputs = [{"tool_call_id": "<id>", "output": "<product_json>"}]
         run = client.agents.submit_tool_outputs_and_process(...)
STEP 38c: Agent receives the product data, formats a natural language response,
         and completes the Run with status = "completed"
```

**What happens at each result scenario:**

| Scenario | Agent Behavior |
|----------|---------------|
| **Exact match (1 result)** | Agent formats specs into a readable response. Worker updates `session_context.current_product` with this product. |
| **Multiple matches** | Agent returns a disambiguation question: *"I found multiple matches: 1) LS_12345_V1 — Pump, 2) LS_12345_V2 — Motor. Which one?"* Worker marks this turn as `awaiting_disambiguation` in session context. |
| **Zero results (primary)** | Agent tries fallback candidate "LS12345" (strip all separators). Calls `product_description_api` again. |
| **Zero results (fallback also fails)** | Agent hands off to Product GPT + Knowledge Base grounding (see §4.6). |

### 4.6 — Fallback Path: Product GPT + Azure AI Search Grounding

When the Product Description API returns zero results for all normalized candidates, the agent does not give up. Instead, it falls back to a combined approach using the Product GPT API and the Azure AI Search knowledge base (web crawler data).

```
STEP 35f: Agent detects zero results from product_description_api (both primary 
         and fallback candidates failed)
STEP 36f: Agent calls TWO tools in sequence:
         
         TOOL 1: search_knowledge_base (Azure AI Search)
         → The agent formulates a semantic search query from the user's question
         → Azure AI Search returns relevant document chunks from the 
           "product-knowledge-index" (web-crawled company website content)
         → These chunks provide GROUNDING CONTEXT
         
         Worker pushes status via Web PubSub:
         { "type": "status", "display_text": "Searching knowledge base..." }
         
         TOOL 2: product_gpt_api (existing external AI app)
         → Agent passes the original user question to the Product GPT API
         → Product GPT returns its own AI-generated answer
         → NOTE: Product GPT may return LARGE, verbose data
STEP 37f: Agent receives BOTH outputs and performs GROUNDED SYNTHESIS:
         
         The agent's system prompt instructs it to:
         1. Use the Product GPT answer as the PRIMARY source (higher trust — 
            curated, tested product data)
         2. Use Azure AI Search chunks as SUPPORTING EVIDENCE to validate 
            and enrich the Product GPT answer
         3. If Product GPT and AI Search AGREE → high confidence response
         4. If Product GPT and AI Search CONFLICT → present the Product GPT 
            answer but note: "Based on additional sources, there may be 
            updates to this information. Please verify with your sales team."
         5. If ONLY AI Search has data (Product GPT had no answer) → present 
            the AI Search data with a confidence qualifier: "Based on 
            information from our website..."
         6. NEVER fabricate — if neither source has data, say so clearly
```

> **Why this grounding strategy?**
> *   The Product GPT API is an existing, tested product → higher data quality → PRIMARY source
> *   The Azure AI Search web crawler data is uncontrolled (you don't manage the crawl pipeline) → lower confidence → SUPPORTING source
> *   By using AI Search as validation rather than primary answer, you avoid surfacing potentially stale or incorrect crawled data while still benefiting from the broader coverage it provides
> *   The agent's system prompt enforces this priority ranking (see §10)

### 4.7 — Editor Model: Summarizing Verbose Responses for Design Engineers

After the orchestrator agent produces its response, it may be too verbose for the target audience (product design engineers who need specific technical details, not marketing content or exhaustive catalogs).

```
STEP 38e: Orchestrator agent completes its response
STEP 39e: The ORCHESTRATOR AGENT itself decides whether editing is needed.
         This decision is embedded in the system prompt (see §10):
         
         CRITERIA FOR TRIGGERING THE EDITOR:
         a) The response contains data from Product GPT (which tends to be verbose)
         b) The user's question was specific (e.g., "what's the torque rating?") 
            but the response is broad
         c) The response exceeds ~500 tokens
         
         If the orchestrator determines editing is needed, it sets a flag 
         in its response metadata: { "needs_editing": true }
STEP 40e: Worker checks the orchestrator's response metadata:
         
         IF needs_editing == true:
           → Worker pushes status:
             { "type": "status", "display_text": "Preparing your answer..." }
           → Worker sends the raw response to the EDITOR AGENT (GPT-4o-mini):
             
             Editor Agent System Prompt:
             "You are a technical editor for product design engineers.
              Given a raw AI response about industrial products, produce
              a concise, technically focused summary. Keep:
              - Exact specifications (dimensions, weights, ratings, materials)
              - Compatibility information
              - Part numbers and model variants
              Remove:
              - Marketing language
              - General company information
              - Redundant descriptions
              - Information not relevant to the user's specific question
              Format: Use tables for specifications when there are 3+ data points."
             
             → Editor processes in ~500ms (GPT-4o-mini is fast)
             → Returns the filtered, concise response
           → Worker uses the EDITED response as the final response
         
         IF needs_editing == false:
           → Worker uses the orchestrator's response directly (no extra latency)
```

> **Why let the orchestrator decide?** Rather than always running the editor (adding 500ms to every response) or using a rigid token-count threshold (which would miss short-but-verbose answers), the orchestrator agent is best positioned to judge: it knows the user's original question, it knows whether the data came from Product GPT (verbose) or Product Description API (structured), and it knows if the answer is already concise enough. This keeps the common case fast (direct Product Description API answers are already structured and pass through without editing) while filtering the verbose cases.

### 4.8 — Response Delivery & State Persistence

```
STEP 39: Worker has the final response (either direct from pre-processor, 
         agent response, or editor-processed)
STEP 40: Worker saves the assistant message to Cosmos DB conversations container:
         {
           "id": "msg_<uuid>",
           "session_id": "<session_id>",
           "correlation_id": "<correlation_id>",
           "role": "assistant",
           "content": "<final_response>",
           "timestamp": "<now>",
           "metadata": {
             "detected_intents": ["part_lookup"],
             "tools_called": ["product_description_api"],
             "normalized_part_numbers": ["LS_12345"],
             "resolved_product": "LS_12345_V2",
             "grounding_sources": [],
             "was_edited": false,
             "routing_path": "deterministic",
             "processing_time_ms": 2340,
             "prompt_tokens": 0,
             "completion_tokens": 0
           }
         }
STEP 41: Worker updates session_context in Cosmos DB:
         - current_product → { part_number, product_name, resolved_at }
         - session_entities → append new part numbers
         - last_active_at → now
         - expires_at → now + 10 minutes (sliding expiration)
         - turn_count → increment
         - pending_messages → decrement
STEP 42: Worker pushes the FINAL response via Web PubSub:
         → client.send_to_group(session_id, {
             "type": "response",
             "message_id": "<msg_id>",
             "content": "<formatted_response>",
             "metadata": {
               "resolved_product": "LS_12345_V2",
               "sources": ["Product Description API"]
             }
           })
STEP 43: Worker checks if there are more queued messages (pending_messages > 0)
         → If YES: dequeue next message from Service Bus and repeat from Step 25
         → If NO: release the Service Bus session lock, wait for next message
```

### 4.9 — Multi-Message Handling (2, 3, or N Messages Before Previous Finishes)

When a user sends multiple messages while the system is still processing:

```
TIMELINE:
  T=0s   MESSAGE 1: "What is the spec for LS-12345?"    → Sent, processing starts
  T=2s   MESSAGE 2: "Also, what pumps do you have?"     → Sent while MSG 1 processing
  T=3s   MESSAGE 3: "And check LS-99001 too"            → Sent while MSG 1 still processing
  T=8s   MESSAGE 1 completes                             → Response delivered
  T=8.1s MESSAGE 2 processing starts                     → Auto-dequeued
  T=14s  MESSAGE 2 completes                             → Response delivered
  T=14.1s MESSAGE 3 processing starts                    → Auto-dequeued
  T=20s  MESSAGE 3 completes                             → Response delivered

HOW IT WORKS:
STEP A: Gateway receives MESSAGE 2 via POST /api/v1/chat/send
STEP B: Gateway checks session_context.pending_messages → value is 1 (MSG 1 in progress)
STEP C: Gateway increments pending_messages to 2
STEP D: Gateway enqueues MESSAGE 2 to Service Bus queue "chat-queue"
        (same Service Bus session as MSG 1 → FIFO ordering guaranteed)
STEP E: Gateway pushes status via Web PubSub:
        { "type": "status", "message_id": "<msg2_id>", "status": "queued",
          "display_text": "Your question is queued (position 2). I'll answer it shortly.",
          "queue_position": 2 }
STEP F: Gateway returns HTTP 202:
        { "message_id": "<msg2_id>", "status": "queued", "queue_position": 2 }
STEP G: When Worker finishes MESSAGE 1:
        → Decrements pending_messages to 1
        → Sends response for MSG 1 via Web PubSub
        → Immediately dequeues MESSAGE 2 from Service Bus (same session lock)
        → Pushes status update for MSG 2:
          { "type": "status", "message_id": "<msg2_id>", "status": "processing",
            "display_text": "Now answering your second question..." }
        → Also pushes queue update for MSG 3:
          { "type": "status", "message_id": "<msg3_id>", "status": "queued",
            "queue_position": 2 }  ← position updated from 3 to 2
```

**Frontend UI behavior for queued messages:**

```
┌─────────────────────────────────────────────┐
│ You: What is the spec for LS-12345?         │
│ 🔄 Searching for product LS-12345...       │  ← active processing indicator
│                                             │
│ You: Also, what pumps do you have?          │
│ ⏳ Queued (position 2)                      │  ← queued indicator
│                                             │
│ You: And check LS-99001 too                 │
│ ⏳ Queued (position 3)                      │  ← queued indicator
└─────────────────────────────────────────────┘

After MSG 1 completes:
┌─────────────────────────────────────────────┐
│ You: What is the spec for LS-12345?         │
│ Bot: Here are the specs for LS_12345_V2...  │  ← completed ✓
│                                             │
│ You: Also, what pumps do you have?          │
│ 🔄 Searching product catalog...             │  ← now processing
│                                             │
│ You: And check LS-99001 too                 │
│ ⏳ Queued (position 2)                      │  ← position updated
└─────────────────────────────────────────────┘
```

> **FIFO guarantee:** Azure Service Bus sessions feature is used. All messages for a `session_id` go to the same Service Bus session, ensuring strict ordering. The Worker locks the Service Bus session during processing, so no other App Service instance can pick up messages from the same user session concurrently.

> **Context continuity:** Because all messages go to the SAME AI Foundry Thread, MESSAGE 3 automatically has the full context of MESSAGE 1 and MESSAGE 2's responses. If MSG 1 set `current_product = LS_12345_V2`, MSG 3 knows about it even though MSG 3 was queued before MSG 1 finished.

### 4.10 — Follow-Up Questions (Same Topic)

```
TURN 1: User: "What is the spec for LS-12345?"
        → Agent resolves LS_12345, sets current_product = LS_12345_V2
TURN 2: User: "What's the weight?"
        → No part number in the query
        → Worker loads session_context → current_product = LS_12345_V2
        → Worker adds session context as a system-level note to the Thread:
          "[System Context: Current product is LS_12345_V2 (Hydraulic Pump Assembly)]"
        → Agent sees this context in the Thread, understands "weight" refers to LS_12345_V2
        → Agent calls product_description_api with LS_12345_V2 for weight data
        → OR answers from previously cached data already in the Thread
```

### 4.11 — Topic Switch (Different Product or Different Topic)

```
TURN 1: User: "What is the spec for LS-12345?"
        → current_product = LS_12345_V2
TURN 2: User: "Tell me about LS-99001"
        → Agent detects a NEW part number LS_99001
        → Agent normalizes → LS_99001
        → Agent calls product_description_api with LS_99001
        → Worker updates session_context:
          - current_product → LS_99001 (replaces LS_12345_V2)
          - session_entities → ["LS_12345_V2", "LS_99001"] (append, don't replace)
        → User can still reference LS_12345_V2 by name: "Go back to the pump we discussed"
TURN 3: User: "What general filtration products do you have?"
        → No part number detected
        → This is a GENERAL query, not a follow-up about current_product
        → Agent recognizes the topic shift and routes to product_gpt_api
        → current_product remains LS_99001 (general queries don't clear it)
```

> **How does the agent know it's a topic switch vs. a follow-up?** The system prompt instructs the agent to:
> *   If the query contains a NEW part number → it's a topic switch → update current_product
> *   If the query is vague and no part number → check if it logically follows from current_product
> *   If the query is clearly about a different product CATEGORY (e.g., "pumps" when current is "motors") → route to Product GPT, keep current_product unchanged
> *   The agent uses the full Thread context (all prior messages) to make this judgment

### 4.12 — Session Expiration (10-Minute Inactivity Timeout)

```
STEP: A background task in the FastAPI app runs every 60 seconds:
      → Queries Cosmos DB for sessions where expires_at < now AND status = "active"
      → For each expired session:
        1. Updates session_context.status = "expired"
        2. Pushes notification via Web PubSub:
           → client.send_to_group(session_id, {
               "type": "session_expired", 
               "message": "Your session has expired due to inactivity. 
                           Please start a new conversation."
             })
        3. Removes the group from Web PubSub (optional cleanup)
        4. Deletes the AI Foundry Thread if one was created (optional — or let it auto-expire)

FRONTEND BEHAVIOR:
      → On receiving "session_expired" event via WebSocket, the UI:
        1. Disables the chat input
        2. Shows a "Session Expired" banner
        3. Offers a "Start New Conversation" button
        4. On click → calls POST /api/v1/session/create → new session cycle
```

> **The 10-minute timer is a sliding window.** Every time the user sends a message, `expires_at` is reset to `now + 10 minutes`. The session only expires if there is no activity for a full 10 minutes.

---

## 5. Auto-Scaling Strategy (Scale-Out & Scale-Down)

### 5.1 — Scale-Out: Add Instances When Queue Grows

```
SCALING RULE (Azure App Service Custom Metric Auto-Scale):
  Metric Source:  Azure Service Bus queue "chat-queue"
  Metric:        ActiveMessageCount (messages waiting in queue)
  
  SCALE OUT:
    When ActiveMessageCount > 10 for 1 minute → add 1 instance
    When ActiveMessageCount > 20 for 1 minute → add 2 instances
    Maximum instances: 5
    Cool-down period: 5 minutes (prevent flapping)
  
  SCALE IN:
    When ActiveMessageCount < 2 for 5 minutes → remove 1 instance
    Minimum instances: 1
    Cool-down period: 10 minutes (conservative scale-in)

WHAT HAPPENS WHEN A NEW INSTANCE STARTS:
  1. App Service starts a new container with the same FastAPI image
  2. On startup (lifespan event), the app:
     a) Initializes the Azure AI Foundry client (reuses the SAME agent)
     b) Initializes the Service Bus client
     c) Starts the background Worker listener
     d) Initializes the Web PubSub client
     e) Initializes OpenTelemetry instrumentation
  3. The new Worker starts competing for Service Bus messages
     → BUT: Service Bus sessions ensure it ONLY picks up messages from
       sessions NOT currently locked by another instance
  4. New user sessions may now be routed to this new instance
  5. Existing sessions continue on their current instance (session lock)
```

### 5.2 — Scale-Down: Remove Instances Without Losing Sessions

> **The core risk:** If Azure removes an App Service instance that is actively processing a message, we could lose that response. Here is how we prevent that:

```
GRACEFUL SHUTDOWN SEQUENCE:
  1. Azure sends SIGTERM to the instance marked for removal
     → Azure waits up to 230 seconds (configurable) before force-killing
  2. FastAPI receives SIGTERM via the lifespan shutdown handler:
     
     @asynccontextmanager
     async def lifespan(app: FastAPI):
         # STARTUP
         worker_task = asyncio.create_task(worker_loop())
         yield
         # SHUTDOWN (triggered by SIGTERM)
         worker_task.cancel()  # signals worker to stop accepting NEW messages
         await worker_task     # waits for current message to finish
     
  3. Worker shutdown behavior:
     a) STOPS accepting new messages from Service Bus
        → Sets a flag: accepting_new_messages = False
     b) CONTINUES processing the current message (if any)
        → This is the critical part — the in-flight message completes
     c) After current message completes:
        → Releases the Service Bus session lock
        → Any remaining queued messages for that session are available
          for OTHER instances to pick up
     d) Worker exits cleanly
  
  4. Azure removes the instance
```

**4-Layer Protection Model — Why No Data Is Lost:**

| Layer | Protection | Explanation |
|-------|-----------|-------------|
| **1. Service Bus Message Locks** | If the worker crashes (worst case), the lock expires after 5 minutes (configurable) and the message goes BACK to the queue for another worker to pick up. The message is NOT lost. |
| **2. Graceful Shutdown** | The worker finishes its current message before exiting. No message is abandoned mid-processing. |
| **3. WebSocket Connections Survive** | Because WebSocket connections are on Azure Web PubSub (NOT on the App Service instance), the user's WebSocket connection SURVIVES when the instance is removed. Zero user disruption. |
| **4. Session State Is in Cosmos DB** | No session state lives in the App Service instance's memory. Everything is in Cosmos DB (`session_context`, `conversations`) and AI Foundry (Thread). Any instance can pick up any session. |

### 5.3 — Scaling Configuration (Bicep)

```json
{
  "name": "chat-queue-autoscale",
  "properties": {
    "profiles": [{
      "capacity": {
        "minimum": "1",
        "maximum": "5",
        "default": "1"
      },
      "rules": [
        {
          "metricTrigger": {
            "metricName": "ActiveMessageCount",
            "metricResourceUri": "<service_bus_queue_resource_id>",
            "operator": "GreaterThan",
            "threshold": 10,
            "timeAggregation": "Average",
            "timeWindow": "PT1M"
          },
          "scaleAction": {
            "direction": "Increase",
            "type": "ChangeCount",
            "value": "1",
            "cooldown": "PT5M"
          }
        },
        {
          "metricTrigger": {
            "metricName": "ActiveMessageCount",
            "metricResourceUri": "<service_bus_queue_resource_id>",
            "operator": "LessThan",
            "threshold": 2,
            "timeAggregation": "Average",
            "timeWindow": "PT5M"
          },
          "scaleAction": {
            "direction": "Decrease",
            "type": "ChangeCount",
            "value": "1",
            "cooldown": "PT10M"
          }
        }
      ]
    }]
  }
}
```

---

## 6. Agent Context Handoff: How Context Flows Between Turns

### 6.1 — Thread-Based Context (Automatic)

Azure AI Foundry Threads provide automatic context continuity:

```
Thread (thread_id: "thread_abc123")
├── Message 1: [user] "What is the spec for LS-12345?"
├── Message 2: [assistant] "Here are the specs for LS_12345_V2..."
│   └── (internally, the agent also remembers: it called product_description_api,
│        got result X, normalized the part number, etc.)
├── Message 3: [user] "What's the weight?"
├── Message 4: [assistant] "The weight of LS_12345_V2 is 12.5 kg..."
└── ...
```

When a new Run starts on the same Thread, the agent sees ALL prior messages. This is the primary context mechanism — you don't need to manually pass conversation history.

### 6.2 — Session Context Injection (Explicit, for Robustness)

In addition to the Thread's built-in memory, the Worker explicitly injects session context at the start of each Run to handle edge cases (e.g., context window limits, token optimization):

```python
# Before starting each Run, the Worker injects a context summary
context = load_session_context(session_id)  # from Cosmos DB
context_message = f"""[SYSTEM CONTEXT - DO NOT REPEAT TO USER]
Current product: {context['current_product']['part_number']} ({context['current_product']['product_name']})
Session entities discussed: {', '.join(context['session_entities'])}
Turn count: {context['turn_count']}
"""
# Add as a message to the Thread
client.agents.create_message(
    thread_id=thread_id,
    role="user",  # injected as user message with system prefix
    content=context_message
)
# Then add the actual user message
client.agents.create_message(
    thread_id=thread_id,
    role="user",
    content=user_message
)
# Start the Run
run = client.agents.create_and_process_run(
    thread_id=thread_id,
    agent_id=agent.id
)
```

### 6.3 — Context Optimization (Token Management)

As conversations grow long, the Thread accumulates many messages. To keep token costs under control:

```python
# Every 10 turns, summarize the conversation and create a new Thread
if context['turn_count'] % 10 == 0 and context['turn_count'] > 0:
    # Get all messages from current thread
    messages = client.agents.list_messages(thread_id=current_thread_id)
    
    # Create a summary using the agent itself
    summary_prompt = "Summarize the key facts from this conversation: products discussed, resolved questions, and any pending follow-ups."
    
    # Create new thread with the summary as first message
    new_thread = client.agents.create_thread()
    client.agents.create_message(
        thread_id=new_thread.id,
        role="user",
        content=f"[CONVERSATION SUMMARY]\n{summary}"
    )
    
    # Update session_context with new thread_id
    update_session_context(session_id, {"ai_foundry_thread_id": new_thread.id})
```

---

## 7. Azure AI Search: Web Crawler & Knowledge Base Setup

### 7.1 — Architecture

```text
Company Website (https://www.company.com)
         │
         ▼ (scheduled crawl)
┌────────────────────────────────────────┐
│ Azure AI Search Indexer                 │
│  Data Source: Web Crawler (built-in)    │
│  Target: https://www.company.com/*     │
│  Schedule: Every 24 hours              │
│  Skills: Split text → Generate vectors │
└────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ Azure AI Search Index                   │
│  Name: "product-knowledge-index"        │
│  Fields:                                │
│    - id (key)                           │
│    - content (text, searchable)         │
│    - content_vector (vector, 3072-dim)  │
│    - url (text, filterable)             │
│    - title (text, searchable)           │
│    - chunk_id (text)                    │
│    - last_crawled_at (datetime)         │
│  Vector Config:                         │
│    - Algorithm: HNSW                    │
│    - Model: text-embedding-3-large      │
│    - Dimensions: 3072                   │
└────────────────────────────────────────┘
```

### 7.2 — Indexer Configuration

```json
{
  "name": "company-web-crawler-indexer",
  "dataSourceName": "company-website-datasource",
  "targetIndexName": "product-knowledge-index",
  "schedule": {
    "interval": "PT24H"
  },
  "parameters": {
    "configuration": {
      "dataToExtract": "contentAndMetadata",
      "parsingMode": "default"
    }
  },
  "skillsetName": "company-web-skillset"
}
```

**Skillset (chunking + embedding):**

```json
{
  "name": "company-web-skillset",
  "skills": [
    {
      "@odata.type": "#Microsoft.Skills.Text.SplitSkill",
      "name": "chunk-text",
      "textSplitMode": "pages",
      "maximumPageLength": 2000,
      "pageOverlapLength": 200
    },
    {
      "@odata.type": "#Microsoft.Skills.Text.AzureOpenAIEmbeddingSkill",
      "name": "generate-embeddings",
      "resourceUri": "<azure_openai_endpoint>",
      "deploymentId": "text-embedding-3-large",
      "modelName": "text-embedding-3-large"
    }
  ]
}
```

### 7.3 — Integration with Azure AI Foundry Agent

The Azure AI Search index is registered as a connected resource in the Azure AI Foundry project. The agent accesses it as a built-in `AzureAISearchTool`:

```python
from azure.ai.projects.models import AzureAISearchTool

search_tool = AzureAISearchTool(
    index_connection_id="<connection_id_from_ai_foundry_project>",
    index_name="product-knowledge-index"
)
```

When the agent calls this tool, Azure AI Foundry automatically:
1.  Takes the agent's search query
2.  Generates an embedding vector for the query
3.  Performs hybrid search (vector + keyword) against the index
4.  Returns the top-K relevant document chunks
5.  The agent uses these chunks as grounding context for its response

---

## 8. API Specification

### 8.1 — REST Endpoints (FastAPI Gateway)

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `POST` | `/api/v1/session/create` | Create new chat session. Returns `session_id` and `ws_url`. | APIM JWT → subscription key |
| `POST` | `/api/v1/chat/send` | Send a chat message. Enqueues to Service Bus. Returns `message_id` + `queue_position`. | APIM JWT → subscription key |
| `GET` | `/api/v1/session/{session_id}/history` | Retrieve conversation history from Cosmos DB. | APIM JWT → subscription key |
| `DELETE` | `/api/v1/session/{session_id}` | End a session explicitly. | APIM JWT → subscription key |
| `GET` | `/api/v1/health` | Health check (no auth). | None |

### 8.2 — WebSocket Protocol (via Azure Web PubSub)

**Connection:** `wss://<pubsub_host>/client/hubs/chat?access_token=<web_pubsub_token>`

The token is returned by `POST /api/v1/session/create`. The frontend connects directly to Azure Web PubSub, NOT to the App Service.

**Server → Client message types (pushed via Web PubSub SDK):**

```typescript
// Status update (sent during processing)
{
  "type": "status",
  "message_id": "msg_uuid",
  "status": "queued" | "processing" | "completed" | "error",
  "display_text": "Searching for product LS-12345...",  // human-readable
  "queue_position": 2,  // only present when status = "queued"
  "timestamp": "2026-07-27T01:45:00Z"
}

// Final response
{
  "type": "response",
  "message_id": "msg_uuid",
  "content": "Here are the specifications for LS_12345_V2...",
  "metadata": {
    "resolved_product": "LS_12345_V2",
    "sources": ["Product Description API"],
    "tools_called": ["product_description_api"],
    "was_edited": false
  },
  "timestamp": "2026-07-27T01:45:05Z"
}

// Disambiguation request
{
  "type": "disambiguation",
  "message_id": "msg_uuid",
  "content": "I found multiple matches. Did you mean:",
  "options": [
    { "part_number": "LS_12345_V1", "name": "Hydraulic Pump" },
    { "part_number": "LS_12345_V2", "name": "Hydraulic Motor" }
  ],
  "timestamp": "2026-07-27T01:45:05Z"
}

// Session expiration
{
  "type": "session_expired",
  "message": "Your session has expired due to inactivity.",
  "timestamp": "2026-07-27T01:55:00Z"
}

// Error
{
  "type": "error",
  "message_id": "msg_uuid",
  "content": "Unable to retrieve product details. Please try again.",
  "error_code": "TOOL_TIMEOUT",
  "timestamp": "2026-07-27T01:45:05Z"
}
```

**Client → Server:** Not used. The frontend sends messages via REST (`POST /api/v1/chat/send`), not via WebSocket. The WebSocket channel is strictly server→client for real-time push.

### 8.3 — External API Tool Definitions

**Product Description API (Existing REST API — used as Function Tool):**

```python
def lookup_product_description(part_number: str) -> dict:
    """
    Look up product details by part number from the Product Description API.
    
    Args:
        part_number: The normalized part number (e.g., "LS_12345")
    
    Returns:
        dict with product details or empty dict if not found
    """
    response = httpx.get(
        f"https://api.company.com/v1/products/{part_number}",
        headers={"Authorization": f"Bearer {get_api_token()}"},
        timeout=10.0
    )
    if response.status_code == 200:
        return response.json()
    elif response.status_code == 404:
        return {"results": [], "count": 0}
    else:
        raise ToolExecutionError(f"Product API returned {response.status_code}")
```

**Product GPT API (Existing External AI App — used as Function Tool):**

```python
def query_product_gpt(query: str) -> dict:
    """
    Query the Product GPT API for general product information.
    This is an existing external AI application with its own model.
    
    Args:
        query: Natural language product question
    
    Returns:
        dict with AI-generated answer about company products
    """
    response = httpx.post(
        "https://product-gpt.company.com/api/v1/ask",
        json={"query": query},
        headers={"Authorization": f"Bearer {get_api_token()}"},
        timeout=30.0
    )
    if response.status_code == 200:
        return response.json()
    else:
        raise ToolExecutionError(f"Product GPT API returned {response.status_code}")
```

---

## 9. Cosmos DB Data Model

All conversation state is persisted in **Azure Cosmos DB (Serverless)**.

### Container: `conversations`
**Partition Key:** `/session_id`

```json
{
  "id": "msg_uuid_001",
  "session_id": "sess_abc123",
  "user_id": "entra_object_id_xyz",
  "correlation_id": "corr_gateway_001",
  "timestamp": "2026-07-27T01:45:00Z",
  "role": "user | assistant",
  "content": "What is the spec for LS-12345?",
  "metadata": {
    "detected_intents": ["part_lookup"],
    "tools_called": ["product_description_api"],
    "normalized_part_numbers": ["LS_12345"],
    "resolved_product": "LS_12345_V2",
    "grounding_sources": ["azure_ai_search"],
    "was_edited": false,
    "routing_path": "deterministic | ai_foundry",
    "processing_time_ms": 2340,
    "prompt_tokens": 150,
    "completion_tokens": 200
  }
}
```

### Container: `session_context`
**Partition Key:** `/session_id`

```json
{
  "id": "ctx_sess_abc123",
  "session_id": "sess_abc123",
  "user_id": "entra_object_id_xyz",
  "status": "active | expired",
  "current_product": {
    "part_number": "LS_12345_V2",
    "product_name": "Hydraulic Pump Assembly",
    "resolved_at": "2026-07-27T01:45:30Z"
  },
  "session_entities": ["LS_12345_V2", "LS_99001"],
  "ai_foundry_thread_id": null,
  "created_at": "2026-07-27T01:40:00Z",
  "last_active_at": "2026-07-27T01:45:30Z",
  "expires_at": "2026-07-27T01:55:30Z",
  "turn_count": 5,
  "pending_messages": 0
}
```

> **Note on `ai_foundry_thread_id`:** This field is `null` when a session is first created. The AI Foundry thread is created lazily on the first message that requires LLM reasoning (see §4.4, Step 33). Sessions where users never type anything — or where all queries are resolved by the deterministic pre-processor — never create a thread.

**TTL Policy:** Set Cosmos DB TTL on `session_context` to 24 hours (86,400 seconds) so expired sessions auto-delete. Conversation history in `conversations` is retained indefinitely for audit/analytics.

### Why Cosmos DB Serverless
*   **Document model fits naturally:** Each chat message = one document. Sessions and context are naturally represented as JSON documents.
*   **No capacity planning:** Serverless auto-scales RU/s based on demand.
*   **Cost-effective at low-medium traffic:** Pay only for consumed RUs (~$0.25 per 1M RUs).
*   **Queryable history:** SQL-like queries enable analytics, audit trails, and debugging.
*   **No additional operational complexity:** Managed Identity access, automatic backups, no eviction policies.

---

## 10. System Prompt Configuration (Orchestrator Agent)

```text
You are an enterprise AI assistant for product and part inquiries at [Company Name].
Your users are Product Design Engineers who need precise technical information.
Your job is to help them find product information using the tools available to you.

═══════════════════════════════════════════
IMPORTANT CONTEXT — PRE-PROCESSING
═══════════════════════════════════════════
Simple, single-intent part number lookups are pre-processed by the system before 
reaching you. If you receive pre-processed product data in your context, use it 
directly — do NOT re-query the product_description_api for the same part number.
You may still receive part number queries when they are part of compound questions 
or require conversational reasoning.

═══════════════════════════════════════════
TOOL ROUTING RULES (in priority order)
═══════════════════════════════════════════
RULE 1 — PART NUMBER DETECTION:
If the user query contains a string matching a part number pattern (prefix starting 
with "LS" followed by digits, with any combination of spaces, hyphens, dots, or 
underscores as separators), and it was NOT already pre-processed, you MUST:
  a) Normalize: uppercase prefix, replace all separators with underscores
     Examples: "LS - 123" → "LS_123", "ls-456" → "LS_456", "Ls12345" → "LS_12345"
  b) Call the `product_description_api` tool with the normalized part number

RULE 2 — CANDIDATE FALLBACK:
If `product_description_api` returns zero results:
  a) Strip ALL separators from the normalized form (e.g., "LS_123" → "LS123")
  b) Retry `product_description_api` with this fallback form
  c) If STILL zero results → proceed to RULE 4 (Product GPT + Knowledge Base)

RULE 3 — DISAMBIGUATION:
If `product_description_api` returns MULTIPLE matches:
  • DO NOT guess or pick one
  • Present ALL candidates with part numbers and descriptions
  • Ask the user to clarify which one they meant
  • Example: "I found multiple matches for LS_12345:
    1) LS_12345_V1 — Hydraulic Pump
    2) LS_12345_V2 — Hydraulic Motor
    Which one are you asking about?"

RULE 4 — GENERAL PRODUCT QUERIES & FALLBACK:
If the user asks about products WITHOUT a specific part number, OR if a part number 
search returned zero results after all fallback attempts:
  a) Call `search_knowledge_base` to find relevant information from our website
  b) Call `product_gpt_api` with the user's question
  c) SYNTHESIZE both results following the GROUNDING RULES below

RULE 5 — FOLLOW-UP QUESTIONS:
If the user asks a question without a part number, AND the conversation context 
contains a previously discussed product (the [SYSTEM CONTEXT] message), assume 
the user is asking about that product UNLESS they explicitly indicate otherwise.
  • Re-query `product_description_api` if needed for additional attributes
  • Or answer from data already available in the conversation

RULE 6 — TOPIC SWITCH:
If the user introduces a NEW part number or explicitly changes topic:
  • Process the new query independently
  • The system will update the current product tracking

═══════════════════════════════════════════
GROUNDING RULES (for Product GPT + AI Search)
═══════════════════════════════════════════
When combining Product GPT and AI Search results:
1. Product GPT is the PRIMARY source (higher trust — curated product data)
2. AI Search results are SUPPORTING evidence (web-crawled, may be stale)
3. If both sources AGREE → respond with high confidence
4. If sources CONFLICT → present the Product GPT answer, add caveat:
   "Note: There may be updates to this information. Please verify with your 
   sales representative."
5. If ONLY AI Search has data → present with qualifier:
   "Based on information from our website: ..."
6. CITE which sources you used at the end of your response

═══════════════════════════════════════════
RESPONSE EDITING DECISION
═══════════════════════════════════════════
After composing your response, evaluate whether it needs editing for brevity.
Add the following JSON block AT THE END of your response (it will be parsed 
and stripped by the system before showing to the user):

[RESPONSE_META]{"needs_editing": true/false}[/RESPONSE_META]

Set needs_editing = true when ALL of these are true:
  • The response includes data from product_gpt_api (which tends to be verbose)
  • The user's question was specific (not a broad "tell me about...")
  • The response exceeds approximately 500 tokens
  • The response contains information not directly relevant to the user's question

Set needs_editing = false when ANY of these are true:
  • The response is from product_description_api only (already structured)
  • The response is a disambiguation question
  • The response is an error message
  • The user asked a broad question ("what products do you have?")

═══════════════════════════════════════════
COMPOUND QUERY HANDLING
═══════════════════════════════════════════
If a user message contains MULTIPLE questions or intents:
  • Identify each distinct sub-query
  • Process each using the appropriate rule above
  • Combine all results into a single, coherent response
  • Use clear section headers if answering multiple questions

═══════════════════════════════════════════
HARD CONSTRAINTS
═══════════════════════════════════════════
• NEVER fabricate or hallucinate product specifications
• ONLY report data returned by the tools
• If any tool fails, inform the user helpfully and suggest retrying
• You ONLY answer questions about company products and parts
• For unrelated questions, respond: "I'm designed to help with product and 
  part inquiries. How can I assist you with our products?"
• Do NOT repeat [SYSTEM CONTEXT] messages to the user
• Keep responses concise and well-formatted (use tables for specs when 
  appropriate)
• Your audience is Product Design Engineers — prioritize technical accuracy 
  and specifications over marketing language
```

---

## 11. Project Structure (Code Layout)

```
enterprise-ai-assistant/
├── backend/                          # FastAPI application
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py                   # FastAPI app entry, lifespan events, 
│   │   │                             # graceful shutdown handler
│   │   ├── config.py                 # Environment config (pydantic-settings)
│   │   ├── dependencies.py           # Dependency injection (DB, AI client, etc.)
│   │   │
│   │   ├── api/                      # REST API layer
│   │   │   ├── __init__.py
│   │   │   ├── router.py             # Main API router
│   │   │   ├── session.py            # POST /session/create, DELETE /session/{id}
│   │   │   ├── chat.py               # POST /chat/send, GET /session/{id}/history
│   │   │   └── health.py             # GET /health
│   │   │
│   │   ├── auth/                     # Authentication
│   │   │   ├── __init__.py
│   │   │   └── apim_validator.py     # APIM subscription key validation + 
│   │   │                             # X-User-Id extraction (lightweight)
│   │   │
│   │   ├── pubsub/                   # Azure Web PubSub integration
│   │   │   ├── __init__.py
│   │   │   ├── client.py             # Web PubSub service client setup
│   │   │   └── publisher.py          # Helper: send_status, send_response, 
│   │   │                             # send_error, send_session_expired
│   │   │
│   │   ├── worker/                   # Background worker (Service Bus consumer)
│   │   │   ├── __init__.py
│   │   │   ├── consumer.py           # Service Bus session-aware message listener
│   │   │   ├── preprocessor.py       # Deterministic pre-processing: regex scan,
│   │   │   │                         # part number normalization, direct API calls
│   │   │   ├── processor.py          # Message processing pipeline (LLM path)
│   │   │   ├── shutdown.py           # Graceful shutdown handler
│   │   │   └── session_expiry.py     # Background session cleanup task
│   │   │
│   │   ├── agents/                   # Azure AI Foundry integration
│   │   │   ├── __init__.py
│   │   │   ├── client.py             # AIProjectClient setup & agent creation
│   │   │   ├── orchestrator.py       # Thread/Run management, tool call handling
│   │   │   ├── editor.py             # Editor agent (GPT-4o-mini) for response 
│   │   │   │                         # summarization
│   │   │   ├── tools/                # Tool function definitions
│   │   │   │   ├── __init__.py
│   │   │   │   ├── product_description.py   # product_description_api tool
│   │   │   │   └── product_gpt.py           # product_gpt_api tool
│   │   │   └── prompts.py            # System prompt constants (orchestrator + editor)
│   │   │
│   │   ├── resilience/               # Resilience patterns (NEW)
│   │   │   ├── __init__.py
│   │   │   ├── circuit_breaker.py    # pybreaker circuit breaker per dependency
│   │   │   ├── retry_policy.py       # tenacity retry policies with backoff
│   │   │   └── timeout.py            # httpx timeout configurations
│   │   │
│   │   ├── observability/            # Distributed tracing (NEW)
│   │   │   ├── __init__.py
│   │   │   ├── tracing.py            # OpenTelemetry setup, custom spans
│   │   │   ├── metrics.py            # Custom metrics (tokens, routing path, etc.)
│   │   │   └── correlation.py        # Correlation ID generation & propagation
│   │   │
│   │   ├── db/                       # Cosmos DB layer
│   │   │   ├── __init__.py
│   │   │   ├── client.py             # Cosmos DB client setup
│   │   │   ├── conversations.py      # conversations container CRUD
│   │   │   └── session_context.py    # session_context container CRUD
│   │   │
│   │   ├── services/                 # Business logic services
│   │   │   ├── __init__.py
│   │   │   ├── session_service.py    # Session lifecycle (create, validate, expire)
│   │   │   └── queue_service.py      # Service Bus enqueue/dequeue (session-aware)
│   │   │
│   │   └── models/                   # Pydantic models
│   │       ├── __init__.py
│   │       ├── requests.py           # API request schemas
│   │       ├── responses.py          # API response schemas
│   │       ├── websocket.py          # WebSocket/PubSub message schemas
│   │       └── db.py                 # Cosmos DB document schemas
│   │
│   ├── tests/
│   │   ├── __init__.py
│   │   ├── conftest.py
│   │   ├── test_auth.py
│   │   ├── test_chat.py
│   │   ├── test_session.py
│   │   ├── test_worker.py
│   │   ├── test_preprocessor.py      # Deterministic pre-processing tests (NEW)
│   │   ├── test_agents.py
│   │   ├── test_resilience.py        # Circuit breaker + retry tests (NEW)
│   │   └── test_scaling.py           # Graceful shutdown tests
│   │
│   ├── pyproject.toml                # Python project config
│   ├── requirements.txt              # Pin dependencies
│   └── Dockerfile
│
├── frontend/                         # Next.js application
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.tsx            # Root layout with MSAL provider
│   │   │   ├── page.tsx              # Landing / redirect to chat
│   │   │   ├── chat/
│   │   │   │   └── page.tsx          # Chat UI page
│   │   │   └── login/
│   │   │       └── page.tsx          # Login page
│   │   │
│   │   ├── components/
│   │   │   ├── ChatWindow.tsx        # Main chat container
│   │   │   ├── MessageBubble.tsx     # Individual message display
│   │   │   ├── StatusIndicator.tsx   # "Searching...", "Processing..." indicator
│   │   │   ├── QueuePositionBadge.tsx # "Queued (position 2)" badge
│   │   │   ├── DisambiguationCard.tsx # Multi-match selection UI
│   │   │   ├── SessionExpiredBanner.tsx
│   │   │   └── ChatInput.tsx         # Message input with send button
│   │   │
│   │   ├── hooks/
│   │   │   ├── useWebPubSub.ts       # Web PubSub WebSocket connection & messages
│   │   │   ├── useChat.ts            # Chat state management (incl. queue tracking)
│   │   │   └── useSession.ts         # Session lifecycle management
│   │   │
│   │   ├── services/
│   │   │   ├── api.ts                # REST API client (axios/fetch)
│   │   │   ├── auth.ts               # MSAL configuration & token management
│   │   │   └── pubsub.ts             # Web PubSub client wrapper
│   │   │
│   │   ├── types/
│   │   │   └── index.ts              # TypeScript interfaces
│   │   │
│   │   └── lib/
│   │       └── msalConfig.ts         # Azure Entra ID app registration config
│   │
│   ├── package.json
│   ├── tsconfig.json
│   ├── next.config.js
│   └── Dockerfile
│
├── infra/                            # Infrastructure as Code
│   ├── main.bicep                    # Azure Bicep deployment
│   ├── modules/
│   │   ├── apim.bicep                # API Management
│   │   ├── app-service.bicep         # App Service + autoscale rules
│   │   ├── cosmos-db.bicep
│   │   ├── service-bus.bicep
│   │   ├── ai-foundry.bicep
│   │   ├── ai-search.bicep
│   │   ├── web-pubsub.bicep          # Azure Web PubSub
│   │   └── entra-id.bicep
│   └── parameters/
│       ├── dev.bicepparam
│       └── prod.bicepparam
│
├── .env.example                      # Environment variable template
├── docker-compose.yml                # Local development
└── README.md
```

---

## 12. Key Dependencies

### Backend (Python 3.11+)

```
fastapi==0.115.*
uvicorn[standard]==0.34.*
azure-ai-projects==1.0.0b*             # Azure AI Foundry Agent SDK
azure-identity==1.19.*                  # DefaultAzureCredential
azure-servicebus==7.13.*               # Service Bus client
azure-cosmos==4.9.*                     # Cosmos DB client
azure-search-documents==11.6.*         # AI Search (for admin/setup scripts)
azure-messaging-webpubsubservice==1.*  # Web PubSub server SDK
azure-monitor-opentelemetry-exporter==1.*  # OpenTelemetry → Azure Monitor (NEW)
opentelemetry-api==1.*                  # OpenTelemetry API (NEW)
opentelemetry-sdk==1.*                  # OpenTelemetry SDK (NEW)
opentelemetry-instrumentation-fastapi==0.*  # Auto-instrumentation (NEW)
opentelemetry-instrumentation-httpx==0.*    # Auto-instrumentation (NEW)
httpx==0.28.*                           # Async HTTP client for tool API calls
pydantic-settings==2.7.*               # Config management
pybreaker==1.*                          # Circuit breaker pattern (NEW)
tenacity==9.*                           # Retry with exponential backoff (NEW)
```

### Frontend (Node.js 20+)

```
next@15
react@19
@azure/msal-react@2                    # MSAL authentication
@azure/msal-browser@3                  # MSAL browser client
@azure/web-pubsub-client@1             # Web PubSub client SDK
typescript@5
```

---

## 13. Error Handling, Resilience & Fallback Behavior

### 13.1 — Error Handling Matrix

| Failure Scenario | Timeout | Retries | Backoff | Behavior | WebSocket Message |
|---|---|---|---|---|---|
| **Product Description API timeout/5xx** | 10s | 3 | Exponential (1s, 2s, 4s) | Retry. If still fails → respond with error. Log to App Insights. | `{ "type": "error", "error_code": "PRODUCT_API_TIMEOUT" }` |
| **Product GPT API timeout/5xx** | 15s | 2 | Exponential (2s, 4s) | Respond with AI Search data only (if available). If neither → generic error. | `{ "type": "error", "error_code": "GPT_API_TIMEOUT" }` |
| **Azure AI Search unavailable** | 10s | 2 | Exponential (1s, 2s) | Skip grounding, use Product GPT alone. Add caveat about information freshness. | Status: *"Answering from product database (knowledge base temporarily unavailable)..."* |
| **Azure AI Foundry Run error/timeout** | 30s | 1 | None | Worker catches exception, sends error via Web PubSub, logs full error. | `{ "type": "error", "error_code": "AGENT_ERROR" }` |
| **Product Description API returns 0 results** | — | — | — | Trigger fallback to Product GPT + AI Search grounding (§4.6). NOT an error — designed fallback. | Status: *"Product not found in catalog. Searching knowledge base..."* |
| **Cosmos DB read failure** | 5s | 3 | Exponential (500ms, 1s, 2s) | Proceed without session context (degraded mode). Log warning. | No user-facing message. |
| **Cosmos DB write failure** | 5s | 3 | Exponential (500ms, 1s, 2s) | Non-blocking — response still delivered via Web PubSub. Background retry. | No user-facing message. |
| **Web PubSub push failure** | 5s | 1 | None | Retry once. If still fails, response is in Cosmos DB — frontend can poll via `GET /session/{id}/history`. | Frontend falls back to REST polling. |
| **Service Bus poison message** | — | Max delivery: 3 | — | Dead-letter queue (DLQ). Monitor via App Insights alerts. | `{ "type": "error", "error_code": "QUEUE_ERROR" }` |
| **Session expired during processing** | — | — | — | Worker completes current message, marks response with session status. | `{ "type": "session_expired" }` |
| **App Service instance removed (scale-down)** | — | — | — | Graceful shutdown: finish current message, release locks. WebSocket survives on Web PubSub. | No user-facing impact. |

> **Implementation Note:** Use `tenacity` for retry policies with exponential backoff. Configure timeout via `httpx` client timeout settings. All retry attempts and final failures are recorded as OpenTelemetry spans.

### 13.2 — Circuit Breaker Pattern

Circuit breakers prevent cascading failures by stopping calls to a failing dependency and allowing it time to recover.

**Circuit Breaker States:**

```text
         5 consecutive failures
         OR >50% failure rate in 60s
    ┌──────────────────────────────────────┐
    │                                      ▼
 CLOSED ◄──── probe succeeds ──── HALF-OPEN ◄──── 30s elapsed ──── OPEN
 (normal)                         (1 probe                        (all calls
                                   request)                        short-circuit
                                      │                            with fallback)
                                      │ probe fails
                                      └──────────────────────────► OPEN
```

**Configuration per dependency:**

| Dependency | Failure Threshold | Open Duration | Fallback When Open |
|-----------|-------------------|---------------|-------------------|
| Product Description API | 5 failures / 60s | 30s | Return: *"Product lookup is temporarily unavailable."* |
| Product GPT API | 5 failures / 60s | 30s | Return: *"General product information is temporarily unavailable."* |
| Azure AI Foundry | 3 failures / 60s | 60s | Return generic error; suggest retrying later |
| Azure AI Search | 5 failures / 60s | 30s | Skip search; proceed without search results |

> **Note on Cosmos DB:** Do NOT apply a custom circuit breaker to Cosmos DB. Cosmos DB has built-in throttling (HTTP 429 with `retry-after` headers) and the Azure SDK handles retries automatically.

**Implementation:**

```python
import pybreaker

product_api_breaker = pybreaker.CircuitBreaker(
    fail_max=5,
    reset_timeout=30,
    listeners=[CircuitBreakerMetricsListener()]  # Emits OpenTelemetry events
)

@product_api_breaker
async def call_product_api(part_number: str) -> dict:
    async with httpx.AsyncClient(timeout=10.0) as client:
        response = await client.get(f"{PRODUCT_API_URL}/products/{part_number}")
        response.raise_for_status()
        return response.json()
```

---

## 14. Observability: Distributed Tracing & Monitoring

### 14.1 — Distributed Tracing (OpenTelemetry)

End-to-end distributed tracing is implemented using **OpenTelemetry** with the **Azure Monitor OpenTelemetry Exporter**.

```text
OpenTelemetry SDK (Python)
    │
    ├─ Auto-instrumentation (zero-code):
    │     ├─ FastAPI (request/response spans)
    │     ├─ httpx / aiohttp (outbound HTTP calls)
    │     ├─ azure-servicebus (enqueue/dequeue spans)
    │     └─ azure-cosmos (read/write spans)
    │
    ├─ Custom spans (manual instrumentation):
    │     ├─ "pre_process.regex_scan" (input query, match result)
    │     ├─ "pre_process.normalize" (input, output candidates)
    │     ├─ "ai_foundry.create_thread" (new/existing, thread_id)
    │     ├─ "ai_foundry.create_run" (thread_id, run_id)
    │     ├─ "ai_foundry.tool_call" (tool_name, duration, status)
    │     ├─ "product_api.search" (query, result_count, duration)
    │     ├─ "general_ai.query" (query summary, duration)
    │     ├─ "content_safety.check" (result, duration)
    │     ├─ "editor.summarize" (input_tokens, output_tokens, duration)
    │     └─ "circuit_breaker.state_change" (dependency, old_state, new_state)
    │
    ├─ Standard attributes on every span:
    │     ├─ correlation_id (generated at Gateway, propagated via Service Bus message)
    │     ├─ session_id
    │     ├─ user_id
    │     └─ message_id
    │
    ├─ AI-specific attributes:
    │     ├─ ai_foundry.run_id
    │     ├─ ai_foundry.thread_id
    │     ├─ ai.prompt_tokens
    │     ├─ ai.completion_tokens
    │     ├─ ai.total_tokens
    │     └─ ai.model_name
    │
    └─ Exporter → Azure Monitor (Application Insights)
         └─ azure-monitor-opentelemetry-exporter package
```

**Correlation ID Propagation:**

1.  **Generated** at the FastAPI Gateway when a request arrives (UUID v4).
2.  **Included** in the Service Bus message payload.
3.  **Extracted** by the Worker on dequeue and set as the active trace context.
4.  **Propagated** to all outbound HTTP calls via W3C Trace Context headers.
5.  **Stored** in the Cosmos DB `conversations` document for post-hoc querying.

### 14.2 — Monitoring & Alerting

| Metric | Source | Alert Threshold |
|--------|--------|-----------------|
| Service Bus DLQ message count | Azure Monitor | > 0 → immediate alert |
| Service Bus ActiveMessageCount | Azure Monitor | > 30 → critical (scaling may be insufficient) |
| Product Description API P95 latency | OpenTelemetry spans | > 5s → warning |
| Product GPT API P95 latency | OpenTelemetry spans | > 15s → warning |
| AI Foundry Run failure rate | Custom spans | > 5% → critical |
| Circuit breaker OPEN | Custom events | Any dependency transitions to OPEN |
| Web PubSub connection count | Azure Monitor | > 80% of tier limit → scale warning |
| Cosmos DB RU consumption | Azure Monitor | > 80% burst → warning |
| App Service instance count | Azure Monitor | = max (5) → capacity warning |
| Token consumption (daily) | Custom metrics | Exceeds daily budget threshold |
| Deterministic routing % | Custom metrics | Track what % of queries skip the LLM |

### 14.3 — Key Debugging Queries

| Question | How to Answer |
|----------|---------------|
| "Why was session X slow?" | Query Application Insights by `session_id`, inspect span durations |
| "How much are we spending on tokens?" | Aggregate `ai.prompt_tokens` + `ai.completion_tokens` metrics |
| "What % of queries skip the LLM?" | Count spans where `routing_path = "deterministic"` vs. `"ai_foundry"` |
| "Is Product API degraded?" | Check circuit breaker state change events + `product_api.search` span error rates |
| "Show me the full trace for message X" | Query by `correlation_id` → see Gateway → Worker → all tool calls |

---

## 15. Cost Estimate (Monthly, Low-Medium Traffic)

| Service | Tier / SKU | Est. Monthly Cost |
|---|---|---|
| Azure APIM | Consumption tier | ~$3-10 (per 1M calls) |
| Azure App Service (auto-scaled 1-5) | B1 (Basic) × 1-5 | ~$13-65 |
| Azure AI Foundry (GPT-4o orchestrator) | Pay-as-you-go | ~$20-80 |
| Azure AI Foundry (GPT-4o-mini editor) | Pay-as-you-go | ~$2-8 |
| Azure Cosmos DB | Serverless | ~$1-10 |
| Azure Service Bus | Standard | ~$10 |
| Azure AI Search | Basic (web crawler) | ~$70 |
| Azure OpenAI (embeddings) | text-embedding-3-large | ~$5-15 |
| Azure Web PubSub | Free → Standard | $0-50 |
| Azure Entra ID | Free tier | $0 |
| Azure AI Content Safety | Free tier (1K calls/mo) | $0 |
| Azure Application Insights | Free tier (5 GB/mo) | $0* |
| **Total Estimate** | | **~$125-320/mo** |

> **Cost drivers:** App Service scaling (depends on load) and Azure AI Search Basic ($70/mo fixed). Web PubSub Free tier is sufficient for dev/low traffic (20 concurrent connections). For production, upgrade to Standard ($50/mo for 1K concurrent connections).

> ***Application Insights:** The free tier includes 5 GB/month of data ingestion. With full OpenTelemetry tracing, high-traffic scenarios may exceed this. Consider sampling (50% rate) if costs rise. At $2.30/GB overage, budget ~$5-15/month for medium-traffic tracing.

---

## 16. Implementation Phases (Recommended Order)

| Phase | Components | Deliverable |
|-------|-----------|-------------|
| **Phase 1** | Infra setup (Bicep): App Service, Cosmos DB, Service Bus, APIM, Web PubSub, AI Foundry project | Deployed empty infrastructure |
| **Phase 2** | FastAPI skeleton, APIM subscription key validation, session endpoints, Web PubSub token generation | Auth + session creation working |
| **Phase 3** | Azure AI Foundry agent creation (orchestrator + editor), tool definitions, orchestrator logic | Agent responds to messages (no queue yet) |
| **Phase 4** | Service Bus integration (session-aware), worker consumer with graceful shutdown, end-to-end message flow | Full queue-based pipeline working |
| **Phase 5** | Deterministic pre-processing layer: regex scanner, part number normalization, direct API calls | Simple queries bypass LLM |
| **Phase 6** | Azure AI Search indexer setup, web crawler configuration, knowledge base population | Knowledge base populated |
| **Phase 7** | Product GPT + AI Search grounding, editor model integration, fallback logic | Fallback + editing path working |
| **Phase 8** | Resilience patterns: circuit breakers, timeout policies, retry with backoff | Resilient external calls |
| **Phase 9** | OpenTelemetry distributed tracing, custom spans, correlation ID propagation, Azure Monitor dashboards | Full observability |
| **Phase 10** | Next.js frontend: MSAL auth, Web PubSub client, chat UI, status indicators, queue position badges | Full UI working |
| **Phase 11** | Multi-message FIFO queuing (N messages), session expiry, disambiguation UI, queue position updates | Edge cases handled |
| **Phase 12** | Auto-scale rules (Bicep), graceful shutdown testing, error handling, monitoring alerts, Content Safety | Production-ready |

---

## 17. Environment Variables

```bash
# Azure Entra ID
AZURE_TENANT_ID=<tenant_id>
AZURE_CLIENT_ID=<app_registration_client_id>
AZURE_AUTHORITY=https://login.microsoftonline.com/<tenant_id>

# Azure APIM
APIM_SUBSCRIPTION_KEY=<internal_subscription_key>  # for validating requests came through APIM
APIM_GATEWAY_URL=https://apim.company.com           # for frontend to know where to send requests

# Azure AI Foundry
AI_FOUNDRY_CONNECTION_STRING=<connection_string>
AI_FOUNDRY_ORCHESTRATOR_MODEL=gpt-4o
AI_FOUNDRY_EDITOR_MODEL=gpt-4o-mini
ORCHESTRATOR_AGENT_ID=<created_agent_id>
EDITOR_AGENT_ID=<created_agent_id>

# Azure Cosmos DB
COSMOS_DB_ENDPOINT=https://<account>.documents.azure.com:443/
COSMOS_DB_DATABASE=enterprise-ai-assistant
COSMOS_CONVERSATIONS_CONTAINER=conversations
COSMOS_SESSION_CONTEXT_CONTAINER=session_context

# Azure Service Bus
SERVICE_BUS_NAMESPACE=<namespace>.servicebus.windows.net
SERVICE_BUS_QUEUE_NAME=chat-queue
SERVICE_BUS_MAX_DELIVERY_COUNT=3

# Azure AI Search
AI_SEARCH_ENDPOINT=https://<service>.search.windows.net
AI_SEARCH_INDEX_NAME=product-knowledge-index
AI_SEARCH_CONNECTION_ID=<ai_foundry_connection_id>

# Azure Web PubSub
WEB_PUBSUB_CONNECTION_STRING=<connection_string>
WEB_PUBSUB_HUB_NAME=chat

# External APIs
PRODUCT_DESCRIPTION_API_BASE_URL=https://api.company.com/v1
PRODUCT_GPT_API_BASE_URL=https://product-gpt.company.com/api/v1

# Session
SESSION_TIMEOUT_MINUTES=10
CONTEXT_OPTIMIZATION_TURN_THRESHOLD=10

# Resilience (NEW)
PRODUCT_API_TIMEOUT_SECONDS=10
PRODUCT_API_RETRY_COUNT=3
PRODUCT_GPT_TIMEOUT_SECONDS=15
PRODUCT_GPT_RETRY_COUNT=2
AI_FOUNDRY_TIMEOUT_SECONDS=30
CIRCUIT_BREAKER_FAIL_MAX=5
CIRCUIT_BREAKER_RESET_TIMEOUT=30

# Observability (NEW)
APPLICATIONINSIGHTS_CONNECTION_STRING=<connection_string>
OTEL_SERVICE_NAME=enterprise-ai-assistant
OTEL_TRACES_SAMPLER_ARG=1.0  # 1.0 = 100% sampling, reduce for high traffic

# Scaling
AUTOSCALE_MIN_INSTANCES=1
AUTOSCALE_MAX_INSTANCES=5
AUTOSCALE_QUEUE_THRESHOLD=10

# App
APP_HOST=0.0.0.0
APP_PORT=8000
```

---

## 18. Future Enhancements (Post-v1)

These items are intentionally deferred to keep v1 simple and evolvable:

*   **Additional Part Number Prefixes:** Extend the normalization regex to handle other prefixes (e.g., "HY_", "PMP_") as business needs emerge. The deterministic pre-processor supports this via regex configuration — no code changes required for simple patterns.
*   **Multi-Tenant Support:** Service Bus topics/subscriptions can partition traffic by tenant. Cosmos DB partition keys already support per-session isolation.
*   **Conversation Analytics:** Cosmos DB's queryable history enables dashboards for most-searched parts, common disambiguation scenarios, and user engagement metrics.
*   **Streaming Responses:** Upgrade from Web PubSub push to token-by-token streaming for real-time AI response rendering.
*   **Feedback Loop:** Allow users to rate responses, feeding data back into prompt optimization.
*   **Advanced Pre-Processor:** Expand the deterministic pre-processor with additional pattern matchers (e.g., order number lookups, serial number queries) as usage patterns emerge from tracing data.
*   **Adaptive Sampling:** Implement OpenTelemetry adaptive sampling to reduce tracing costs at high volume while preserving error and slow-request traces.
*   **Semantic Caching:** Cache frequent part number lookups to reduce Product Description API calls and improve latency.
