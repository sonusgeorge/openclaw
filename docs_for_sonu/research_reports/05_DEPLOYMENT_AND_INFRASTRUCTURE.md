# Agent Report 5: Deployment & Infrastructure

**Agent ID:** a60af0b
**Tokens Processed:** ~58,000
**Duration:** ~119 seconds

---

# OPENCLAW DEPLOYMENT & INFRASTRUCTURE - COMPLETE TECHNICAL SUMMARY

## EXECUTIVE SUMMARY

OpenClaw uses a pnpm monorepo with comprehensive CI/CD automation, Docker containerization, and multi-platform build infrastructure. Deployment targets include Docker, Fly.io, and Render.com.

---

## 1. DOCKER CONFIGURATION

### 1.1 Production Dockerfile

- **Base:** Node.js 22 on Debian Bookworm
- **Build Tool:** Bun (installed via curl)
- **Package Manager:** pnpm 10.23.0 (via corepack)
- **Security:** Non-root user (UID 1000)
- **Ports:** 18789 (gateway), 18790 (bridge)
- **Build Args:** `OPENCLAW_DOCKER_APT_PACKAGES` for additional packages
- **Default CMD:** `node dist/index.js gateway --allow-unconfigured`

### 1.2 docker-compose.yml

**Services:**
1. **openclaw-gateway** - Daemon service with auto-restart
2. **openclaw-cli** - Interactive CLI with TTY

**Environment Variables:**
- `OPENCLAW_GATEWAY_TOKEN` - Authentication
- `CLAUDE_AI_SESSION_KEY` - Claude AI session
- `CLAUDE_WEB_SESSION_KEY` - Claude web session
- `OPENCLAW_GATEWAY_BIND` - Binding address (default: lan)

**Volumes:**
- `~/.openclaw/` - Configuration directory
- `~/.openclaw/workspace/` - Workspace directory

### 1.3 Sandbox Dockerfiles

**Dockerfile.sandbox** - Minimal testing sandbox (bash, curl, git, jq, python3, ripgrep)

**Dockerfile.sandbox-browser** - Browser automation sandbox with:
- Chromium
- X11VNC + NoVNC
- Ports: 9222 (CDP), 5900 (VNC), 6080 (NoVNC web)

### 1.4 Test Dockerfiles

- **scripts/e2e/Dockerfile** - Full build E2E testing
- **scripts/docker/install-sh-smoke/Dockerfile** - Root user install test
- **scripts/docker/install-sh-nonroot/Dockerfile** - Non-root install test (Ubuntu 24.04 with retry logic)

---

## 2. CI/CD CONFIGURATION (GitHub Actions)

### 2.1 Main CI Workflow (ci.yml - 642 lines)

**Jobs:**

1. **install-check** - pnpm install --frozen-lockfile validation

2. **checks** - Multi-matrix build & test:
   - TypeScript verification (pnpm tsgo)
   - Build & Lint (pnpm build && pnpm lint)
   - Tests (pnpm test with vitest)
   - Protocol check (pnpm protocol:check)
   - Format check (pnpm format)
   - Bun tests (bunx vitest run)

3. **secrets** - Secret scanning with detect-secrets v1.5.0

4. **checks-windows** - Windows-specific (4 vCPU, 4GB RAM, single-threaded tests)

5. **checks-macos** - macOS testing (PR-only)

6. **macos-app** - Swift validation (lint, format, xcode-build, 43% coverage minimum)

7. **ios** - iOS testing (CURRENTLY DISABLED: `if: false`)

8. **android** - Android (OpenJDK 21, SDK Level 36, Gradle tests)

### 2.2 Docker Release Workflow (docker-release.yml - 144 lines)

**Triggers:** Push to main, tags v*, workflow_dispatch

**Jobs:**
1. build-amd64 (ubuntu-latest)
2. build-arm64 (ubuntu-24.04-arm)
3. create-manifest (multi-platform)

**Registry:** GitHub Container Registry (GHCR)
**Tags:** Branch-based, version-based, latest

### 2.3 Other Workflows

- **install-smoke.yml** - Installation smoke tests
- **formal-conformance.yml** - External formal model checks (informational)
- **workflow-sanity.yml** - YAML tab detection
- **labeler.yml** - Auto PR labeling
- **auto-response.yml** - Issue auto-response and routing

---

## 3. ENVIRONMENT VARIABLES

### Production
```
PORT=8080
NODE_ENV=production
OPENCLAW_GATEWAY_TOKEN=<generated>
OPENCLAW_GATEWAY_BIND=lan
OPENCLAW_STATE_DIR=/data/.openclaw
OPENCLAW_WORKSPACE_DIR=/data/workspace
```

### Authentication
```
CLAUDE_AI_SESSION_KEY=
CLAUDE_WEB_SESSION_KEY=
CLAUDE_WEB_COOKIE=
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_WHATSAPP_FROM=
```

### CI
```
NODE_OPTIONS=--max-old-space-size=4096
CLAWDBOT_TEST_WORKERS=1 (Windows)
CI=true
```

