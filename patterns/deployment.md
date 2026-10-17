# Deployment Pattern

Containerization and cloud deployment architecture for MCP servers.

---

## Multi-Stage Docker Build

Every MCP server uses an identical Dockerfile pattern optimized for minimal image size and production security.

```dockerfile
# Stage 1: Builder — install ALL dependencies
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci

# Stage 2: Production — only production dependencies
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force
COPY --from=builder /app/node_modules ./node_modules
COPY src ./src

# Non-sensitive defaults
ENV NODE_ENV=production
ENV SERVER_PORT=3010
ENV SERVER_PATH=/mcp-server

EXPOSE 3010

# Container health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider \
  http://localhost:3010/mcp-server/health || exit 1

# Run the HTTP transport (production)
CMD ["node", "src/server-http.js"]
```

**Key optimizations:**
- Alpine Linux base (~18 MB) for minimal attack surface
- Multi-stage build eliminates dev dependencies from final image
- `npm cache clean --force` removes build artifacts
- Health check enables orchestrator monitoring
- Streamable HTTP transport selected for production (not stdio)

---

## CI/CD Pipeline (GitHub Actions)

Automated build and deploy pipeline triggered on push to main branch.

```yaml
name: Deploy to Cloud
on:
  push:
    branches: [main, master]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure Cloud Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Login to Container Registry
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and Push Docker Image
        run: |
          docker build --platform linux/amd64 \
            -t $REGISTRY/$IMAGE:${{ github.sha }} \
            -t $REGISTRY/$IMAGE:latest .
          docker push $REGISTRY/$IMAGE --all-tags

      - name: Download Current Task Definition
        run: |
          aws ecs describe-task-definition \
            --task-definition $TASK_FAMILY \
            --query taskDefinition > task-definition.json

      - name: Inject Secrets from Secrets Manager
        run: |
          jq '.containerDefinitions[0].secrets += [
            {"name": "MCP_API_KEY", "valueFrom": "${{ secrets.SECRET_ARN }}"}
          ]' task-definition.json > updated-task-def.json

      - name: Update Task Definition with New Image
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: updated-task-def.json
          container-name: mcp-server
          image: ${{ env.REGISTRY }}/${{ env.IMAGE }}:${{ github.sha }}

      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.render.outputs.task-definition }}
          service: mcp-server-service
          cluster: mcp-cluster
          wait-for-service-stability: true
```

---

## ECS Fargate Task Definition

Container task configuration for serverless container deployment:

```json
{
  "family": "mcp-server-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "mcp-server",
      "image": "registry/mcp-server:latest",
      "portMappings": [
        { "containerPort": 3010, "protocol": "tcp" }
      ],
      "environment": [
        { "name": "NODE_ENV", "value": "production" },
        { "name": "SERVER_PORT", "value": "3010" }
      ],
      "secrets": [
        { "name": "API_KEY", "valueFrom": "arn:aws:secretsmanager:..." },
        { "name": "MCP_API_KEY", "valueFrom": "arn:aws:secretsmanager:..." }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "wget -q --spider http://localhost:3010/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 10
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/mcp-server",
          "awslogs-region": "<region>",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

---

## Deployment Architecture

```
                    Internet
                       │
                       ▼
              ┌────────────────┐
              │  Load Balancer │
              │    (ALB)       │
              └───────┬────────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
   ┌───────────┐┌───────────┐┌───────────┐
   │ MCP Task 1││ MCP Task 2││ MCP Task N│
   │ (Fargate) ││ (Fargate) ││ (Fargate) │
   └─────┬─────┘└─────┬─────┘└─────┬─────┘
         │            │            │
         └────────────┼────────────┘
                      │
              ┌───────┴────────┐
              │ Secrets Manager│
              │ (API Keys,     │
              │  Tokens, etc.) │
              └───────┬────────┘
                      │
              ┌───────┴────────┐
              │  CloudWatch    │
              │  (Logging)     │
              └────────────────┘
```

---

## Port Allocation Convention

Each MCP server runs on a unique port to enable co-located deployment:

| Server | Port | Path |
|---|---|---|
| Retail Integration | 3010 | /retail |
| Healthcare Integration | 3011 | /healthcare |
| Insurance Integration | 3012 | /insurance |
| ITSM Integration | 3013 | /itsm |
| EHR Integration (Patient) | 3014 | /ehr |
| EHR Integration (Clinical) | 3010 | /ehr-clinical |
| Clinical Triage | 3016 | /triage |
| Pharmacy DB | 3017 | /pharmacy |
| CRM Proxy | 2018 | /crm-proxy |
| Lab Pod Management | 1013 | /pods |

---

## Key Design Decisions

1. **Fargate over EC2** — Serverless containers eliminate server management overhead. CPU/memory sized per server workload.
2. **ALB per server** — Each MCP server gets its own load balancer target group, enabling independent scaling.
3. **Secrets Manager over env files** — Secrets injected at container runtime, never baked into images.
4. **CloudWatch logging** — Centralized log aggregation with structured JSON logging.
5. **GitHub Actions** — CI/CD integrated directly with source control. Push to main triggers full deploy pipeline.
