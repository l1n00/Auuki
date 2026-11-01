# AGENTS.md — Auuki AI / D100 telemetry lab

## Mission
Extend the upstream open-source Auuki indoor cycling application with real-time diagnostics and an AI chat assistant that can query actual trainer telemetry during a ride. Primary test device: Decathlon Van Rysel D100 (Zwift Cog edition), connected over Bluetooth FTMS to Chrome on macOS. Rider FTP: **149 W**, configurable in settings. Preserve Auuki's existing workout, ERG, resistance and FIT functionality.

Upstream: https://github.com/dvmarinoff/Auuki (branch `master` at project creation). Respect its AGPL-3.0 license, copyright notices and attribution. Keep modifications isolated and easy to rebase.

## First action: inspect, don't assume
Before editing code, inspect README, package.json, src/, tests, Bluetooth/FTMS device adapters, workout engine, state management and build/test commands. Document the actual architecture in `docs/architecture.md`. Do not assume React, TypeScript or a particular bundler. Prefer the existing stack and patterns. Establish a working upstream build and test baseline, recording any failures.

## Non-negotiable requirements
1. **Single trainer controller.** Auuki owns the Bluetooth FTMS connection and all trainer commands. No competing BLE client. Telemetry capture is observational; no extra FTMS polling or control writes unless needed and justified.
2. **Safety.** AI must never autonomously change target power, resistance, grade, or workout state. Chat tools are read-only. Keep explicit manual controls and an obvious way to stop/disable resistance. Rate-limit commands, validate ranges and preserve existing safety behavior.
3. **No invented measurements.** Differentiate requested target, measured power, command acknowledgement, device notification and estimated latency. Missing cadence stays null: do not fabricate it from speed. Never claim physical resistance has changed merely because a write was accepted.
4. **Lightweight/local-first.** No Docker requirement. Frontend in Chrome on Mac, local Python FastAPI backend, SQLite persistence. No external telemetry services. The only required external service for AI chat is the OpenAI API.
5. **Privacy/security.** Never put the OpenAI API key in browser code or Git. Use server-side environment variables and `.env.example`. Bind backend to 127.0.0.1 by default. Limit CORS to the actual local frontend origin; validate WebSocket origin and incoming payloads. No public deployment or network exposure by default. Send only the necessary, bounded telemetry window to OpenAI when answering a question, not a continuous raw stream.
6. **Browser constraints.** Web Bluetooth requires a secure context (`localhost` or HTTPS) and a user gesture for device selection. Handle disconnections and page reloads gracefully.
7. **Compatibility.** Preserve upstream workout imports/exports, ERG, Resistance, simulation mode and existing integrations. Do not make AI or backend availability necessary to ride.
8. **Tests.** Use a deterministic simulated FTMS telemetry source so the complete system can be tested without a physical trainer.

## Proposed architecture (adapt to actual repo)
- **Auuki frontend:** an optional diagnostic module subscribes to existing normalized telemetry and command/event streams; it does not take over FTMS.
- **Telemetry bridge:** sends versioned, timestamped events via WebSocket to local backend; bounded queue, reconnection, throttled UI updates and backpressure handling.
- **Backend:** Python 3.11+ FastAPI, Pydantic models, WebSocket ingestion, SQLite storage, session endpoints, diagnostic aggregation and OpenAI chat endpoint. Use the official current OpenAI Python SDK and a configurable model.
- **AI tools:** read-only functions to obtain session summary, telemetry over a requested time window, command timeline and anomalies. Tool responses include sample count, timestamps, units, missing-field flags and data-quality indicators. The assistant answers in Italian by default and cites concrete times and values from the active session.
- **UI:** compact optional panel with connection status, actual power, requested power, cadence (if available), speed (if available), FTMS mode, last command/ack, live power-vs-target plot, and a chat drawer. Avoid disrupting Auuki's main interface.

## Telemetry contract
Define and document a stable schema, e.g.:
- `schema_version`, `session_id`, `event_id`, `event_type`
- `timestamp_wall_utc` (ISO 8601), `timestamp_monotonic_ms` (relative timing), `source`
- `power_w`, `target_power_w`, `cadence_rpm`, `speed_kph` (nullable where unavailable)
- `mode` (`erg`, `resistance`, `simulation`, `unknown`)
- `resistance_level`, `grade_pct` (nullable)
- `command_id`, `command_type`, `command_value`, `ack_status`, `error_code` (as applicable)
- `device_name`, `connection_state`, `quality_flags`

