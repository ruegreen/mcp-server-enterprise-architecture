# Authentication Pattern

The framework implements a dual-layer authentication model: north-facing (agent-to-server) and south-facing (server-to-API).

---

## North-Facing Authentication (Agent → MCP Server)

AI agents authenticate to the MCP server via API key middleware on HTTP/SSE transports. Stdio transport is exempt (local process communication).

```javascript
function authenticateApiKey(req, res, next) {
  // Health and root endpoints bypass auth for monitoring
  if (req.path.endsWith('/health') || req.path === '/') {
    return next();
  }

  // Backward compatible: skip if no key configured
  if (!config.mcpApiKey) {
    return next();
  }

  const apiKey = req.headers['x-api-key'];

  if (!apiKey) {
    return res.status(401).json({
      jsonrpc: '2.0',
      error: { code: -32001, message: 'Unauthorized: Missing X-API-Key header' },
      id: null
    });
  }

  if (apiKey !== config.mcpApiKey) {
    return res.status(403).json({
      jsonrpc: '2.0',
      error: { code: -32002, message: 'Forbidden: Invalid API key' },
      id: null
    });
  }

  next();
}

// Applied to Express middleware stack
app.use(authenticateApiKey);
```

---

## South-Facing Authentication (MCP Server → Enterprise APIs)

Five authentication strategies are implemented across the server ecosystem, each suited to different enterprise API requirements.

### Strategy 1: API Key Header

The simplest pattern. API key is injected as a custom header.

```javascript
getAuthHeaders() {
  const headers = { 'Content-Type': 'application/json' };

  if (this.authMode === 'apikey' && this.apiKey) {
    headers['x-api-key'] = this.apiKey;
  }

  return headers;
}
```

**Used by:** Retail, Insurance, Healthcare, Pod Management, Triage

### Strategy 2: JWT Bearer Token

Pre-obtained JWT token set in Authorization header.

```javascript
getAuthHeaders() {
  const headers = { 'Content-Type': 'application/json' };

  if (this.authMode === 'jwt' && this.jwtToken) {
    headers['Authorization'] = `Bearer ${this.jwtToken}`;
  }

  return headers;
}
```

**Used by:** Retail, Insurance, Healthcare (as alternative to API key)

### Strategy 3: Basic Authentication

Username/password encoded as Base64 for legacy platform APIs.

```javascript
getAuthHeaders() {
  const credentials = Buffer.from(
    `${this.username}:${this.password}`
  ).toString('base64');

  return {
    'Authorization': `Basic ${credentials}`,
    'Content-Type': 'application/json'
  };
}
```

**Used by:** ITSM Integration

### Strategy 4: OAuth2 via API Gateway (Token Broker)

A centralized API gateway manages OAuth token lifecycle. MCP servers request tokens via the gateway rather than handling OAuth flows directly.

```javascript
class GatewayTokenClient {
  constructor() {
    this.cachedToken = null;
    this.tokenExpiry = null;
  }

  async getAccessToken() {
    // Return cached token if still valid (with 60-second buffer)
    if (this.cachedToken && Date.now() < (this.tokenExpiry - 60000)) {
      return this.cachedToken;
    }

    const response = await fetch(
      `${gatewayUrl}/integrationtokens/${integrationName}`,
      {
        headers: {
          'x-token-passphrase': config.gatewayTokenPassphrase,
          'Accept': 'application/json'
        }
      }
    );

    const data = await response.json();
    this.cachedToken = data.token;
    this.tokenExpiry = Date.now() + (data.expiresIn * 1000);
    return this.cachedToken;
  }

  async getAuthHeaders() {
    const token = await this.getAccessToken();
    return { 'Authorization': `Bearer ${token}` };
  }
}
```

**Used by:** EHR integrations (production mode), CRM proxy

### Strategy 5: OAuth2 Local Browser Flow

For desktop development, a local Express server handles the OAuth callback after the user authorizes in their browser.

```javascript
class LocalOAuth {
  async authorize() {
    if (this.isTokenValid()) return this.accessToken;

    if (this.refreshToken) {
      try { return await this.refreshAccessToken(); }
      catch { /* fall through to new auth */ }
    }

    return new Promise((resolve, reject) => {
      const state = randomUUID();
      const authUrl = this.buildAuthorizationUrl(state);

      // Start local callback server
      const callbackApp = express();
      callbackApp.get('/callback', async (req, res) => {
        const { code, state: returnedState } = req.query;
        // Exchange code for token
        const tokenResponse = await this.exchangeCodeForToken(code);
        this.accessToken = tokenResponse.access_token;
        this.refreshToken = tokenResponse.refresh_token;
        resolve(this.accessToken);
      });

      const server = callbackApp.listen(callbackPort);

      // Open browser for user authorization
      openBrowser(authUrl);

      // Timeout after 2 minutes
      setTimeout(() => reject(new Error('Authorization timeout')), 120000);
    });
  }
}
```

**Used by:** EHR integrations (local development mode)

---

## Authentication Abstraction Layer

An `AuthManager` class abstracts away the authentication strategy, allowing tool handlers to call `authorize()` without knowing which method is active.

```javascript
class AuthManager {
  async initialize() {
    if (config.authMode === 'gateway') {
      this.provider = new GatewayTokenClient();
    } else if (config.authMode === 'local') {
      this.provider = new LocalOAuth();
    }
    await this.provider.initialize();
  }

  async authorize() {
    return this.provider.getAccessToken();
  }

  isAuthenticated() {
    return this.provider.hasValidToken();
  }

  async cleanup() {
    await this.provider.cleanup?.();
  }
}
```

**Usage in tool handlers:**
```javascript
case 'find_patient': {
  await authManager.authorize();  // Ensure valid token
  const result = await fhirClient.searchPatients(args);
  return formatResult(result);
}
```

---

## Token Refresh on 401 (Proxy Pattern)

The CRM proxy server implements automatic token refresh when receiving 401 responses from upstream:

```javascript
async forwardToUpstream(sessionId, mcpRequest) {
  let response = await fetch(upstreamUrl, {
    method: 'POST',
    headers: await this.authClient.getAuthHeaders(),
    body: JSON.stringify(mcpRequest)
  });

  if (response.status === 401) {
    await this.authClient.refreshToken();
    response = await fetch(upstreamUrl, {
      method: 'POST',
      headers: await this.authClient.getAuthHeaders(),
      body: JSON.stringify(mcpRequest)
    });
  }

  return await response.json();
}
```

---

## Authentication Strategy by Integration Type

| Integration | North-Facing | South-Facing | Token Mgmt |
|---|---|---|---|
| EHR (Patient) | API Key | OAuth2 (Gateway or Local) | Gateway caches, Local refreshes |
| EHR (Clinical) | API Key | OAuth2 (Gateway or Local) | Gateway caches, Local refreshes |
| EHR (Agent Care) | None (stdio) | OAuth2 (Browser Flow) | In-memory with refresh |
| ITSM | API Key | Basic / OAuth / API Key | Config-selected |
| CRM (Proxy) | API Key | Gateway Token → Upstream | 401 retry with refresh |
| Clinical Triage | API Key | API Key | Static |
| Pharmacy | API Key | None (Public API) | N/A |
| Retail/Insurance | API Key | API Key / JWT | Static |
