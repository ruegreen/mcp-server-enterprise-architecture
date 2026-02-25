# Server Framework Pattern

The core pattern for building an MCP server that supports multiple transport protocols from a single codebase.

---

## Multi-Transport Server Architecture

Each MCP server supports three transport mechanisms from the same tool definitions, allowing deployment flexibility across local development and cloud production environments.

### Transport Selection

```
├── src/index.js        → stdio transport   (local desktop agent)
├── src/server-http.js  → Streamable HTTP   (cloud deployment, production)
└── src/server-sse.js   → SSE transport     (legacy agents, browser-based)
```

### Shared Server Creation Pattern

```javascript
// All three transports instantiate the same MCP server with identical capabilities
const server = new Server(
  { name: 'platform-mcp-server', version: '1.0.0' },
  { capabilities: { resources: {}, tools: {} } }
);

// Register identical handlers across all transports
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return { tools: toolDefinitions };
});

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;
  return await handleToolCall(name, args);
});

server.setRequestHandler(ListResourcesRequestSchema, async () => {
  return { resources: resourceDefinitions };
});

server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  return await handleResourceRead(request.params.uri);
});
```

### Transport-Specific Initialization

**Stdio (Local Desktop Agent):**
```javascript
const transport = new StdioServerTransport();
await server.connect(transport);
```

**Streamable HTTP (Cloud Production):**
```javascript
const app = express();
app.use(cors());
app.use(express.json());

// Session-based transport with resumable streams
const transports = {};
const eventStore = new InMemoryEventStore();

app.post('/mcp', async (req, res) => {
  const sessionId = req.headers['mcp-session-id'];

  if (!sessionId) {
    // New session — create transport
    const transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: () => randomUUID(),
      eventStore,
      onsessioninitialized: (sid) => {
        transports[sid] = transport;
      }
    });
    await server.connect(transport);
    await transport.handleRequest(req, res, req.body);
  } else {
    // Existing session — reuse transport
    const transport = transports[sessionId];
    await transport.handleRequest(req, res, req.body);
  }
});
```

### Health Check Pattern

Every HTTP/SSE server exposes an unauthenticated health endpoint for container orchestration:

```javascript
app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    service: 'platform-mcp-server',
    version: '1.0.0',
    transport: 'streamable-http',
    timestamp: new Date().toISOString(),
  });
});
```

### Graceful Shutdown

```javascript
process.on('SIGINT', async () => {
  for (const sessionId in transports) {
    try {
      await transports[sessionId].close();
      delete transports[sessionId];
    } catch (error) {
      console.error(`Error closing session ${sessionId}:`, error);
    }
  }
  process.exit(0);
});
```

---

## Key Design Decisions

1. **Same tool logic, different transports** — Business logic is transport-agnostic. Only the protocol binding changes.
2. **Session management for HTTP only** — Stdio is single-process, SSE is stateless-ish, but Streamable HTTP requires explicit session tracking.
3. **Health checks bypass auth** — Enables load balancer and container orchestration probes without credentials.
4. **In-memory event store** — Enables resumable streams for Streamable HTTP without external dependencies. Suitable for stateless cloud deployments where sessions are ephemeral.