---

## 4. DEPLOYMENT CONFIGURATIONS

### 4.1 Render.yaml

```yaml
services:
  - type: web
    name: openclaw
    runtime: docker
    plan: starter
    healthCheckPath: /health
    disk:
      name: openclaw-data
      mountPath: /data
      sizeGB: 1
```

Auto-generates: `OPENCLAW_GATEWAY_TOKEN`
Manual: `SETUP_PASSWORD`

### 4.2 Fly.io (fly.toml)

```toml
app = "openclaw"
primary_region = "iad"
[env]
NODE_ENV = "production"
NODE_OPTIONS = "--max-old-space-size=1536"
[processes]
app = "node dist/index.js gateway --allow-unconfigured --port 3000 --bind lan"
[[vm]]
size = "shared-cpu-2x"
memory = "2048mb"
[mounts]
source = "openclaw_data"
destination = "/data"
```

---

## 5. TESTING CONFIGURATION

### Vitest Config Variants (8 total)

| Config | Scope | Workers |
|--------|-------|---------|
| vitest.config.ts | Main (all unit) | 3 CI, up to 16 local |
| vitest.unit.config.ts | Non-gateway unit | Same as main |
| vitest.e2e.config.ts | E2E tests | 2 CI, 1-4 local |
| vitest.live.config.ts | Live API tests | 1 (sequential) |
| vitest.gateway.config.ts | Gateway protocol | Same as main |
| vitest.extensions.config.ts | Extension tests | Same as main |
| ui/vitest.config.ts | UI browser tests | Playwright/Chromium |
| ui/vitest.node.config.ts | UI node tests | Standard |

### Coverage Thresholds
- Lines: 70%
- Functions: 70%
- Statements: 70%
- Branches: 55%

### Test Categories
- **Unit**: Local, deterministic, mocked externals
- **E2E**: Full system flow, slower
- **Live**: Real API integration (OPENCLAW_LIVE_TEST=1)
- **Docker**: Full environment simulation

---

## 6. BUILD SYSTEM

### Build Process
1. Bundle A2UI specification (`pnpm canvas:a2ui:bundle`)
2. Transpile TypeScript -> JavaScript (`tsdown`)
3. Generate source maps & minification
4. Copy hooks metadata
5. Generate build info file

### Platform Build Scripts

**macOS:**
- build-and-run-mac.sh
- package-mac-app.sh
- codesign-mac-app.sh
- notarize-mac-artifact.sh
- create-dmg.sh
- make_appcast.sh (Sparkle update feed)

**Mobile:**
- sandbox-setup.sh
- sandbox-browser-setup.sh
- mobile-reauth.sh

### Quality Gates
- TypeScript type checking: `pnpm tsgo`
- Linting: `oxlint` with type awareness
- Formatting: `oxfmt`
- Protocol validation: Generated code matches checked-in version
- Lines of code: Max 500 per file

---

## 7. MONOREPO STRUCTURE

### pnpm-workspace.yaml
```yaml
packages:
  - .              # Root (CLI + gateway)
  - ui             # Control UI
  - packages/*     # Compatibility shims
  - extensions/*   # 31+ extension packages
```

### onlyBuiltDependencies (native binaries)
- @whiskeysockets/baileys
- @lydell/node-pty
- @matrix-org/matrix-sdk-crypto-nodejs
- authenticate-pam
- esbuild
- protobufjs
- sharp
- @napi-rs/canvas

---

## 8. SECURITY

### CI Security
- **detect-secrets** scans for API keys, passwords, tokens
- Baseline file prevents false positives

### Docker Security
- Non-root user (UID 1000)
- .dockerignore excludes sensitive files
- Lean images (~500MB-1GB)

### macOS Security
- Code signing
- Apple notarization
- SwiftLint + SwiftFormat enforcement

---

## 9. RELEASE WORKFLOW

### Release Check
```bash
pnpm release:check
# Runs: build + test:unit + protocol:check
```

### Version Scheme
Calendar versioning: 2026.2.4 (YYYY.M.D)

### Distribution
- npm publish (prepack runs build)
- Docker images (GHCR, multi-arch)
- macOS DMG (code-signed, notarized)
- Update channels: stable, beta, dev

---

## SUMMARY TABLE

| Component | Type | Location |
|-----------|------|----------|
| Docker Production | Dockerfile | root |
| Docker Compose | docker-compose.yml | root |
| Docker Sandbox | Dockerfile.sandbox | root |
| Docker Browser | Dockerfile.sandbox-browser | root |
| CI Main | .github/workflows/ci.yml | .github |
| Docker Release | docker-release.yml | .github |
| Render Deploy | render.yaml | root |
| Fly.io Deploy | fly.toml | root |
| Test Main | vitest.config.ts | root |
| Test E2E | vitest.e2e.config.ts | root |
| Test Live | vitest.live.config.ts | root |
| Test UI | ui/vitest.config.ts | ui/ |
| Build Scripts | scripts/ | scripts/ |
| pnpm Workspace | pnpm-workspace.yaml | root |
