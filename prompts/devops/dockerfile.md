---
title: Production-Ready Dockerfile Generator
category: devops
version: 1.0.0
works_with: [claude, openai, ollama, gemini]
---

## Purpose
Generate a production-ready, security-hardened, multi-stage Dockerfile for any application — not the 3-line toy version you find in tutorials.

## When to Use
- Starting a new service and need a proper Dockerfile from day one
- Your existing Dockerfile produces oversized images or has security vulnerabilities
- Preparing a service for a cloud deployment that requires best-practice container configs

## The Prompt

```
You are a DevOps engineer and container security expert. Generate a production-ready Dockerfile for the following application.

**Application type:** {{APP_TYPE}}
(Examples: "Node.js Express API", "Python FastAPI", "Go HTTP server", "Next.js SSR app")

**Runtime version:** {{RUNTIME_VERSION}}
(Examples: "Node 20", "Python 3.12", "Go 1.22")

**Build requirements:**
{{BUILD_REQUIREMENTS}}
(Examples: "runs `npm run build` to compile TypeScript", "requires `poetry install --no-dev`", "uses CGO disabled for static binary")

**Environment variables needed at runtime:**
{{ENV_VARS}}
(Examples: "DATABASE_URL, JWT_SECRET, PORT", "none — all env is injected at deploy time")

**Port the app listens on:** {{PORT}}

**Special requirements:**
{{SPECIAL_REQUIREMENTS}}
(Examples: "needs ffmpeg installed", "must run as non-root", "image must be under 100MB", "none")

Generate:
1. **The Dockerfile** — Multi-stage build. Production stage must:
   - Use a minimal base image (distroless, alpine, or slim variant)
   - Run as a non-root user
   - Have no development dependencies in the final image
   - Include a proper HEALTHCHECK instruction
   - Use explicit image digest tags, not `latest`
   - Set appropriate labels (maintainer, version, description)

2. **.dockerignore** — A complete .dockerignore file for this project type

3. **Build & run commands** — The exact `docker build` and `docker run` commands to test locally

4. **Security notes** — Any potential vulnerabilities in this setup and how to address them

5. **Image size estimate** — Approximate final image size
```

## Example Input

| Variable | Value |
|---|---|
| `{{APP_TYPE}}` | `Node.js Express API` |
| `{{RUNTIME_VERSION}}` | `Node 20` |
| `{{BUILD_REQUIREMENTS}}` | `runs npm run build to compile TypeScript` |
| `{{ENV_VARS}}` | `DATABASE_URL, JWT_SECRET, PORT` |
| `{{PORT}}` | `3000` |
| `{{SPECIAL_REQUIREMENTS}}` | `Must run as non-root` |

## Example Output

```dockerfile
# ==================== BUILD STAGE ====================
FROM node:20.14.0-alpine3.20 AS builder
LABEL stage=builder

WORKDIR /app

# Copy dependency manifests first for layer caching
COPY package*.json ./
RUN npm ci --include=dev

# Copy source and build
COPY tsconfig.json ./
COPY src/ ./src/
RUN npm run build

# ==================== PRODUCTION STAGE ====================
FROM node:20.14.0-alpine3.20 AS production

LABEL maintainer="your-team@example.com" \
      version="1.0.0" \
      description="Express API service"

# Create non-root user
RUN addgroup -g 1001 -S nodegroup && \
    adduser -u 1001 -S nodeuser -G nodegroup

WORKDIR /app

# Install only production deps
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force

# Copy compiled output from builder
COPY --from=builder /app/dist ./dist

# Set file ownership
RUN chown -R nodeuser:nodegroup /app

USER nodeuser

ENV NODE_ENV=production \
    PORT=3000

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

**.dockerignore**
```
node_modules
dist
.git
.env*
*.log
coverage
.nyc_output
README.md
```

**Build & Run**
```bash
docker build -t my-api:latest .
docker run -p 3000:3000 \
  -e DATABASE_URL=postgres://... \
  -e JWT_SECRET=changeme \
  my-api:latest
```

**Security Notes**
- All deps pinned to exact versions via package-lock.json ✅
- No shell in production image (attack surface reduced) — use `exec` form CMD ✅
- Secrets are NOT baked into the image — injected at runtime via env ✅
- Consider: add `--read-only` flag to `docker run` for immutable filesystem

**Estimated Image Size:** ~180MB
```

## Tips
- Always specify `{{RUNTIME_VERSION}}` with a patch version (e.g., `Node 20.14`) — `latest` in production is an incident waiting to happen
- If your app needs secrets at build time (e.g., private npm registry), mention it — the solution involves Docker BuildKit secrets, not ENV
- For the smallest possible images, ask for "distroless" as a special requirement — useful for Go and Java apps