Capture raw FTMS bytes optionally for debugging with explicit opt-in and documentation; also retain parsed measurements. Timestamp at receipt, retain actual event ordering, and make sampling/drop statistics visible. Target 1 Hz normalized display/storage where appropriate, but retain relevant raw command/ack events and avoid hiding rapid transitions. Do not conflate a 1 Hz visualization with Bluetooth notification frequency.

## Diagnostics
Implement deterministic calculations for:
- Target vs actual power: error, absolute error, moving average, variability.
- Step response: command timestamp, acknowledgement latency where observable, settling time within configurable tolerance (e.g. ±10 W or ±5%, whichever is larger, sustained for configurable seconds), and overshoot.
- Disconnects, stale telemetry, missing measurements, rejected/unsupported commands, and anomalous oscillations.
- Distinguish protocol acknowledgement from mechanical power settling; expose unknown/insufficient-data cases explicitly.
- FTP-based zone display using 149 W default; user-editable and stored locally.

## Implementation phases
### Phase 0 — Baseline and plan
- Inspect upstream and existing tests; document build/run instructions and component map.
- Identify the exact event hooks for incoming FTMS data, outgoing commands and responses.
- Propose a minimal diff and file layout. Do not rewrite upstream architecture.

### Phase 1 — Telemetry without AI
- Add feature-flagged instrumentation, in-memory ring buffer and browser diagnostics.
- Implement a simulator and tests for parsing, units, timestamping, disconnection and event correlation.
- Ensure existing Auuki behavior remains unchanged when diagnostics are disabled.

### Phase 2 — Local backend
- Add `backend/` with FastAPI, WebSocket ingestion, SQLite sessions, migrations/init, bounded retention and CSV export.
- Provide `/health`, active session summary, time-window telemetry, events and diagnostics endpoints.
- Handle reconnection, duplicate event IDs, malformed payloads, missing data and storage failures.
- Document exact local launch commands for macOS; keep riding usable if backend is down.

### Phase 3 — AI chat
- Add server-side OpenAI integration with read-only tools, bounded context and explicit data freshness.
- Chat must answer questions like “Perché la potenza oscilla?”, “Quanto tempo ha impiegato il D100 a stabilizzarsi dopo il passaggio a 120 W?” and “Il target ERG è stato raggiunto?” using recorded samples.
- If evidence is absent or stale, say so. No autonomous trainer commands.
- Make model, token budget and API usage configurable; no key in frontend.

### Phase 4 — D100 hardware validation
- Guide user through a controlled test: connect, ride at 80 W, then 100 W, 120 W and recovery, each for a short documented interval; check device support and adjust safe targets if necessary.
- Test Resistance separately with small manual changes, only on user action.
- Record firmware/device identification where available, notification frequency, actual supported FTMS operations, target response, connection stability and any Zwift Cog limitations.
- Do not assume D100 reports cadence or implements every optional FTMS feature.
- Generate a diagnostic report and add a device compatibility note based on observed evidence.

## Acceptance criteria for MVP
- Auuki still runs and controls a workout with backend and AI disabled.
- Simulator feeds realistic data through the same pipeline as hardware and produces a live graph and persisted session.
- Target commands and actual power are distinguishable and time-correlated.
- Backend reconnects safely; no duplicate samples from retransmission.
- AI can answer an example question with exact measured values and time window from SQLite; refuses to invent missing data.
- No secrets in commits, no unsolicited FTMS writes, no regressions in existing workouts.
- Automated tests and a clear `README-AI.md` with Mac setup and troubleshooting.

## Workflow for Codex
- Work in small, reviewable commits. Run relevant tests and lint after each phase.
- Before changing device-control code, explain the change and possible safety implications.
- Prefer upstream-compatible additions over invasive refactors.
- Keep a `docs/progress.md` with implemented items, verification, limitations and next steps.
- For this first Codex session, complete **Phase 0**, then implement **Phase 1** if the baseline is healthy. Do not start backend/AI implementation until telemetry hooks and simulator are validated.
- Never claim physical D100 compatibility without an actual hardware test.
