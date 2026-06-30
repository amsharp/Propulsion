# CLAUDE.md

Guidance for an AI agent resuming work on this repo. Read this first.

## What this project is

**Propulsion** lets the owner schedule, voice-control, and auto-recover a
**Bestway / Lay-Z-Spa "Airjet"** WiFi hot tub through **Google Home**. It is a
single long-running Node.js service that acts as a **standalone Google Smart
Home Action** (no Home Assistant). It also auto-clears the pump's **low-flow
(E02)** fault, which otherwise beeps until cleared.

Owner context (from the original conversation):
- Hardware: **Bestway Airjet** pump (classic Gizwits-cloud model, `product_name`
  `"Airjet"`). NOT the newer AWS-IoT "V02" models.
- Chosen approach: **standalone Google Smart Home Action** (not HA).
- Day-one scope: on/off scheduling, set temperature, status/notifications.
- Specifically requested: when it faults, **power-cycle to clear it and warn me**
  — and have the watchdog **trigger off the error in real time** (done via the
  Gizwits push WebSocket).
- Deployment target: **Railway** (Dockerfile + railway.json shipped).

## Critical constraints for the agent

- **Git branch:** develop and push ONLY to `claude/hot-tub-scheduling-temp-nh22ib`.
  Always `git push -u origin claude/hot-tub-scheduling-temp-nh22ib`.
- **GitHub scope:** access is restricted to `amsharp/propulsion` only. Do not try
  to read/search other repos.
