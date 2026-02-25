# Architecture Diagrams

Visual documentation of the enterprise MCP server framework architecture.

---

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph AI_Agents["AI Agents"]
        A1["Desktop Agent<br/>(stdio)"]
        A2["Cloud Agent<br/>(HTTP)"]
        A3["Legacy Agent<br/>(SSE)"]
    end

    subgraph MCP_Framework["MCP Server Framework"]
        AUTH["Auth Middleware"]
        TOOLS["Tool Registry"]
        RESOURCES["Resource Registry"]
        SESSION["Session Manager"]
    end

    subgraph Integrations["Platform Integrations"]
        EHR["EHR<br/>(FHIR R4/STU3)"]
        ITSM["ITSM<br/>(Table API)"]
        CRM["CRM<br/>(MCP Proxy)"]
        TRIAGE["Clinical Triage"]
        PHARMA["Pharmacy DB"]
        RETAIL["Retail / Insurance"]
    end

    subgraph APIs["Enterprise APIs"]
        EHR_API["EHR Platform"]
        ITSM_API["ITSM Platform"]
        CRM_API["CRM Platform"]
        TRIAGE_API["Triage Engine"]
        GW["API Gateway<br/>(Token Broker)"]
    end

    A1 & A2 & A3 --> AUTH
    AUTH --> TOOLS & RESOURCES
    A2 --> SESSION
    TOOLS --> EHR & ITSM & CRM & TRIAGE & PHARMA & RETAIL
    EHR --> EHR_API & GW
    ITSM --> ITSM_API
    CRM --> CRM_API & GW
    TRIAGE --> TRIAGE_API
```

---

## 2. Tool Registration and Discovery Flow

```mermaid
sequenceDiagram
    participant Client as AI Agent
    participant MCP as MCP Server
    participant Registry as Tool Registry
    participant Handler as Tool Handler
    participant API as Enterprise API

    Client->>MCP: tools/list request
    MCP->>Registry: getToolDefinitions()
    Registry-->>MCP: Array of tool schemas (name, description, inputSchema)
    MCP-->>Client: Available tools with JSON Schema definitions

    Client->>MCP: tools/call { name, arguments }
    MCP->>Handler: handleToolCall(name, args)
    Handler->>Handler: Validate arguments against schema
    Handler->>Handler: Route via switch(name)
    Handler->>API: Execute API call with auth headers
    API-->>Handler: API response
    Handler->>Handler: Format response (JSON or text)
    Handler-->>MCP: { content: [{ type: 'text', text: result }] }
    MCP-->>Client: Tool execution result
```

---

## 3. Request Lifecycle

```mermaid
flowchart TD
    A[Incoming MCP Request] --> B{Transport Type?}

    B -->|stdio| C[StdioServerTransport]
    B -->|HTTP| D[StreamableHTTPServerTransport]
    B -->|SSE| E[SSEServerTransport]

    D --> F{Has Session ID?}
    F -->|No| G[Create New Session + Transport]
    F -->|Yes| H[Lookup Existing Transport]
    G --> I[Store in transports map]

    C & H & I & E --> J{Auth Required?}
    J -->|HTTP/SSE| K[Validate X-API-Key Header]
    J -->|stdio| L[Skip Auth — Local Process]

    K -->|Valid| M[Route to Request Handler]
    K -->|Invalid| N[Return 401/403 JSON-RPC Error]
    L --> M

    M --> O{Request Method?}
    O -->|tools/list| P[Return Tool Definitions]
    O -->|tools/call| Q[Execute Tool Handler]
    O -->|resources/list| R[Return Resource List]
    O -->|resources/read| S[Fetch Resource Content]

    Q --> T[API Client: Build Request]
    T --> U[Inject Auth Headers]
    U --> V[Call Enterprise API]
    V --> W{Response OK?}
    W -->|Yes| X[Cache Response if Enabled]
    X --> Y[Format & Return Result]
    W -->|No| Z{Stale Cache Available?}
    Z -->|Yes| AA[Return Stale Cache + Warning]
    Z -->|No| AB[Return Error with isError: true]
