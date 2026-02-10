# Agent Report 1: Repository Structure & Configuration

**Agent ID:** a63479d
**Tokens Processed:** ~47,000
**Duration:** ~121 seconds

---

# COMPREHENSIVE OPENCLAW REPOSITORY EXPLORATION - COMPLETE SUMMARY

## EXECUTIVE OVERVIEW

**OpenClaw** is a sophisticated, self-hosted, multi-channel AI gateway application written in TypeScript. It connects messaging platforms (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, and 20+ others) to AI agents (primarily Anthropic's Claude Opus 4.6). The project is structured as a monorepo with native mobile/desktop apps, a web control UI, extensive plugin system, and comprehensive CLI.

**Current Version**: 2026.2.4
**License**: MIT
**Node Requirement**: >=22.12.0
**Package Manager**: pnpm 10.23.0
**Repository**: github.com/openclaw/openclaw

---

## 1. ROOT-LEVEL DIRECTORY STRUCTURE

```
d:\Test APPs\openclaw/
|
+-- .agent/                      # Agent workflow configurations
+-- .agents/                     # Agent skills (merge-pr, prepare-pr, review-pr)
+-- .github/                     # GitHub CI/CD and issue management
|   +-- ISSUE_TEMPLATE/
|   +-- workflows/              # CI/CD pipeline files
|
+-- .pi/                         # Pi agent extensions, git hooks, prompts
+-- .vscode/                     # VS Code workspace settings
+-- .swiftformat                 # Swift code formatting config
+-- .oxlintrc.json              # Oxlint (Rust-based linter) config
|
+-- apps/                        # Platform-specific native apps
|   +-- android/                # Android app (Kotlin/Gradle)
|   +-- ios/                    # iOS app (Swift/Xcode)
|   +-- macos/                  # macOS app (Swift)
|   +-- shared/                 # OpenClawKit shared Swift framework
|
+-- assets/                      # Chrome extension & static assets
+-- docs/                        # Comprehensive markdown documentation
+-- extensions/                  # 31 plugin channels for integration
|
+-- patches/                     # pnpm patches for dependencies
+-- scripts/                     # Build, deploy, and utility scripts
+-- skills/                      # Runtime skills directory
|
+-- src/                         # Main TypeScript source code (43 subdirs)
+-- ui/                          # Web control UI (Lit-based)
+-- vendor/                      # Vendored dependencies (a2ui, swagger)
|
+-- Swabble/                     # Separate Swift project (submodule-like)
|
+-- Configuration Files
|   +-- package.json             # Root monorepo package
|   +-- pnpm-workspace.yaml      # Workspace configuration
|   +-- tsconfig.json            # TypeScript compiler options
|   +-- Dockerfile               # Production Docker image
|   +-- docker-compose.yml       # Docker Compose services
|   +-- fly.toml                 # Fly.io deployment config
|   +-- [vitest configs]         # 8 test configuration variants
|
+-- README.md                    # Main documentation (103KB)
+-- CHANGELOG.md                 # Version history
+-- LICENSE                      # MIT license
+-- openclaw.mjs                 # CLI entry point script
+-- [miscellaneous files]        # env examples, git config, etc.
```

---

## 2. PACKAGE.JSON FILES - DETAILED CONTENTS

### 2.1 ROOT PACKAGE.JSON

**Metadata**:
- `name`: openclaw
- `version`: 2026.2.4
- `description`: WhatsApp gateway CLI (Baileys web) with Pi RPC agent
- `license`: MIT
- `type`: module (ES modules throughout)

**Entry Points**:
```json
{
  "bin": { "openclaw": "openclaw.mjs" },
  "main": "dist/index.js",
  "exports": {
    ".": "./dist/index.js",
    "./plugin-sdk": "./dist/plugin-sdk/index.js",
    "./cli-entry": "./openclaw.mjs"
  }
}
```

**ALL 103 NPM SCRIPTS** (organized by category):

*Android*:
- `android:assemble` - Build debug APK
- `android:install` - Install on device
- `android:run` - Install and run
- `android:test` - Unit tests

*Build & Compilation*:
- `build` - Main: bundle a2ui + tsdown + copy assets + hooks + build info
- `canvas:a2ui:bundle` - Bundle A2UI canvas system
- `prepack` - Pre-publish: build + ui:build

*Code Quality*:
- `check` - Full check: tsgo + lint + format
- `check:docs` - Docs: format + lint + build
- `check:loc` - Lines of code check (max 500)
- `lint` - Oxlint type-aware
- `lint:all` - All linting (JS + Swift)
- `lint:docs` - Markdown linting
- `lint:docs:fix` - Auto-fix docs
- `lint:fix` - Oxlint + format fix
- `lint:swift` - SwiftLint
- `format` - Oxfmt check
- `format:all` - Format JS + Swift
- `format:docs` - Docs format check
- `format:docs:fix` - Auto-fix docs format
- `format:fix` - Auto-fix all formatting
- `format:swift` - SwiftFormat

*Development*:
- `dev` - Run node dev server
- `gateway:dev` - Gateway development (skip channels)
- `gateway:dev:reset` - Gateway dev with reset
- `gateway:watch` - Watch mode with auto-reload
- `tui:dev` - Terminal UI dev mode
- `ui:dev` - Web UI dev server

*Documentation*:
- `docs:bin` - Build docs list
- `docs:build` - Build docs site (Mint)
- `docs:dev` - Docs dev server
- `docs:list` - List docs

*iOS*:
- `ios:build` - Build iOS debug
- `ios:gen` - Generate Xcode project
- `ios:open` - Open Xcode project
- `ios:run` - Build and run simulator

*macOS*:
- `mac:open` - Open .app in Finder
- `mac:package` - Package .app distribution
- `mac:restart` - Restart daemon

*Protocol Generation*:
- `protocol:check` - Verify protocol schema
- `protocol:gen` - Generate JS protocol
- `protocol:gen:swift` - Generate Swift protocol

*Plugins & Compatibility*:
- `moltbot:rpc` - Moltbot RPC mode
- `openclaw` - Run CLI from source
- `openclaw:rpc` - RPC mode
- `plugins:sync` - Sync plugin versions

*Release*:
- `release:check` - Pre-release validation

*Server*:
- `start` - Run server

*Testing* (comprehensive):
- `test` - Parallel unit tests
- `test:all` - Full pipeline (lint + build + unit + e2e + live + docker)
- `test:coverage` - Coverage report
- `test:docker:all` - All docker tests
- `test:docker:cleanup` - Clean docker resources
- `test:docker:doctor-switch` - Doctor command docker test
- `test:docker:gateway-network` - Network docker test
- `test:docker:live-gateway` - Live gateway models docker test
- `test:docker:live-models` - Live models docker test
- `test:docker:onboard` - Onboarding docker test
- `test:docker:plugins` - Plugins docker test
- `test:docker:qr` - QR import docker test
- `test:e2e` - End-to-end tests
- `test:force` - Force test run
- `test:install:e2e` - Installation e2e
- `test:install:e2e:anthropic` - Installation with Anthropic
- `test:install:e2e:openai` - Installation with OpenAI
- `test:install:smoke` - Smoke test
- `test:live` - Live API tests
- `test:ui` - UI tests
- `test:watch` - Watch mode

*Other*:
- `prepare` - Git hooks setup
- `tui` - Terminal UI

**Production Dependencies** (155 total - key categories):

*AI/LLM Core*:
- `@mariozechner/pi-agent-core` (0.52.6)
- `@mariozechner/pi-ai` (0.52.6)
- `@mariozechner/pi-coding-agent` (0.52.6)
- `@mariozechner/pi-tui` (0.52.6)

*Messaging Platforms*:
- `@slack/bolt` (^4.6.0)
- `@line/bot-sdk` (^10.6.0)
- `grammy` (^1.39.3)
- `@whiskeysockets/baileys` (7.0.0-rc.9)
- `discord-api-types` (^0.38.38)

*Cloud/Infrastructure*:
- `@aws-sdk/client-bedrock` (^3.984.0)
- `express` (^5.2.1)
- `hono` (4.11.7)
- `ws` (^8.19.0)

*Media & Processing*:
- `sharp` (^0.34.5)
- `pdfjs-dist` (^5.4.624)
- `playwright-core` (1.58.1)

*Terminal & CLI*:
- `commander` (^14.0.3)
- `@clack/prompts` (^1.0.0)
- `chalk` (^5.6.2)

*Data & Utilities*:
- `zod` (^4.3.6)
- `sqlite-vec` (0.1.7-alpha.2)
- `croner` (^10.0.1)

**Engine Requirements**:
- Node: >=22.12.0
- Package Manager: pnpm 10.23.0

### 2.2 WORKSPACE PACKAGES

**UI Package** (`ui/package.json`):
- name: openclaw-control-ui
- Dependencies: lit, marked, dompurify, @noble/ed25519
- DevDependencies: vite 7.3.1, vitest, @vitest/coverage-v8

**Extension Packages** (31 total) - each follows:
```json
{
  "name": "@openclaw/<channel-name>",
  "version": "2026.2.4",
  "type": "module",
  "private": true,
  "openclaw": {
    "extensions": ["./index.ts"]
  }
}
```

---

## 3. CONFIGURATION FILES

### 3.1 TypeScript Configuration
- Target: ES2023
- Module: NodeNext
- Strict: true
- Declaration: true
- experimentalDecorators: true (for UI framework)
- Includes: src/**/* and ui/**/*

### 3.2 pnpm Workspace
- Packages: root, ui, packages/*, extensions/*
- onlyBuiltDependencies: baileys, node-pty, matrix-sdk-crypto, sharp, canvas, esbuild, protobufjs

### 3.3 Docker Configuration
- Base: node:22-bookworm
- Security: Non-root user (uid 1000)
- Ports: 18789 (gateway), 18790 (bridge)
- Default CMD: gateway --allow-unconfigured

### 3.4 Fly.io Deployment
- Region: iad (US East)
- VM: shared-cpu-2x, 2048MB
- Persistent volume: openclaw_data at /data
- Auto-start: true, Min machines: 1

### 3.5 Vitest Configuration
- Pool: forks (process isolation)
- Workers: 3 in CI, up to 16 locally
- Timeout: 120s (180s on Windows)
- Coverage: 70% lines/functions/statements, 55% branches

---

## 4. SOURCE CODE DIRECTORY STRUCTURE (43 directories)

```
src/
+-- Core Systems: agents/, gateway/, channels/, cli/, commands/, config/, plugins/, plugin-sdk/, types/
+-- Channel Implementations: discord/, slack/, telegram/, signal/, line/, imessage/, whatsapp/, web/, webchat/
+-- Agent Capabilities: browser/, canvas-host/, tui/, terminal/, tts/, acp/, macos/, media-understanding/
+-- Infrastructure: daemon/, logging/, security/, sessions/, process/, infra/, providers/, node-host/
+-- Features: memory/, wizard/, cron/, routing/, hooks/, pairing/, auto-reply/, link-understanding/, markdown/, media/
+-- Support: test-helpers/, test-utils/, utils/, shared/, scripts/, docs/
```

---

## 5. EXTENSION SYSTEM (31 Plugins)

**Channel Integrations (18)**: whatsapp, telegram, slack, discord, signal, line, imessage, googlechat, msteams, mattermost, matrix, nextcloud-talk, bluebubbles, twitch, nostr, tlon, feishu, zalo/zalouser

**Voice & Media**: voice-call

**Memory & Storage**: memory-core, memory-lancedb

**Authentication**: google-antigravity-auth, google-gemini-cli-auth, minimax-portal-auth, qwen-portal-auth

**Utilities**: llm-task, diagnostics-otel, open-prose, copilot-proxy, lobster

---

## 6. KEY PROJECT CHARACTERISTICS

**Design Principles**:
1. Self-Hosted First
2. Multi-Channel (single gateway for many platforms)
3. Plugin Architecture
4. Local Gateway (central control plane)
5. Type Safety (strict TypeScript)
6. Test Coverage (70% minimum)
7. Security (pairing flows, allowlists, non-root execution)

**Repository Statistics**:
- Source Files: 40+ directories with 100s of TypeScript files
- Extensions: 31 channel/service plugins
- Tests: 8 different test configurations
- Scripts: 20+ build and deployment scripts
- Documentation: 50+ markdown files
- NPM Scripts: 103 total commands
- Dependencies: 155 production, 21 development
- Monorepo Packages: 37 (root + ui + packages + extensions)
