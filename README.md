# Helio — AI Image Editing Platform (Enterprise)

Helio is a production-ready, extensible web application for AI-powered image editing and refinement. Built with Next.js (App Router), React, TypeScript, Tailwind CSS and the Fal AI client, Helio provides a secure, local-first UX backed by IndexedDB for conversation and asset persistence and a server-side API for interacting with model provider endpoints.

This README provides an executive summary, architecture overview, installation and deployment instructions, operational guidance, security considerations, API contract, data model, troubleshooting, and contribution guidance for enterprise teams.

---

## Table of contents

- Executive summary
- Project status
- Architecture overview
- Quickstart (developer)
- Production deployment (recommended)
- Configuration & secrets
- API reference
- Data model
- Observability & diagnostics
- Security & compliance
- Testing & CI
- Release & branching strategy
- Contributing
- Appendix: file map

## Executive summary

Helio exposes a web interface to upload or attach images and refine them using one or more AI image-editing models. The application is intentionally modular so teams can swap model providers, customize UX flows, and integrate enterprise features like RBAC, audit logging, and deterministic reproducibility.

Key features

- Single-page image editing UX with conversation history and generated image versions
- Local persistence using IndexedDB for offline-first behavior and immediate UX responsiveness
- Server-side API proxy to the Fal AI model endpoints (`@fal-ai/client`) to keep credentials server-side
- Pluggable model registry (`lib/model-endpoints.json`) with model-specific parameters
- Minimal, secure defaults suitable for private deployments

## Project status

- Framework: Next.js (App Router), React 19, TypeScript
- CI: none configured by default — recommended to add GitHub Actions or an enterprise CI pipeline
- Tested: basic smoke-tested locally; recommend adding unit/integration tests and E2E coverage
- License: see `LICENSE`

## Architecture overview

High-level components

- Next.js App (frontend + serverless API routes)
  - UI: `app/*`, `components/*` — React + Tailwind components and providers
  - API: `app/api/generate-image/route.ts` — server-side proxy to Fal AI
- Client-side persistence: `lib/indexeddb.ts` — conversation and generated images store
- Client hooks & API: `lib/queries.ts` — react-query hooks that encapsulate local persistence and server interactions
- Model registry: `lib/model-endpoints.json` — human-editable mapping of friendly model names to provider endpoints

Operational flow

1. User uploads or selects an image and enters a textual prompt.
2. Client saves conversation state to IndexedDB and issues a POST to `/api/generate-image` with the user's API key and prompt.
3. Server-side route validates inputs, configures the Fal AI client, and invokes the selected model endpoint.
4. The server returns the generated image URL; the client updates the conversation and UI.

Security model

- By default the app stores a per-user `falKey` in `localStorage` for convenience during development. For production, move credential management to the server and require authenticated access to server APIs. Use vault-backed secrets and short-lived tokens.

## Quickstart (developer)

Prerequisites

- Node.js 20+ (LTS recommended)
- pnpm (preferred, project includes `pnpm-lock.yaml`) or npm/yarn

Local install & run

```bash
cd /workspaces/helio
pnpm install
pnpm dev
```

Open http://localhost:3000. The app uses an in-browser local database (IndexedDB) so you can iterate without external services. To enable image generation, obtain a Fal AI API key and enter it in Settings in the UI.

Quick test

1. Upload an image.
2. Enter a prompt and choose a model.
3. Click Generate and verify the image appears and that `/api/generate-image` logs the request server-side.

## Production deployment (recommended)

This project is optimized for serverless deployments (Vercel, Netlify, Cloud Run). Recommended steps for enterprise deployment:

1. Use a managed hosting platform (Vercel or your preferred cloud) with separate environments for staging and production.
2. Configure environment secrets in the platform or in your secret manager (HashiCorp Vault, Azure Key Vault, AWS Secrets Manager).
3. Replace client-side `falKey` storage with server-side per-user credential management. Implement authentication (OIDC/SAML) and an API that issues short-lived credentials.
4. Harden the deployment with TLS, HSTS, secure headers and WAF rules; restrict model provider endpoints to known IPs or VPC egress where supported.

Example (Vercel)

