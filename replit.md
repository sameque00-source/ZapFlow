# Workspace

## Overview

pnpm workspace monorepo using TypeScript. Hosts the **ZapFlow** WhatsApp automation SaaS — an AI-powered attendant that answers WhatsApp messages, supports multiple WhatsApp sessions per user, with per-contact memory, AI funnel stages, audio transcription, and a real-time dashboard.

## Stack

- **Monorepo tool**: pnpm workspaces
- **Node.js version**: 24
- **Package manager**: pnpm
- **TypeScript version**: 5.9
- **API framework**: Express 5 + native `ws` WebSocket
- **WhatsApp**: Baileys (`baileys` package, multi-session auth under `data/users/{uid}/sessions/{sid}/wa-auth/`)
- **AI**: OpenAI via Replit AI Integrations (GPT-5.4 chat completions + Whisper audio transcription)
- **Storage**: JSON files under `artifacts/api-server/artifacts/api-server/data/` (actual runtime path due to `process.cwd()` resolution)
- **Frontend**: React + Vite + Tailwind + shadcn/ui (white + #00c853 green ManyChat/HubSpot theme)
- **Charts**: Recharts (line chart + bar chart on dashboard)
- **Auth**: Firebase Auth (Google SSO + email/password) + session cookies
- **System deps**: `ffmpeg` (declared in `replit.nix`) — required at runtime to convert OpenAI TTS mp3 → ogg/opus for WhatsApp voice notes (PTT). Must stay in `replit.nix` or audio works in Preview but fails (ENOENT) on the published VM deploy.

## Artifacts

- `artifacts/api-server` — Express + WebSocket backend at `/api` and `/ws`. Runs Baileys multi-session, OpenAI calls, broadcasts live events.
- `artifacts/zapflow` — React + Vite operator dashboard at `/`. Pages: Dashboard (charts), Sessões, Mensagens, Treinamento, Configurações, Planos.
- `artifacts/mockup-sandbox` — design sandbox at `/__mockup`.

## Environment / Secrets

- `FIREBASE_API_KEY` — Firebase project API key (client-side)
- `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` — Google OAuth for Firebase
- `SESSION_SECRET` — Express session cookie signing
- `AI_INTEGRATIONS_OPENAI_BASE_URL`, `AI_INTEGRATIONS_OPENAI_API_KEY` — Replit OpenAI integration

## Data Layout

```
artifacts/api-server/artifacts/api-server/data/users/{userId}/
  training.json                          # Global AI training text (fallback for all sessions)
  sessions-list.json                     # List of sessions [{id, name, createdAt}]
  sessions/{sessionId}/
    training.json                        # Per-session AI training (overrides global if present)
    wa-auth/                             # Baileys multi-file auth state
    contacts.json                        # Per-session contacts with history + funnel stage
    messages.json                        # Per-session message log
```

On first access, existing flat data is auto-migrated to the multi-session format (session "default").

## Backend Endpoints (`/api`)

### Auth
- `POST /auth/register`, `POST /auth/login`, `POST /auth/logout`, `GET /auth/me`
- `POST /auth/firebase-token` — Firebase idToken → session cookie

### WhatsApp Sessions (multi-session)
- `GET /sessions` — list all sessions with runtime status
- `POST /sessions` — create new session `{name}`
- `DELETE /sessions/:id` — delete session
- `POST /sessions/:id/connect` — start QR flow
- `POST /sessions/:id/disconnect` — pause connection
- `POST /sessions/:id/logout` — clear auth + disconnect
- `POST /sessions/:id/reset-qr` — force new QR
- `GET /sessions/:id/contacts` — contacts for session
- `GET /sessions/:id/messages` — messages for session
- `GET /sessions/:id/stats` — stats for session

### Backward-compat (operates on "default" session)
- `GET /whatsapp/status`, `POST /whatsapp/connect`, `/disconnect`, `/logout`, `/reset-qr`

### Data
- `GET /messages?limit=…` — all messages across all sessions
- `GET /contacts?sessionId=…`, `GET /contacts/:jid?sessionId=…`
- `GET /stats` — aggregated stats across all sessions
- `GET /stats/messages-by-day?days=7` — chart data

### Training
- `GET /training`, `PUT /training`

### WebSocket events (`/ws`)
- `status` — `{sessionId, state, qrDataUrl, jid, user, lastError, updatedAt}`
- `sessions_list` — full sessions with runtime state (sent on WS connect)
- `message` — `{sessionId, message}`
- `typing` — `{sessionId, jid, number, name, active}`
- `stats` — aggregated stats object

## Frontend Pages

| Route | Page | Description |
|-------|------|-------------|
| `/` | Dashboard | Recharts line + bar charts, stat cards, recent activity |
| `/sessoes` | Sessions | Multi-WA session management, QR codes, connect/disconnect |
| `/mensagens` | Messages | Contact list with funnel stage filter + chat history pane |
| `/treinamento` | Training | AI training text editor |
| `/configuracoes` | Settings | Account info, session overview, AI settings links |
| `/planos` | Plans | Free / Pro / Premium pricing cards |

## AI Behaviour

- **Single-product assistant lock**: each assistant (session) is bound to ONE fixed product (its `session_product_links` entry). The active product is resolved by `resolveBoundProduct()` in `whatsapp.ts` and is NEVER switched by the client's message. Images come only from the bound product; the AI prompt (`buildEnforcedSystem`, "ASSISTENTE DE PRODUTO ÚNICO" rule) forbids switching/mixing catalogs/inventing products and redirects when the client asks about another product.
- Model: GPT-5.4 (via Replit OpenAI integration)
- Memory: last 30 messages per contact stored in `history`
- Funnel stages: `novo` → `conversando` → `interessado` → `cliente`
- Interest score: 0–10, auto-updated per message keyword
- Typing simulation: delay proportional to reply length
- Audio: Whisper transcription via OpenAI API

## Key Commands

- `pnpm run typecheck` — full typecheck across all packages
- `pnpm --filter @workspace/api-server run dev` — API + WebSocket server
- `pnpm --filter @workspace/zapflow run dev` — ZapFlow web dashboard

See the `pnpm-workspace` skill for workspace structure and TypeScript setup.