```

---

## 4. Authentication Flow Abstraction

```mermaid
flowchart TD
    A[Tool Call Received] --> B{Auth Mode?}

    B -->|API Key| C[Read API Key from Config]
    C --> D["Set x-api-key Header"]
    D --> J[Make API Request]

    B -->|JWT Bearer| E[Read JWT Token from Config]
    E --> F["Set Authorization: Bearer Header"]
    F --> J

    B -->|OAuth2 — Gateway| G[Request Token from API Gateway]
    G --> G1{Token Cached & Valid?}
    G1 -->|Yes| G2[Use Cached Token]
    G1 -->|No| G3["GET /integrationtokens/{name}<br/>with x-token-passphrase"]
    G3 --> G4[Cache Token with Expiry Buffer]
    G4 --> G2
    G2 --> F

    B -->|OAuth2 — Local| H[Check Token Validity]
    H --> H1{Token Valid?}
    H1 -->|Yes| H2[Use Existing Token]
    H1 -->|Expired| H3{Refresh Token Available?}
    H3 -->|Yes| H4[Refresh Token]
    H3 -->|No| H5[Start Browser OAuth Flow]
    H5 --> H6[User Authorizes in Browser]
    H6 --> H7[Callback Server Receives Code]
    H7 --> H8[Exchange Code for Token]
    H4 & H8 --> H2
    H2 --> F

    B -->|Basic Auth| I["Base64(username:password)"]
    I --> I1["Set Authorization: Basic Header"]
    I1 --> J

    B -->|MCP Proxy| K[Forward to Upstream MCP Server]
    K --> K1[Inject Gateway Token]
    K1 --> K2{Response 401?}
    K2 -->|Yes| K3[Refresh Token & Retry]
    K2 -->|No| K4[Return Response]
    K3 --> K4

    J --> L{Response Status?}
    L -->|200| M[Return Data]
    L -->|401| N[Token Expired — Refresh & Retry]
    L -->|4xx/5xx| O[Throw Error]
```

---

## 5. Multi-Transport Architecture

```mermaid
graph LR
    subgraph Transports["Three Transport Options"]
        STDIO["stdio<br/>─────────<br/>Desktop Agent<br/>Local development<br/>Single process"]
        HTTP["Streamable HTTP<br/>─────────<br/>Cloud deployment<br/>Session-based<br/>Resumable streams<br/>Multi-client"]
        SSE["SSE<br/>─────────<br/>Legacy agents<br/>Browser-based<br/>Unidirectional"]
    end

    subgraph Shared["Shared Core"]
        SERVER["MCP Server Instance"]
        TOOLS["Tool Definitions"]
        HANDLERS["Tool Handlers"]
        CLIENTS["API Clients"]
        CONFIG["Configuration"]
    end

    STDIO --> SERVER
    HTTP --> SERVER
    SSE --> SERVER
    SERVER --> TOOLS
    SERVER --> HANDLERS
    HANDLERS --> CLIENTS
    CLIENTS --> CONFIG
```

---

## 6. Deployment Architecture

```mermaid
graph TB
    subgraph Cloud["Cloud Infrastructure (AWS)"]
        ALB["Application Load Balancer"]
        subgraph ECS["ECS Fargate Cluster"]
            T1["MCP Server Task 1<br/>(Docker Container)"]
            T2["MCP Server Task 2<br/>(Docker Container)"]
            T3["MCP Server Task N<br/>(Docker Container)"]
        end
        ECR["Container Registry<br/>(ECR)"]
        SM["Secrets Manager"]
        CW["CloudWatch Logs"]
    end

    subgraph CI["CI/CD Pipeline"]
        GH["GitHub Repository"]
        GA["GitHub Actions"]
    end

    subgraph Local["Local Development"]
        CD["Desktop Agent<br/>(stdio transport)"]
        ENV[".env Configuration"]
    end

    GH -->|push to main| GA
    GA -->|build & push| ECR
    GA -->|deploy| ECS
    ECR --> T1 & T2 & T3
    SM -->|inject secrets| T1 & T2 & T3
    T1 & T2 & T3 -->|logs| CW
    ALB --> T1 & T2 & T3
    T1 & T2 & T3 -->|health check| ALB

    CD -->|stdio| ENV
```

---

## 7. Platform Integration Comparison

| Integration | Tools | Auth Model | Complexity | Key Pattern |
|---|---|---|---|---|
| EHR (Patient-facing) | 17 | OAuth2 (Gateway + Local) | High | FHIR R4, research APIs, caching |
| EHR (Clinical) | 50+ | OAuth2 (Gateway + Local) | Very High | FHIR R4+STU3, write ops, 40+ formatters |
| EHR (Agent Care) | 17 | OAuth2 (Browser Flow) | High | TypeScript, NLP query parsing, prompts |
| ITSM | 21 | Basic / OAuth / API Key | Medium | Table API queries, ID resolution |
| CRM (Proxy) | Dynamic | Gateway Token + Proxy | High | Transparent MCP proxy, session mapping |
| Clinical Triage | 14 | API Key | Medium | NLP intake, 3 conversation modes, trimming |
| Pharmacy DB | 3 | None (Public API) | Low | Retry + backoff, TTL caching |
| Retail / Insurance | 5 each | API Key / JWT | Low | Standard CRUD, multi-transport |
| Lab Pod Mgmt | 16 | API Key / JWT | Medium | Multi-collection, keyword management |
