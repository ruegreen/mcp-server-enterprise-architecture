# API Integration Patterns

Patterns for integrating MCP servers with enterprise APIs across different domains and complexity levels.

---

## Singleton API Client

Every integration uses a singleton client pattern — one instance per platform, shared across all tool handlers and transports.

```javascript
class PlatformClient {
  constructor() {
    this.baseUrl = config.apiBaseUrl;
    this.authMode = config.authMode;
  }

  async makeRequest(url, options = {}) {
    const response = await fetch(url, {
      ...options,
      headers: {
        ...this.getAuthHeaders(),
        ...options.headers
      }
    });

    const data = await response.json();

    if (!response.ok) {
      throw new Error(
        data.message || `API request failed with status ${response.status}`
      );
    }

    return data;
  }
}

// Singleton export
export const platformClient = new PlatformClient();
```

---

## FHIR R4 Integration Pattern (Healthcare)

The most complex integration pattern, supporting FHIR standard healthcare APIs with query parameter mapping and dual version support.

```javascript
class FhirClient {
  constructor(authManager) {
    this.authManager = authManager;
    this.baseUrl = config.fhirBaseUrl;      // R4 endpoint
    this.stu3Url = config.fhirStu3BaseUrl;  // STU3 endpoint (legacy resources)
  }

  async request(endpoint, params = {}, cacheKey = null) {
    const token = await this.authManager.getAccessToken();
    const url = new URL(`${this.baseUrl}/${endpoint}`);

    // Filter null params and append to URL
    Object.entries(params).forEach(([key, value]) => {
      if (value !== undefined && value !== null) {
        url.searchParams.append(key, value);
      }
    });

    const response = await axios.get(url.toString(), {
      headers: {
        'Authorization': `Bearer ${token}`,
        'Accept': 'application/fhir+json'
      },
      timeout: 30000
    });

    return response.data;
  }

  // FHIR search parameter mapping
  async searchPatients(params = {}) {
    return this.request('Patient', {
      family: params.lastName,
      given: params.firstName,
      birthdate: params.birthDate,
      gender: params.gender,
      identifier: params.identifier,
      phone: params.phone,
      email: params.email,
      'address-city': params.addressCity,
      'address-state': params.addressState,
      _count: Math.min(params._count || 20, 100)
    });
  }
}
```

---

## ITSM Table API Pattern

Query-string-based filtering with ITSM-style query operators:

```javascript
async listIncidents(filters = {}, limit = 100, offset = 0) {
  const query = [];

  if (filters.state) query.push(`state=${filters.state}`);
  if (filters.priority) query.push(`priority=${filters.priority}`);
  if (filters.assigned_to) query.push(`assigned_to=${filters.assigned_to}`);
  if (filters.category) query.push(`category=${filters.category}`);

  const queryString = query.length > 0
    ? `sysparm_query=${query.join('^')}`
    : '';

  return this.makeRequest(
    `/api/now/table/incident?${queryString}&sysparm_limit=${limit}&sysparm_offset=${offset}`
  );
}

// Dual identifier resolution — accept business IDs or system IDs
async getIncident(identifier) {
  const isSysId = /^[a-f0-9]{32}$/i.test(identifier);

  if (isSysId) {
    return this.makeRequest(`/api/now/table/incident/${identifier}`);
  } else {
    return this.makeRequest(
      `/api/now/table/incident?sysparm_query=number=${identifier}&sysparm_limit=1`
    );
  }
}
```

---

## MCP Proxy Pattern (CRM Integration)

Instead of calling a REST API directly, this pattern transparently forwards MCP requests to an upstream MCP server hosted by the CRM platform.

```javascript
class MCPProxy {
  async forwardToUpstream(clientSessionId, mcpRequest) {
    const session = this.sessionManager.getUpstreamSession(clientSessionId);
    const authHeaders = await this.authClient.getAuthHeaders();

    const response = await fetch(this.upstreamMcpUrl, {
      method: 'POST',
      headers: {
        ...authHeaders,
        'mcp-session-id': session.id,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(mcpRequest)
    });

    if (response.status === 401) {
      await this.authClient.refreshToken();
      // Retry with fresh token
      return this.retryRequest(session, mcpRequest);
    }

    return await response.json();
  }
}
```