- Do NOT expose Fal AI credentials via `NEXT_PUBLIC_...` environment variables.
- Add secrets in Vercel's dashboard and use server-side code to retrieve them.

## Configuration & secrets

Local configuration

- `localStorage` keys used by the app: `falKey`, `selectedModel` (managed via `lib/queries.ts`).

Server configuration

- The project does not require environment variables by default, but for production you should configure:
  - Vault or secret manager access for Fal AI keys
  - Monitoring/metrics endpoint configuration
  - Optional feature flags

Secret handling guidance

- Never commit API credentials to the repository.
- Use least-privilege credentials and short-lived tokens where possible.

## API reference

### POST /api/generate-image

Server-side proxy that validates input and calls Fal AI via `@fal-ai/client`.

Request body (application/json)

- `falKey` (string) — Fal AI API key (must be kept secret)
- `prompt` (string) — textual instructions for the image edit
- `imageUrl` (string) — URL or base64 data URL of the source image
- `model` (string) — friendly model name (from `lib/model-endpoints.json` or built-in mappings)

Responses

- 200 OK: `{ imageUrl: string, model: string }`
- 400 Bad Request: `{ error: string }` (missing/invalid fields)
- 500 Internal Server Error: `{ error: string }` (provider or internal error)

Notes

- The server selects model-specific parameters (strength, steps, guidance_scale) based on friendly model name and calls the Fal AI `subscribe` API. Adapt for streaming/long-running responses as needed.

## Data model

Primary types (see `lib/indexeddb.ts`)

- `Conversation` — { id, title, messages[], generatedImages[], createdAt, updatedAt }
- `ChatMessage` — { id, type: 'user'|'assistant', content, image?, generatedImage?, timestamp }
- `GeneratedImage` — { id, url, prompt, timestamp, model? }

Persistence

- Client persists conversations and generated images in IndexedDB under the database `ImageEditorDB` using `lib/indexeddb.ts`.
- For multi-user deployments, replace local persistence with a server-side store (Postgres, DynamoDB, etc.) and implement sync/merge strategies and access control.

## Observability & diagnostics

Logging

- Server logs: `/app/api/generate-image/route.ts` logs FAL request metadata and provider responses.
- Client logs: debug logs are prefixed with `[v0]` for easier grep during development.

Metrics

- Instrument `/api/generate-image` with request latency, error rate, and provider call counts.
- Tag metrics by tenant/team/model for cost tracking.

Tracing

- Add OpenTelemetry tracing in serverless functions to correlate UI actions with provider calls.

## Security & compliance

- Do not store production Fal AI keys in client-side storage. Use secure vaults.
- Sanitize and validate user prompts before persisting to server-side stores.
- Enforce RBAC and auditing for production usage.
- Apply standard web security headers and a strict CSP.

## Testing & CI

Recommendations

- Add unit tests for `lib/indexeddb.ts` and `lib/queries.ts` using Vitest or Jest.
- Add E2E tests with Playwright to validate upload -> generate -> display flows.
- Add a GitHub Actions workflow for lint, typecheck, build, and test.

## Release & branching strategy

- Trunk-based development with short-lived feature branches is recommended.
- Use semantic versioning and tag releases from `main`.
- Protect `main` with required PR reviews and status checks.

## Contributing

1. Fork the repository and create a feature branch.
2. Run `pnpm install` and `pnpm dev` locally.
3. Add tests for new behavior and open a PR against `main`.

## Appendix: file map

- `app/` — Next.js application routes and pages (App Router)
  - `app/page.tsx` — main image editor UI
  - `app/layout.tsx` — root layout and metadata
  - `app/api/generate-image/route.ts` — server API proxy to Fal AI
- `components/` — UI primitives and providers
- `lib/` — client hooks (`queries.ts`), model registry, and IndexedDB (`indexeddb.ts`)
- `public/` — static assets
- `next.config.mjs`, `postcss.config.mjs`, `tailwind` config
- `package.json`, `pnpm-lock.yaml`, `tsconfig.json`

---

Contact & support

For enterprise support, operate from a private fork and configure CI, monitoring, and secret management per organizational policy.

If you want, I can:

- Add a GitHub Actions CI workflow
- Replace client-side `falKey` storage with a secure server-side flow
- Add Playwright E2E tests that exercise the image generation flow
