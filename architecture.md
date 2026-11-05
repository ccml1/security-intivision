# intivision @ 5999fbf — architecture summary (run-1, quick profile)

## 1. Product, principals, protected resources
"GUARDIAN / VIGIA": neighbourhood video-surveillance. YOLO person/pose detection scores scenes (BAJO/MEDIO/ALTO), stores events and evidence JPEGs, triggers a siren/LED (ESP32 via MQTT) and Telegram photo alerts. Multi-tenant web dashboard at intivision.online (Caddy -> FastAPI).

Principals:
- **Anonymous web visitor**: `/login`, `/health`, unauthenticated `/ingest` attempts, WebSocket handshake attempts.
- **`resident`**: own private cameras (create, max 10), live stream `/ws/live`, upload `/analyze` (≤200 MB), read own events/evidence, toggle `admin_can_view` on own cameras.
- **`community_admin`**: resident abilities plus read of community-wide events/evidence/cameras that are common, cameraless, or `admin_can_view` (admin views of others' private resources are audited).
- **Edge device (mini PC)**: bearer token (SHA-256 stored), `POST /ingest` only, bound to one community and its assigned cameras.
- **MQTT peers**: anyone able to publish/subscribe on the configured broker/topic; the ESP32 obeys `ON[:n]` / `WARN` / `OFF`.
- **Operator**: shell in the container, `backend/manage.py` (full DB authority, trusted by design).

Protected resources: other users' private camera events and evidence images, session/edge tokens, account availability (lockout), the shared siren and Telegram channel, the DB, host resources.

## 2. Comparable baseline
No source-grounded comparable product. Protocol baselines: MQTT on a public broker (HiveMQ) without auth is world-readable/writable by topic name; Caddy reverse proxy with X-Forwarded-For trust.

## 3. Stack, deployment, execution limits
Python 3.11, FastAPI/uvicorn, psycopg2 (parameterized SQL), bcrypt, OpenCV/FFmpeg decoders, ultralytics YOLO, paho-mqtt, requests (Telegram), custom HTML templating (`backend/pages.py` `fill`/`esc`), vanilla JS (`backend/templates/index.html`), PostgreSQL 16 (`bd_intivision/schema_vigia.sql`, composite `(id, community_id)` FKs), ESP32 Arduino sketch (`wokwi/sketch.ino`).

Deployment (`docker-compose.yml`): default services `backend` (published `8000:8000` on all interfaces, root, mounts `./events`, `./backend/data`) and `caddy` (80/443 -> `backend:8000`). Profile `live`: `detector` (`main.py`) and `mqtt` (mosquitto, `1883:1883`, `allow_anonymous true`). Postgres in a separate compose bound to `127.0.0.1:5433`. `config.yaml` (tracked) selects actuator `mqtt` on `broker.hivemq.com:1883`, topic `guardian/demo7k3x9/siren`, null credentials; the same broker/topic are hard-coded in `wokwi/sketch.ino`.

Execution: no OS-enforced sandbox (Windows host, Docker daemon down), no Postgres, no models. **No target code is executed in this run; all evidence is source-only.** No test suite exists (`backend/test_ingest.py` is a manual network client — prohibited).

## 4. Entry surfaces and key source-to-sink paths
- `POST /login` (backend/app.py ~437) -> `auth.authenticate` (bcrypt, per-account lockout in DB, per-IP in-memory throttle, `client_ip` trusts last XFF hop from private peers) -> session cookie `vigia_session` (HttpOnly, SameSite=Lax, Secure from X-Forwarded-Proto).
- Unsafe-method middleware `seguridad` -> `cross_site_request` (Origin vs Host/ALLOWED_ORIGINS, then Sec-Fetch-Site; absent headers pass).
- `WS /ws/live?camera=` -> `origin_allowed` (absent Origin passes) -> cookie session (checked once) -> `resolve_camera` (owner, private) -> unbounded JPEG frames -> `cv2.imdecode` -> YOLO under shared `infer_lock` -> `insert_event` -> `dispatch_live_alert` (global per-level cooldown) -> MQTT + Telegram.
- `POST /analyze` -> `limitar_subidas` (session + Content-Length) -> `resolve_camera` -> `clean_filename` -> temp file -> `cv2.VideoCapture` -> evidence `evt_{label}_{ms}.jpg` in shared `events/` -> `insert_event` -> `dispatch_alerts` (no cooldown) -> MQTT + Telegram.
- `POST /ingest` -> bearer `edge_from_token` -> Content-Length ≤100k check -> `validate_event` (image regex, camera UUID) -> `edge_camera` -> `insert_event` -> `media_files.storage_key` UNIQUE `ON CONFLICT DO NOTHING` (file need not exist).
- `GET /evidence/{name}` -> regex -> `media_files` scope query -> `FileResponse(os.path.join(EVIDENCE_DIR, name))`.
- `GET /`, `/events`, `/cameras`, `/cameras/{id}`, `POST /cameras`, `POST /cameras/{id}/visibilidad` -> four separate scoping SQL implementations (`list_events`, `cameras._scope`, `get_camera`, evidence query) -> server-rendered HTML via `esc`; client re-injects server HTML with `innerHTML` (index.html ~649).
- MQTT: backend/detector publish fixed payloads; ESP32 subscribes and acts with no sender identity; `ON:<n>` duration unbounded in the sketch.

## 5. Trust boundaries and strongest visible control
- Internet -> app: Caddy TLS; but backend port 8000 also published directly (deployment fact decides exposure).
- Visitor -> session: bcrypt + lockout + per-IP throttle; cookie flags.
- Cross-site browser -> state change: Origin/Sec-Fetch-Site check + SameSite=Lax.
- User -> other user's / other community's data: per-route SQL scoping + composite FKs.
- Edge device -> community data: token -> community/camera binding.
- Anyone on broker -> physical siren: topic name only (no auth/TLS).
- Users of one community -> shared alerting resources: global cooldown only.
- Repo/history -> secrets: `.gitignore`; history contains a removed Caddy `basic_auth` bcrypt hash (commits b99bce4..41b30dc, removed in 7345cc9).

## 6. Starting paths
`backend/app.py`, `backend/auth.py`, `backend/cameras.py`, `backend/pages.py`, `backend/db.py`, `backend/templates/index.html`, `backend/templates/login.html`, `bd_intivision/schema_vigia.sql`, `engine.py`, `actuator.py`, `notifier.py`, `settings.py`, `config.yaml`, `wokwi/sketch.ino`, `docker-compose.yml`, `Caddyfile`, `mosquitto.conf`, `backend/Dockerfile`, `Dockerfile`.

## 7. Prior coverage
None: no prior ledger or findings exist for this repository.

## 8. Companion selection
- `WEB-PROTOCOL-AND-AUTH.md`: sessions, CSRF, cookie transport, XFF trust, bearer tokens for edge devices.
- `CLIENT-SIDE.md`: WebSocket origin handling, innerHTML re-injection of server HTML.
- `DATA-ISOLATION-AND-LIFECYCLE.md`: multi-tenant events/evidence/cameras, flat shared evidence namespace, storage_key claiming.
- `PROTOCOLS-RPC-AND-MESSAGING.md`: MQTT siren control topic.
- `RESOURCE-EXHAUSTION-AND-AVAILABILITY.md`: video/frame decoding and YOLO work by authenticated users, shared alert channel.
- `CLOUD-AND-DEPLOYMENT.md`: compose port publication, Caddy ingress, anonymous mosquitto, root containers.
- Not selected: `SUPPLY-CHAIN-AND-RELEASE.md` (no CI/release/plugins; unpinned deps handled as an obvious-things hardening check), `AI-AND-LLM.md` (no LLM), `MEMORY-SAFETY-AND-BINARY.md` (no native code in repo; OpenCV/FFmpeg are third-party), `DESKTOP-MOBILE-AND-LOCAL-IPC.md` (no native app/IPC).