**Session lifecycle for proxy:**
1. Client connects → proxy creates local session
2. Proxy initializes upstream session with CRM platform
3. CRM returns upstream session ID
4. Proxy maps client session → upstream session
5. All subsequent requests forwarded with correct session mapping
6. Session cleanup on disconnect

---

## Response Caching Pattern

Used by EHR and pharmacy integrations to reduce API load and provide stale-data fallback on errors.

```javascript
const cache = new NodeCache({
  stdTTL: 3600,      // 1-hour default TTL
  checkperiod: 120   // Cleanup check every 2 minutes
});

async request(endpoint, params, cachePrefix) {
  const cacheKey = `v1:${cachePrefix}:${JSON.stringify(params)}`;

  // Check cache first
  const cached = cache.get(cacheKey);
  if (cached) return cached;

  try {
    const result = await this.callApi(endpoint, params);
    cache.set(cacheKey, result);
    return result;
  } catch (error) {
    // Stale cache fallback on error
    const stale = cache.get(cacheKey);
    if (stale) {
      console.warn('Returning stale cache due to API error');
      return { ...stale, _warning: 'Data may be stale' };
    }
    throw error;
  }
}
```

---

## Retry with Exponential Backoff

Used by the pharmacy/drug database integration for resilient external API calls:

```javascript
async makeRequest(queryParams = {}, attemptNumber = 1) {
  const correlationId = `mcp-${randomUUID()}`;

  try {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), this.timeout);

    const response = await fetch(url, {
      signal: controller.signal,
      headers: { 'Accept': 'application/json' }
    });

    clearTimeout(timeoutId);

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    return await response.json();
  } catch (error) {
    console.error(`[${correlationId}] Attempt ${attemptNumber}:`, error.message);

    if (attemptNumber < this.maxRetries && this.shouldRetry(error)) {
      await this.delay(this.retryDelay * attemptNumber);
      return this.makeRequest(queryParams, attemptNumber + 1);
    }
    throw error;
  }
}

shouldRetry(error) {
  if (error.name === 'AbortError') return true;             // Timeout
  if (error.message?.includes('fetch')) return true;         // Network
  if (error.message?.match(/HTTP (408|429|5\d\d)/)) return true; // Server
  return false;
}
```

---

## Response Formatting Pattern

Complex integrations (EHR, ITSM) include dedicated formatter functions that transform raw API responses into structured, readable output.

```javascript
const formatters = {
  formatPatientSearch(bundle) {
    if (!bundle?.entry?.length) return 'No patients found.';

    return bundle.entry
      .filter(e => e.resource?.resourceType === 'Patient')
      .map(e => {
        const p = e.resource;
        return [
          `Patient ID: ${p.id}`,
          `Name: ${p.name?.[0]?.text || 'N/A'}`,
          `Gender: ${p.gender || 'N/A'}`,
          `DOB: ${p.birthDate || 'N/A'}`
        ].filter(Boolean).join('\n');
      })
      .join('\n---\n');
  },

  formatConditions(bundle) { /* ... */ },
  formatMedications(bundle) { /* ... */ },
  // ... 40+ formatters for clinical resources
};
```

---

## Payload Trimming (Size-Constrained Responses)

The clinical triage integration trims responses to stay within protocol payload limits:

```javascript
function trimConversationResponse(data) {
  const trimmed = { ...data };

  // Remove fields the agent doesn't need
  delete trimmed.suggested_messages;

  // Minimize option arrays to essential fields only
  if (trimmed.next_step?.question?.options) {
    trimmed.next_step.question.options =
      trimmed.next_step.question.options.map(opt => ({
        option_id: opt.option_id,
        value: opt.value
      }));
  }

  // Remove null fields
  Object.keys(trimmed).forEach(key => {
    if (trimmed[key] === null) delete trimmed[key];
  });

  return trimmed;
}
```
