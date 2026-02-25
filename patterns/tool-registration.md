# Tool Registration Pattern

How tools are defined, registered, and dispatched across the MCP server framework.

---

## Tool Definition

Tools are defined as arrays of objects conforming to the MCP tool schema. Each tool specifies its name, description, and a JSON Schema for input validation.

### Inline Definition (Simple Servers)

```javascript
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: 'get_record',
        description: 'Retrieve a record by ID or phone number',
        inputSchema: {
          type: 'object',
          properties: {
            number: {
              type: 'string',
              description: 'Record ID or phone number to look up'
            }
          },
          required: ['number']
        }
      },
      {
        name: 'list_records',
        description: 'List records with pagination',
        inputSchema: {
          type: 'object',
          properties: {
            limit: { type: 'number', default: 100 },
            skip: { type: 'number', default: 0 }
          }
        }
      }
    ]
  };
});
```

### Modular Definition (Complex Servers)

For servers with many tools (20+), definitions are extracted to a dedicated module:

```javascript
// tools.js — Centralized tool definitions
export const TOOL_DEFINITIONS = [
  {
    name: 'find_patient',
    description: 'Search for patients by name, DOB, or identifier',
    inputSchema: {
      type: 'object',
      properties: {
        lastName: { type: 'string' },
        firstName: { type: 'string' },
        birthDate: { type: 'string' },
        identifier: { type: 'string' },
        _count: { type: 'number', default: 20 }
      }
    }
  },
  // ... 50+ more tool definitions
];

export async function handleToolCall(name, args) {
  switch (name) {
    case 'find_patient':
      return await fhirClient.searchPatients(args);
    // ... routing for all tools
    default:
      throw new Error(`Unknown tool: ${name}`);
  }
}
```

```javascript
// server.js — Uses the shared module
import { TOOL_DEFINITIONS, handleToolCall } from './tools.js';

server.setRequestHandler(ListToolsRequestSchema, async () => {
  return { tools: TOOL_DEFINITIONS };
});

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;
  try {
    const result = await handleToolCall(name, args);
    return {
      content: [{ type: 'text', text: JSON.stringify(result, null, 2) }]
    };
  } catch (error) {
    return {
      content: [{ type: 'text', text: JSON.stringify({ error: error.message }) }],
      isError: true
    };
  }
});
```

---

## Tool Dispatch Pattern

All servers use switch-statement routing for tool execution:

```javascript
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  try {
    switch (name) {
      case 'get_record': {
        const result = await apiClient.getRecord(args.number);
        return { content: [{ type: 'text', text: JSON.stringify(result, null, 2) }] };
      }
      case 'create_record': {
        const result = await apiClient.createRecord(args);
        return { content: [{ type: 'text', text: JSON.stringify(result, null, 2) }] };
      }
      default:
        throw new Error(`Unknown tool: ${name}`);
    }
  } catch (error) {
    return {
      content: [{
        type: 'text',
        text: JSON.stringify({
          error: error.message,
          details: 'Check API availability and authentication configuration.'
        })
      }],
      isError: true
    };
  }
});
```

---

## Dynamic Tool Discovery (Proxy Pattern)

For proxy-based servers that forward requests to upstream MCP servers, tools are discovered dynamically rather than statically defined:

```javascript
server.setRequestHandler(ListToolsRequestSchema, async (request, extra) => {
  const sessionId = extra?.sessionId || 'default';
  await waitForUpstreamInit(sessionId);

  // Forward the tools/list request to upstream MCP server
  const mcpRequest = {
    jsonrpc: '2.0',
    id: Date.now(),
    method: 'tools/list',
    params: request.params || {}
  };

  const response = await mcpProxy.forwardToUpstream(sessionId, mcpRequest);
  return response.result || response;
});
```

This pattern enables a proxy server to expose whatever tools the upstream server provides without any hardcoded definitions.

---

## Tool Complexity Spectrum

| Complexity | Tools | Pattern | Example |
|---|---|---|---|
| Simple CRUD | 5 | Inline definitions, switch dispatch | Retail, Insurance |
| Domain-Specific | 14-21 | Inline or modular, specialized handlers | ITSM, Triage |
| Clinical/Complex | 17-50+ | Modular definitions, auth pre-flight, formatters | EHR integrations |
| Proxy/Dynamic | 0 (discovered) | Forward to upstream, transparent relay | CRM proxy |