- **Do NOT create a PR** unless the user explicitly asks.
- **Secrets:** never commit `.env`, tokens, or the Home Graph service-account
  JSON. Don't ask the user to paste account-level credentials into chat; prefer
  scoped tokens / dashboard entry. (User asked about deploying via a Railway
  token — we recommended a scoped project token + dashboard for secrets, and
  noted the Google account-linking step is interactive and can't be automated.)
- **Model identity:** never put the model id in commits/PRs/code.

## Architecture (data flow)

```
Google Home ──HTTPS──▶ /fulfillment ─▶ google/smarthome.js ─▶ BestwayClient ─▶ Gizwits cloud ─▶ Spa
                       /oauth/*  (account linking, google/oauth.js)              ▲
Scheduler (node-cron) ─────────▶ BestwayClient ─────────────────────────────────┤
Gizwits push WebSocket ─▶ FaultWatchdog.handleRealtimeAttrs ─▶ restart + notify ─┘
Polling watchdog (every 2 min) ─▶ FaultWatchdog.check ───────────────────────────┘
```

## File map

| Path | Responsibility |
| --- | --- |
| `src/index.js` | Entrypoint: wires client + scheduler + watchdog + realtime + server, starts listening. |
| `src/server.js` | Express app: `/fulfillment` (Bearer), OAuth routes, `/api/*` admin (X-Admin-Token), `/healthz`. |
| `src/config.js` | Env/.env loader (no dotenv dep). All config in one object. `DATA_DIR` override for volumes. |
| `src/log.js` | Tiny leveled logger. |
| `src/store.js` | Atomic JSON-file persistence (used by oauth + schedules). Honors `config.dataDir`. |
| `src/bestway/constants.js` | Gizwits app id, regional roots, **Airjet attribute profile**, **fault codes + `detectFaults()`**. |
| `src/bestway/client.js` | `BestwayClient`: login/token, status (with faults), set temp/power/heat/filter/bubbles, `restartCirculation`. `fetchImpl` injectable. |
| `src/bestway/realtime.js` | `GizwitsRealtime` push WebSocket + pure `parseMessage`/`buildLoginFrame`. |
| `src/google/smarthome.js` | SYNC/QUERY/EXECUTE. Spa modeled as THERMOSTAT (OnOff + TemperatureSetting). C/F conversion. |
| `src/google/oauth.js` | Minimal OAuth2 auth-code server for Google account linking. |
| `src/google/homegraph.js` | Optional reportState/requestSync via service-account JWT. No-op if unconfigured. |
| `src/scheduler/scheduler.js` | `Scheduler`: cron jobs from `data/schedules.json`. Injectable `store` for tests. |
| `src/recovery/watchdog.js` | `FaultWatchdog`: detect → restart circulation → notify; back-off + cooldown; `trigger()` guard; `handleRealtimeAttrs`. |
| `src/notify/notifier.js` | `formatStatus` + webhook `notify`. |
| `src/cli.js` | CLI: devices/status/dump/on/off/temp/bubbles/faults/recover/schedule:*. |
| `test/*.test.js` | node:test suites (21 tests) with injected fakes; no network/account needed. |
| `Dockerfile`, `railway.json`, `.dockerignore` | Railway deploy. `DATA_DIR=/data`, healthcheck `/healthz`. |

## Reverse-engineered Bestway/Gizwits API (the crux — verify against real account)

Source of truth: community integration `github.com/cdpuk/ha-bestway`. The Bestway
cloud has no official public API and may change. Constants live in
`src/bestway/constants.js`.

- **Gizwits app id:** `98754e684ec045528b073876c34c7348` (header `X-Gizwits-Application-Id`).
- **API roots:** EU `https://euapi.gizwits.com`, US `https://usapi.gizwits.com`.
- **Headers:** `Content-Type: application/json; charset=UTF-8`, app-id header, and
  `X-Gizwits-User-token: <token>` once logged in.
- **Login:** `POST /app/login` body `{username, password, lang:"en"}` →
  `{uid, token, expire_at}` (expire_at = epoch seconds).
- **List devices:** `GET /app/bindings` → `{devices:[{did, product_name,
  dev_alias, is_online, host, wss_port, ...}]}`.
- **Status:** `GET /app/devdata/{did}/latest` → `{attr:{...}}`.
- **Control:** `POST /app/control/{did}` body `{attrs:{key:value}}`.
- **Push WebSocket:** `wss://{host}:{wss_port}/ws/app/v1` (host/port from binding).
  Login frame `{cmd:"login_req", data:{appid, uid, token, p0_type:"attrs_v4",
  heartbeat_interval:180, auto_subscribe:true}}`; updates arrive as
  `{cmd:"s2c_noti", data:{did, attrs}}`; heartbeat `{cmd:"ping"}` every 180s.

**Airjet attribute keys** (`AIRJET_PROFILE.attrs`): `temp_now` (current),
`temp_set` (target), `temp_set_unit` (0/C, 1/F), `power`, `heat_power`,
`filter_power`, `wave_power` (bubbles), `locked`. On/off values are 1/0.
Temp range clamps: C 20–40, F 68–104.

**Fault codes** (`detectFaults` scans `attr`): keys matching `E\d{2}`,
`system_err\d+`, or `earth` that are truthy. **`E32` is NOT a fault** (means
"target reached") and is excluded. **`E02` = low water flow** and is the only
`AUTO_CLEARABLE_CODES` member by default. `ERROR_MEANINGS` maps codes to text.

## Google Smart Home model

- One device, type `action.devices.types.THERMOSTAT`, traits `OnOff` +
  `TemperatureSetting` (modes `off`/`heat`, unit `C`, range 20–40).
- **OnOff** → `client.setHeating(on)` (power + heater + filter).
- **ThermostatSetMode** heat/off → `setHeating`.
- **ThermostatTemperatureSetpoint** (Google always Celsius) → converted to the
  pump's configured unit before `setTargetTemperature`.
- QUERY reads current/target temp, converting pump F→C for Google.
- Account linking: owner-only consent gated by `LINK_USERNAME`/`LINK_PASSWORD`
  (these are NOT the Bestway creds). OAuth client id/secret are owner-chosen and
  must match the Actions console.

## Commands

```bash
npm install
npm test                       # 21 tests, all should pass, no network needed
npm start                      # run the full service
npm run cli -- status          # quick check against the real account (needs .env)
npm run cli -- dump            # print raw attrs — use to verify/fix attribute keys
npm run cli -- faults | recover
```

Config: copy `.env.example` → `.env`. Key vars: `BESTWAY_USERNAME/PASSWORD/REGION`,
optional `BESTWAY_DEVICE_ID`; `PUBLIC_BASE_URL`; `OAUTH_CLIENT_ID/SECRET`,
`LINK_USERNAME/PASSWORD`; watchdog (`AUTO_CLEAR_LOW_FLOW`, `WATCHDOG_*`,
`REALTIME_ENABLED`); `DATA_DIR` (point at a volume in prod); optional
`HOMEGRAPH_SERVICE_ACCOUNT_FILE`, `NOTIFY_WEBHOOK_URL`.

## Deployment (Railway)

Dockerfile + `railway.json` shipped. Steps: deploy repo → **add a persistent
volume at `/data`** (else OAuth tokens + schedules wiped on redeploy; Dockerfile
sets `DATA_DIR=/data`) → set env vars (do NOT set `PORT`, Railway injects it) →
generate domain → set `PUBLIC_BASE_URL` to it. Health check: `GET /healthz`.
Must be always-on (no sleep) for scheduler/watchdog/socket. README §3b has the
full walkthrough.

## Current state

- ✅ Implemented + committed + pushed: Bestway client, fault detection/recovery,
  real-time socket, Google fulfillment + OAuth, optional Home Graph, scheduler,
  notifications, CLI, server, Railway deploy files, README, CLAUDE.md.
- ✅ All 21 unit tests pass; server boots and `/healthz` responds; degrades
  gracefully (real-time → polling) when the cloud is unreachable.
- Two commits on the branch: the app, then the Railway setup.

## NOT YET VERIFIED / likely next work

1. **Live API correctness** — code was NOT run against a real Bestway account
   from this environment. First real-world step: `npm run cli -- status` /
   `dump`. If any attribute key differs on the user's firmware, fix the single
   `AIRJET_PROFILE.attrs` map. Endpoint paths/attribute names are the highest-risk
   guesses.
2. **Push WebSocket** host/port/frames are unverified against the live cloud.
3. **Google Actions console** project + account linking is interactive and not
   done — user must do it; we provide the URLs/handlers.
4. **Railway deploy** not actually executed; build/health unproven on the platform.
5. Home Graph reportState/requestSync untested (optional feature).

## Conventions

- ESM, Node ≥18.17 (Railway image uses node:22). No TypeScript.
- Minimal deps (only `express`, `node-cron`); tests use built-in `node:test`.
- Inject collaborators (`fetchImpl`, `store`, `wait`, `now`, `WebSocketImpl`) to
  keep units testable without network/timers — keep this pattern when extending.
- Run `npm test` before committing. End commit messages with the required
  Co-Authored-By / Claude-Session trailers.
