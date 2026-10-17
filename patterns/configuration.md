# Configuration Management Pattern

How MCP servers manage environment-specific configuration, secrets, and deployment settings across local development and cloud production.

---

## Configuration Module

Every server uses a centralized configuration module that loads settings from environment variables with sensible defaults.

```javascript
import dotenv from 'dotenv';
import { fileURLToPath } from 'url';
import { dirname, join } from 'path';

const __dirname = dirname(fileURLToPath(import.meta.url));
dotenv.config({ path: join(__dirname, '..', '.env') });

export const config = {
  // API Configuration
  apiBaseUrl: process.env.API_BASE_URL || 'https://api.example.com',

  // Authentication
  authMode: process.env.AUTH_MODE || 'apikey',
  apiKey: process.env.API_KEY,
  jwtToken: process.env.JWT_TOKEN,

  // Server
  serverPort: parseInt(process.env.SERVER_PORT) || 3010,
  serverPath: process.env.SERVER_PATH || '/mcp-server',

  // MCP Client Auth
  mcpApiKey: process.env.MCP_API_KEY
};
```

---

## Configuration Source Priority

For production deployments, configuration sources are loaded in priority order:

```javascript
export async function loadConfig() {
  let secrets = {};

  // 1. Cloud secrets manager (highest priority in production)
  if (process.env.AWS_SECRET_NAME && process.env.NODE_ENV === 'production') {
    const client = new SecretsManagerClient({ region: process.env.AWS_REGION });
    const command = new GetSecretValueCommand({
      SecretId: process.env.AWS_SECRET_NAME
    });
    const response = await client.send(command);
    secrets = JSON.parse(response.SecretString);
  }

  // 2. Environment variables (from .env or container injection)
  // 3. Hardcoded defaults (lowest priority)
  return {
    serverPort: parseInt(secrets.SERVER_PORT || process.env.SERVER_PORT || '3010'),
    authMode: secrets.AUTH_MODE || process.env.AUTH_MODE || 'gateway',
    apiKey: secrets.API_KEY || process.env.API_KEY,
    // ... merge all sources
  };
}
```

**Priority hierarchy:**
1. Cloud Secrets Manager (production)
2. Environment variables
3. `.env` file (via dotenv)
4. Hardcoded defaults

---

## Configuration Validation

Startup validation warns about missing credentials without crashing, enabling partial functionality.

```javascript
function validateConfig(cfg) {
  const warnings = [];
  const errors = [];

  // Required checks
  if (!cfg.apiBaseUrl) errors.push('API_BASE_URL is required');

  // Auth-mode-specific validation
  if (cfg.authMode === 'apikey' && !cfg.apiKey) {
    warnings.push('API_KEY not configured — API calls will fail');
  }
  if (cfg.authMode === 'oauth' && !cfg.gatewayPassphrase) {
    errors.push('GATEWAY_TOKEN_PASSPHRASE required for OAuth mode');
  }

  // Optional but recommended
  if (!cfg.mcpApiKey) {
    warnings.push('MCP_API_KEY not configured — MCP auth disabled');
  }

  warnings.forEach(w => console.warn(`Warning: ${w}`));
  errors.forEach(e => console.error(`Error: ${e}`));
}
```

---

## Environment Variable Categories

Across all servers, environment variables follow a consistent naming convention:

| Category | Variables | Example |
|---|---|---|
| API Endpoint | `API_BASE_URL`, `FHIR_BASE_URL` | `https://api.example.com` |
| Auth Mode | `AUTH_MODE` | `apikey`, `jwt`, `gateway`, `local`, `basic`, `oauth` |
| API Credentials | `API_KEY_*`, `JWT_TOKEN` | Platform-specific key |
| OAuth | `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, `OAUTH_TOKEN_HOST` | OAuth2 config |
| Gateway | `GATEWAY_BASE_URL`, `GATEWAY_TOKEN_PASSPHRASE` | Token broker config |
| Server | `SERVER_PORT`, `SERVER_PATH` | `3010`, `/mcp-server` |
| MCP Auth | `MCP_API_KEY` | Client-to-server auth key |
| Cloud | `AWS_REGION`, `AWS_SECRET_NAME` | `<region>`, `<secret-name>` |

---

## Docker Environment Defaults

Dockerfiles include non-sensitive defaults while expecting secrets to be injected at runtime:

```dockerfile
# Safe to include in image
ENV NODE_ENV=production
ENV SERVER_PORT=3010
ENV SERVER_PATH=/mcp-server

# Injected at runtime via orchestrator
# ENV API_KEY=...
# ENV MCP_API_KEY=...
# ENV GATEWAY_TOKEN_PASSPHRASE=...
```

---

## Key Design Decisions

1. **dotenv for local, Secrets Manager for production** — Developers use `.env` files locally; production reads from cloud secrets manager at startup.
2. **Graceful degradation** — Missing optional config logs warnings rather than crashing. Servers can start with partial functionality.
3. **Auth mode switching via single env var** — `AUTH_MODE=apikey|jwt|gateway|local|basic|oauth` selects the entire authentication strategy.
4. **No config files** — No YAML/JSON config files. Everything flows through environment variables for container-native deployment.
