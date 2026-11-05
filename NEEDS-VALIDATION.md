# NEEDS-VALIDATION — intivision run-1 (perfil quick)

Pistas con traza en el código fuente cuya conclusión depende de un hecho que no está en el repositorio o que requiere ejecutar el código. **No son vulnerabilidades confirmadas y no llevan severidad.** Ordenadas por prioridad sugerida. El contenido de cada registro se copia tal cual de `findings.json` (en inglés).

## 1. Siren control topic sits on a public MQTT broker with no producer authentication

- **Fingerprint:** `wokwi/sketch.ino:onMessage:unauthenticated-siren-control-on-public-mqtt-broker`

The tracked config.yaml sets the actuator to type mqtt on the public broker broker.hivemq.com:1883, topic guardian/demo7k3x9/siren, with null credentials. docker-compose.yml mounts this file read-only into the backend and detector. settings.py lets the environment override only the MQTT username and password, never host or topic. So the deployed backend sends its ON:10 and WARN commands to that public topic unless the operator edits the file on the server. The only consumer in the repository is wokwi/sketch.ino. It hard-codes the same broker and topic, connects anonymously over plaintext TCP, and acts on every message with no sender identity, signature or freshness check. Any payload that starts with ON sounds the buzzer and LED for an unbounded number of seconds; ON:2592000 holds it about 30 days, and repeated messages extend it. OFF, or WARN (which calls noTone), silences an active alarm. If the broker admits anonymous clients to this topic, any other client could therefore sound, prolong or silence whatever device subscribes, and could subscribe to learn in real time when the system raises MEDIO/ALTO alerts. As committed, the sketch joins the WiFi network Wokwi-GUEST and its header calls it an ESP32 simulated in Wokwi; the backend comments also say the MQTT commands are there to be seen in Wokwi. So the repository shows only a simulated consumer. Real-world impact needs a physical device that subscribes to this topic, and a broker that admits anonymous access. Neither can be decided from source.

**Causa raíz propuesta:** Siren control has no producer authentication; the only control is a topic name on a shared public broker. The producer (actuator.py MQTTActuator, configured by the tracked config.yaml) and the consumer (wokwi/sketch.ino connectMqtt) both connect without credentials or TLS. The consumer (wokwi/sketch.ino onMessage) neither authenticates nor bounds commands.

**Traza:**

- `entrypoint` [wokwi/sketch.ino:41](wokwi/sketch.ino) (onMessage): Callback for every message the broker delivers on the subscribed topic. It receives the raw payload with no publisher identity.
- `propagation` [wokwi/sketch.ino:52](wokwi/sketch.ino) (onMessage): Any payload starting with 'ON' is treated as a high-level alarm command.
- `propagation` [wokwi/sketch.ino:55](wokwi/sketch.ino) (onMessage): The duration is parsed with toInt() from the publisher's text, with no lower or upper bound.
- `sink` [wokwi/sketch.ino:57](wokwi/sketch.ino) (onMessage): tone(PIN_BUZZER, 1200) sounds the buzzer (LED set at line 56). Auto-off is scheduled at line 58 from the unbounded duration.

**Evidencia:**

- config.yaml:48: Tracked actuator host is the public broker broker.hivemq.com, port 1883 (line 49); type mqtt at line 46.
- config.yaml:50: Tracked siren topic guardian/demo7k3x9/siren.
- config.yaml:51: MQTT username and password are null (lines 51-52).
- settings.py:18: The environment can override only the actuator username and password (lines 18-19), not host or topic.
- docker-compose.yml:19: Repository config.yaml mounted read-only into the backend; the same for the detector at line 37.
- actuator.py:44: Backend producer connects in plaintext with no tls_set; credentials are applied only if set (lines 40-41).
- backend/app.py:375: Backend publishes ON on every live ALTO alert that passes the cooldown, so a subscriber could see it and answer with OFF.
- backend/app.py:333: dispatch_alerts docstring says one MQTT command is sent per event 'para verla en Wokwi'; line 351 repeats that the messages are spaced to be seen in Wokwi.
- wokwi/sketch.ino:2: Header describes the firmware as an ESP32 simulated in Wokwi.
- wokwi/sketch.ino:20: WiFi SSID Wokwi-GUEST with an empty password (line 21): as committed, the firmware joins the simulator's network, not a site network.
- wokwi/sketch.ino:24: Firmware hard-codes the same topic; the broker host is hard-coded at line 22.
- wokwi/sketch.ino:29: Plain WiFiClient (no TLS) for the MQTT connection.
- wokwi/sketch.ino:86: mqtt.connect(id) with no username or password: an anonymous session.
- wokwi/sketch.ino:58: offAt uses 32-bit unsigned arithmetic on the unbounded duration; ON:2592000 keeps the buzzer on about 30 days while uptime is under about 19.7 days.
- wokwi/sketch.ino:61: WARN calls noTone, silencing an active buzzer.
- wokwi/sketch.ino:66: OFF turns buzzer and LED off immediately, whoever sent it.

**Bloqueos (lo que falta saber):**

- Whether any physical siren or LED device subscribes to guardian/demo7k3x9/siren is a deployment fact. The only consumer in the repository is wokwi/sketch.ino, which as committed joins the Wokwi-GUEST simulator network. With only the simulation, no real-world alarm can be sounded or silenced.
- The access policy of broker.hivemq.com (anonymous publish, subscribe, '#' wildcard and retained messages on this topic) is a third-party broker fact. Contacting the broker is prohibited in this run.
- Whether the production backend and detector still mount the committed config.yaml unchanged (host broker.hivemq.com, this topic) is a deployment fact. The environment cannot override host or topic, but the operator may edit the file on the server.

**Plan de validación:**

- *Despliegue (lo observa el dueño, sin pruebas en vivo):* Owner-observed only. Do not publish to or subscribe on the production topic, and do not contact the broker. (1) On the server, read the config.yaml mounted into guardian-backend (and guardian-detector, if it runs) and record actuator.type, host, port, topic and whether MQTT_USERNAME/MQTT_PASSWORD are set. (2) State whether any physical ESP32 or other siren/LED device is installed and which broker and topic its firmware uses, or whether only the Wokwi browser simulation subscribes. (3) From the broker operator's published documentation, record whether broker.hivemq.com accepts anonymous publish, subscribe, '#' subscriptions and retained messages. The candidate holds only if (1) still names the public broker and topic, (2) names a physical device on that topic, and (3) admits anonymous access.

---

## 2. /ingest lets an edge device register any evidence filename as its community's media, without proof it produced the file

- **Fingerprint:** `backend/app.py:ingest:edge-supplied-evidence-storage-key`

POST /ingest takes an 'image' name from the edge device and checks only its character set. insert_event then stores that name as a media_files.storage_key for the device's community, with camera NULL unless an assigned camera is given. It does not check that the device produced or uploaded the file, and the file need not exist yet: the size is simply recorded as 0. No endpoint exists through which an edge device can upload an image, so the name can only refer to files other principals write into the shared EVIDENCE_DIR.

storage_key is globally unique with first-writer-wins conflict handling. A device holding a valid token for community A can therefore pre-claim names that will later be produced elsewhere. GET /evidence/{name} then lets community A's admin read the file through the c.id IS NULL branch. Examples are a community B resident's predictable evt_<filename>_<ms>.jpg from /analyze, or a hidden private camera's video evidence in community A itself. It can also claim the live detector's unowned intruso_*/clip_*/events.jsonl files when that profile shares ./events. When the victim's analysis later inserts its row, the row is silently dropped, so the victim loses access to their own evidence image. live_ names use microsecond timestamps and are not practically predictable. The path is source-complete but was not executed; the cross-community read also depends on predicting the victim's upload filename.

**Causa raíz propuesta:** insert_event accepts a client-chosen object key from /ingest, validated only by IMAGE_NAME_RE, as a storage_key in a tenant-agnostic global namespace with ON CONFLICT DO NOTHING. Nothing binds the key to an object the edge device produced. /evidence authorises from that row and serves the file found by bare name in the shared directory.

**Traza:**

- `entrypoint` [backend/app.py:619](backend/app.py) (ingest (POST /ingest)): Edge request with a bearer token, resolved to the device at line 623.
- `propagation` [backend/auth.py:221](backend/auth.py) (edge_from_token): Returns only the device id and community_id for an active token hash; the token carries no object-level scope.
- `propagation` [backend/app.py:104](backend/app.py) (validate_event): image is accepted if it matches ^[A-Za-z0-9._-]{1,120}$; there is no ownership or existence check.
- `propagation` [backend/app.py:644](backend/app.py) (ingest): insert_event(limpio, None, edge community_id, 'edge', camera_id), where camera_id is None unless the device names a camera assigned to it.
- `propagation` [backend/app.py:248](backend/app.py) (insert_event): The media_files row (community A, user NULL, camera NULL, storage_key = image) is inserted with ON CONFLICT (storage_key) DO NOTHING, so the first claimant owns the global name.
- `propagation` [engine.py:95](engine.py) (GuardianEngine.analyze_video): A victim's later /analyze writes evt_{label}_{stamp}.jpg (line 88) into the same EVIDENCE_DIR.
- `propagation` [backend/app.py:608](backend/app.py) (analyze): The victim's insert_event reaches the same ON CONFLICT clause, and its media row is not inserted.
- `propagation` [backend/app.py:664](backend/app.py) (evidence (GET /evidence/{name})): The admin query for community A matches the claimed storage_key; camera NULL satisfies c.id IS NULL.
- `sink` [backend/app.py:683](backend/app.py) (evidence): FileResponse(os.path.join(EVIDENCE_DIR, name)) returns the file currently on disk, which is the victim's image, to community A's admin.

**Evidencia:**

- backend/app.py:71: IMAGE_NAME_RE is a character-set check only.
- backend/app.py:228: The client value (after basename) is used verbatim as storage_key.
- backend/app.py:244: A missing file only sets the size to 0; the claim row is still inserted.
- bd_intivision/schema_vigia.sql:186: storage_key is UNIQUE globally, not scoped by community_id.
- backend/cameras.py:171: camera_id from the edge is bound to the device's community and assignment. This shows the design scopes edge input to the device, but no equivalent binding exists for image.
- backend/test_ingest.py:22: The documented edge payload sends no image field, and the backend has no edge upload endpoint, so a legitimate device has no file of its own to reference.
- engine.py:87: Video evidence names derive only from the uploader's filename and the frame's video-time milliseconds, so they can be enumerated for a guessed filename.
- backend/app.py:262: list_events selects the image by event_id: A's injected event shows the claimed image, and the victim's event has none.
- docker-compose.yml:38: The detector (profile live) mounts the same ./events directory. It writes events.jsonl and intruso_<YYYYmmdd_HHMMSS>.jpg (notifier.py:22, 27) and clip_<YYYYmmdd_HHMMSS>.mp4 (main.py:141) without media rows; this extension depends on deployment.

**Bloqueos (lo que falta saber):**

- No OS-enforced sandbox, PostgreSQL instance or YOLO model was available, so no target code ran. Neither the edge claim row nor the admin's 200 response with the victim's bytes was observed, so the run's rules do not allow a confirmed verdict.
- The cross-community capture requires predicting the victim's sanitised upload filename. Its stamp is enumerable at 1/fps*sample_every steps, but the filename is not visible to community A. How predictable real resident filenames are, and whether the live detector shares ./events in production, are deployment facts not in source.

**Plan de validación:**

- *Local (en sandbox, datos ficticios):* In an isolated sandbox with no network, load bd_intivision/schema_vigia.sql into a throwaway PostgreSQL 16. Create communities A and B, admin AA in A, an active edge device in A with a known dummy token, and resident RB in B with private camera CB. Point EVIDENCE_DIR at a scratch directory. With FastAPI TestClient: (1) POST /ingest with the edge token and body {"level":"BAJO","score":0,"people":0,"image":"evt_clip.mp4_00000500.jpg"}; expect 200 and a media_files row (community A, user_id NULL, camera_id NULL, size_bytes 0). (2) Write a marker JPEG to EVIDENCE_DIR/evt_clip.mp4_00000500.jpg and call insert_event({'level':'MEDIO','score':3,'people':1,'image':'evt_clip.mp4_00000500.jpg'}, user=RB, source_kind='video', camera_id=CB), as analyze_video plus /analyze would. Expect no new media_files row. (3) GET /evidence/evt_clip.mp4_00000500.jpg as AA: expect 200 with the marker bytes. As RB: expect 404. Stop at the first cross-community byte match.
- *Despliegue (lo observa el dueño, sin pruebas en vivo):* Owner-observed, read-only query on the production database: SELECT storage_key, community_id, created_at FROM media_files WHERE user_id IS NULL AND (storage_key LIKE 'evt\_%' OR storage_key LIKE 'live\_%' OR storage_key LIKE 'intruso\_%' OR storage_key LIKE 'clip\_%' OR storage_key = 'events.jsonl'). Rows returned are edge claims of backend- or detector-generated names. Also confirm whether any edge device legitimately sends an image field, and whether the live profile runs on the same host as the backend.

---

## 3. Video evidence names are not unique per owner, so /evidence can serve one user's frame to whoever claimed the name first

- **Fingerprint:** `backend/app.py:evidence:first-claimant-storage-key-serves-shared-evidence-file`

POST /analyze names each evidence frame evt_<sanitised upload filename>_<8-digit video-time ms>.jpg and writes it into the single EVIDENCE_DIR that every user, community and the optional live detector share. The name has no user, community or random part. insert_event stores the name as media_files.storage_key, which is globally UNIQUE, using ON CONFLICT (storage_key) DO NOTHING. GET /evidence/{name} authorises the caller against that one row and then streams whatever file is currently on disk under the name.

If two residents, in the same or different communities, upload files with the same sanitised name and both analyses fire an event at the same video-time stamp, the second cv2.imwrite replaces the first owner's frame. The second owner's media row is silently dropped. The first owner, and that owner's admin when the camera is visible to them, are then served the second user's private frame. The second user's event shows no image. Run in reverse, a deliberate later upload replaces a victim's stored evidence with attacker content. A deliberate earlier upload pre-claims names so the attacker reads a victim's later frame. Source establishes the path; the cross-owner read was not observed because no target code was run. The edge /ingest pre-claim of arbitrary names is a separate missing control, recorded under backend/app.py:ingest:edge-supplied-evidence-storage-key.

**Causa raíz propuesta:** Evidence object keys for uploaded videos come only from the client filename and video-relative time (engine.py:87-88), not from a server-generated, owner-scoped key. They are written by bare name into one shared directory, and the global storage_key keeps the first claimant's row while a later writer replaces the file. /evidence binds authorisation to the row, not to the file actually produced for that principal: sha256 is never populated and there is no per-owner path segment.

**Traza:**

- `entrypoint` [backend/app.py:577](backend/app.py) (analyze (POST /analyze)): An authenticated resident uploads a video; the multipart filename is chosen by the uploader.
- `propagation` [backend/app.py:585](backend/app.py) (analyze): label = clean_filename(video.filename) is deterministic for a given client filename. It is passed to the engine as source_label at line 603.
- `propagation` [engine.py:88](engine.py) (GuardianEngine.analyze_video): img_name = evt_{source_label}_{stamp}.jpg, where stamp is int(video_time*1000) zero-padded (line 87) and video_time = idx / fps (line 76).
- `propagation` [engine.py:95](engine.py) (GuardianEngine.analyze_video): cv2.imwrite writes the frame to EVIDENCE_DIR/<img_name> with no existence check, so it replaces a file already there.
- `propagation` [backend/app.py:248](backend/app.py) (insert_event): The media_files INSERT uses ON CONFLICT (storage_key) DO NOTHING, so the first claimant keeps the row and a later producer's row is dropped. It is called per event at line 608.
- `propagation` [backend/app.py:669](backend/app.py) (evidence (GET /evidence/{name})): The resident query matches only the stored row (m.user_id = caller OR c.user_id = caller); the admin query (lines 662-664) matches by community and camera visibility.
- `propagation` [backend/app.py:673](backend/app.py) (evidence): The only file check is os.path.exists on EVIDENCE_DIR/<name>; nothing ties the bytes to the row's owner.
- `sink` [backend/app.py:683](backend/app.py) (evidence): FileResponse(path) returns the current bytes under the name, which may be another principal's frame, to the row's claimant.

**Evidencia:**

- bd_intivision/schema_vigia.sql:186: storage_key text NOT NULL UNIQUE is one namespace across all communities and users.
- bd_intivision/schema_vigia.sql:189: A sha256 column exists, but insert_event (backend/app.py:245-249) never sets it, so the file content is not bound to the row.
- backend/app.py:30: EVIDENCE_DIR = BASE/events is a single flat directory, commented as shared with the detector.
- backend/app.py:80: clean_filename keeps the last 80 sanitised characters of the client filename, so identical client names give identical labels.
- engine.py:87: The stamp is the zero-padded video-relative millisecond offset, with no wall-clock, owner or random component.
- engine.py:46: One ThreatScorer is shared by every analyze_video call (used at line 83), so which sampled frames produce MEDIO/ALTO events depends on prior analyses as well as the video. This makes a collision at a given stamp runtime-dependent.
- backend/app.py:610: The /analyze response returns the events list including each image name, so an uploader learns the names they now own.
- backend/app.py:262: The event's image is selected only by event_id, so after a conflict the later uploader's event shows no image while the file holds their frame.
- docker-compose.yml:38: The optional detector service (profile live, line 41) mounts the same ./events directory as the backend (line 20).

**Bloqueos (lo que falta saber):**

- No OS-enforced sandbox, PostgreSQL instance or YOLO model was available, so no target code ran. The end-to-end result, where the first claimant's GET /evidence returns another principal's frame, was not observed.
- Whether two uploads with the same sanitised filename produce events at the identical millisecond stamp depends on model detections, container fps, analysis wall-clock hour (sensitive_hours) and the engine-wide ThreatScorer state carried between analyses (engine.py:46, 83). This can only be settled by running the pipeline.

**Plan de validación:**

- *Local (en sandbox, datos ficticios):* In an isolated sandbox with no network, load bd_intivision/schema_vigia.sql into a throwaway PostgreSQL 16. Create communities X and Y, resident A in X with private camera CA, and resident B in Y with private camera CB. Point EVIDENCE_DIR at a scratch directory and stub PersonDetector.detect to return one person box on every frame (pose disabled), with sensitive_hours covering the current hour. Make two visually distinct 2-second 30 fps MP4 fixtures, both uploaded with the filename clip.mp4. With FastAPI TestClient and dummy sessions, POST /analyze as A on camera CA, recreate the engine so the scorer state is fresh, then POST /analyze as B on camera CB. Record events[].image from both responses. If a name is shared, GET /evidence/<name> as A: expect 200 with B's frame bytes, compared by hash with the file B's analysis wrote. GET as B: expect 404. /events as B should show its event with image null. Stop at the first wrong-owner byte match.
- *Despliegue (lo observa el dueño, sin pruebas en vivo):* Owner-observed, read-only query on the production database: SELECT e.id, e.community_id, e.user_id FROM events e WHERE e.source = 'video' AND NOT EXISTS (SELECT 1 FROM media_files m WHERE m.event_id = e.id). Video events with no media row are storage_key collisions that already happened; for each, compare the colliding storage_key's owning row (media_files.user_id/community_id) with the event's owner.

---

## 4. Alerts from every community's private cameras go to one global Telegram chat and siren, ignoring admin_can_view

- **Fingerprint:** `backend/app.py:get_dispatcher:global-alert-sinks-ignore-tenant-and-camera-visibility`

Every MEDIO/ALTO event the backend dispatches comes from a resident's own private camera: /ws/live and /analyze accept only the caller's private, active camera (resolve_camera, default_camera). dispatch_live_alert and dispatch_alerts receive only the event dict (image name, level, score, reasons). They send it to a process-wide dispatcher built once from config.yaml and the environment: one MQTT siren topic, one Telegram bot and chat_id, one zone name. The event's community, camera owner and admin_can_view flag are never consulted. The web application hides a private camera's events and evidence from the community admin when admin_can_view is false. It audits admin views of other residents' private cameras, and tells the owner that the community administrator cannot see this camera and its events. The Telegram path posts that camera's evidence frame to the configured chat with no visibility check and no audit. It labels every alert with the single configured zone ('Cuadra 01 - Piloto') and fires the single siren for events from any community. The schema's per-user and per-community notification_channels (with min_level) and alert_deliveries show that per-tenant routing was planned, but no code implements it. The design question is decisive. If the deployment has one community and the chat holds only the trusted operator, who already has full database and evidence access, no boundary is crossed. In that case this is a single-installation pilot design. The boundary is crossed if the chat includes anyone else (for example the community admin or neighbours) or if more than one community exists. Then private frames that residents hid from the admin, or frames from other communities, are disclosed to that chat, and other communities sound the pilot site's siren.

**Causa raíz propuesta:** get_dispatcher() (backend/app.py) builds a single global actuator and Notifier from config, and both dispatch functions drop the event's user, camera and community. Alert routing and the admin_can_view visibility rule are therefore never applied on the external alert path.

**Traza:**

- `entrypoint` [backend/app.py:539](backend/app.py) (ws_live): An authenticated resident of any community opens a live stream for their own camera.
- `propagation` [backend/cameras.py:160](backend/cameras.py) (resolve_camera): Only the caller's own private, active camera is accepted, so every dispatched event belongs to a private camera.
- `propagation` [backend/app.py:534](backend/app.py) (_process_frame): Only the event dict is passed to dispatch_live_alert. User, camera_id and community are dropped; they were used for insert_event at line 533.
- `propagation` [backend/app.py:322](backend/app.py) (get_dispatcher): Global dispatcher: one Notifier with the configured telegram_chat_id, one actuator from config.yaml (line 321), one zone name (line 324).
- `propagation` [backend/app.py:375](backend/app.py) (dispatch_live_alert): Siren command on the single configured topic, whatever the event's community.
- `propagation` [backend/app.py:381](backend/app.py) (dispatch_live_alert): The private camera's evidence frame is handed to the global notifier with no admin_can_view check and no audit.
- `sink` [notifier.py:59](notifier.py) (Notifier._send_photo): requests.post to the Telegram sendPhoto API with the single configured chat_id and the private-camera image.

**Evidencia:**

- backend/app.py:282: Web event listing hides a private camera's events from the community admin unless admin_can_view is true.
- backend/app.py:664: Evidence download applies the same admin_can_view restriction.
- backend/app.py:726: Admin views of another resident's private camera are audit-logged (insert at line 729); the Telegram path has no equivalent.
- backend/pages.py:168: The owner is told the community administrator 'no puede' (cannot) see the camera and its events when it is hidden.
- backend/templates/index.html:372: Users are told only that MEDIO/ALTO events 'activan las alertas configuradas' (trigger the configured alerts), with no recipient named.
- backend/app.py:609: Upload path dispatches the events of the caller's camera (resolved at line 582) to the same global sinks: siren at line 348, worst event's frame to the global chat at line 356.
- bd_intivision/schema_vigia.sql:227: notification_channels is scoped per user or per community, with min_level at line 237; no Python code uses it.
- bd_intivision/schema_vigia.sql:255: alert_deliveries per event and channel is defined but unused.
- backend/manage.py:57: create_community lets the operator add any number of communities; the global dispatcher is not partitioned by them.
- notifier.py:51: Every caption uses the single configured zone (config.yaml zone_name at line 1).
- settings.py:17: TELEGRAM_CHAT_ID from the environment sets the single notifier chat_id.
- docker-compose.yml:15: One TELEGRAM_CHAT_ID for the whole backend.

**Bloqueos (lo que falta saber):**

- Membership of the production TELEGRAM_CHAT_ID chat (set via .env, not in the repository) is unknown. If only the trusted operator is in it, the photo path does not cross a boundary.
- Whether the deployment holds more than one community, or residents with private cameras whose admin_can_view is false, is database state not visible in the repository.
- Whether TELEGRAM_TOKEN and TELEGRAM_CHAT_ID are set in production is unknown: config.yaml has null and the values come from .env.

**Plan de validación:**

- *Despliegue (lo observa el dueño, sin pruebas en vivo):* Owner-observed only; send no test alerts. (1) State whether TELEGRAM_TOKEN and TELEGRAM_CHAT_ID are set in the production .env. If they are, list who belongs to that chat (operator only, community admin, neighbours or others). (2) Run read-only queries against the production database: SELECT count(*) FROM communities; and SELECT community_id, count(*) FROM cameras WHERE kind = 'private' AND NOT admin_can_view GROUP BY community_id. (3) State where the device subscribed to the siren topic is installed. The candidate holds if the chat contains anyone besides the trusted operator while hidden private cameras exist, or if more than one community exists.

---

## 5. Caddy basic_auth bcrypt hash for user 'angel', once the only gate on the production site, remains in git history and on a remote-tracking branch tip

- **Fingerprint:** `Caddyfile/basic-auth-bcrypt-hash-in-git-history`

Commit b99bce4 (2026-11-01) added to the tracked Caddyfile, inside the intivision.online site block, 'basic_auth { angel $2a$14$M... }': a well-formed 60-character bcrypt hash at cost 14. At that commit backend/app.py had no application authentication (/, /events, /evidence/{name}, /analyze, /ws/live and /ingest took no credentials), so this credential was the only gate on the production dashboard until session login arrived in 41b30dc; port 8000 was already published, so the gate could also be bypassed directly at the time. cfca76c added 'Caddyfile' to .gitignore, which has no effect on an already-tracked file. 7345cc9 ('Caddyfile sin contrasena ni hash') removed the block and the ignore entry at the tip only. The hash remains in eight commits reachable from main (b99bce4, cfca76c, 89b2072, 747a21c, 352d16a, b11bddb, 1229858, 41b30dc) and at the tip of remote-tracking branch origin/chore/aplanar-bd_intivision (747a21c). The origin remote is a GitHub repository. Anyone who can read the repository gets an offline guessing oracle with no rate limit or lockout; cost 14 makes guessing slow, so only a weak, dictionary-grade or reused password would be recoverable. The current Caddyfile, which docker-compose mounts into guardian-caddy, has no basic_auth, and the application now has its own login, so the hash guards nothing in the current source. Whether this exposure matters depends on facts outside the repository: whether any reader outside the credential holders can read the repository, whether the deployed Caddyfile still contains this hash, and whether the password is weak or reused for another account. Nothing in the commits shows the password was rotated.

**Causa raíz propuesta:** A production basic_auth password hash was committed into the tracked Caddyfile. Removing it at the tip (7345cc9) did not rewrite history or delete the stale remote branch, and nothing in source shows the password was rotated.

**Traza:**

- `entrypoint` [Caddyfile:1](Caddyfile) (intivision.online site block at commit b99bce4): Commit b99bce4 added basic_auth with user 'angel' and a $2a$14$M... bcrypt hash at lines 2-4 of that version of the file, guarding the whole production site.
- `propagation` [.gitignore:1](.gitignore) (.gitignore history (commits cfca76c and 7345cc9)): Commit cfca76c appended 'Caddyfile' to .gitignore. Ignore rules do not untrack a tracked file, so the hash stayed in every later commit until 7345cc9 removed both the hash and the ignore entry.
- `sink` [Caddyfile:2](Caddyfile) (Caddyfile at 7345cc9 and HEAD 5999fbf): The block was deleted at the tip only. Reachable history on main and the origin/chore/aplanar-bd_intivision tip (747a21c) still contain the hash for any repository reader.

**Evidencia:**

- Caddyfile:1: git show b99bce4:Caddyfile shows 'basic_auth { angel $2a$14$M... }' (60 characters, cost 14). git log --all -S basic_auth lists only b99bce4 (added) and 7345cc9 (removed); scanning every commit in git rev-list --all finds the block in exactly eight commits.
- backend/app.py:1: git show b99bce4:backend/app.py shows routes /health, /ws/live, /analyze, /ingest, /events, /evidence/{name} and / with no authentication dependency or token check. Session login arrived only in 41b30dc, so basic_auth was then the only gate.
- docker-compose.yml:10: "8000:8000" has been published since 2218cb2, so between b99bce4 and 41b30dc the basic_auth gate could also be bypassed through the direct port (historical).
- docker-compose.yml:60: The deployed Caddy configuration is the host's ./Caddyfile bind-mounted read-only, so the live gate depends on the host's working copy rather than on git history.
- .gitignore:1: cfca76c added 'Caddyfile' to .gitignore without untracking it, and 7345cc9 removed that entry. The current .gitignore (lines 1-14) has no Caddyfile rule.

**Bloqueos (lo que falta saber):**

- Whether anyone other than the credential holders can read the repository (GitHub visibility of the repository and of branch chore/aplanar-bd_intivision) was not checked, because GitHub access is out of scope; local remote-tracking refs may also be stale.
- Whether the Caddyfile deployed on the intivision.online host (bind-mounted into guardian-caddy) still contains basic_auth with this hash is a deployment fact outside the repository.
- Only the credential owner knows whether the plaintext password is weak enough for offline guessing at bcrypt cost 14, or is reused for an application account, server SSH, GitHub or e-mail.

**Plan de validación:**

- *Despliegue (lo observa el dueño, sin pruebas en vivo):* Owner checks three facts without sending any traffic to the site: (1) on GitHub, whether AngelGabri3l/intivision is public or who has read access, and whether branch chore/aplanar-bd_intivision still exists; (2) on the production host, whether /etc/caddy/Caddyfile inside guardian-caddy (the host's ./Caddyfile) contains a basic_auth entry whose hash starts with $2a$14$M; (3) the credential holder privately confirms whether that password is used for any other account. If the repository is readable beyond the credential holders and either (2) or (3) is true, the exposure is live: rotate the password, keep the new hash out of git, delete the stale branch and consider rewriting history.

---

## 6. Backend port 8000 is published on all host interfaces, bypassing the Caddy TLS ingress that client_ip and cookie security rely on

- **Fingerprint:** `docker-compose.yml/backend-port-8000-published-bypasses-caddy-ingress`

docker-compose.yml publishes the uvicorn backend as "8000:8000" with no host IP, so Docker exposes it on every host interface over plain HTTP. Caddy (80/443) is the intended ingress and already reaches backend:8000 over the compose default network, so it does not need this publication. The backend's trust decisions assume all traffic arrives through Caddy. A client that connects to :8000 directly instead of through Caddy: (1) gets no TLS, so a /login password, the session cookie, /ingest bearer tokens and /ws/live frames sent that way cross the network in cleartext; (2) gets a session cookie without Secure and no HSTS unless it sends X-Forwarded-Proto: https itself; (3) chooses its own client IP whenever uvicorn sees its connection from a private address, because client_ip then returns the last X-Forwarded-For element. In case (3) the attacker can rotate that value to escape the 30-failures-per-15-minutes per-IP login throttle, or set it to a victim's IP so that the victim, who reaches the backend through Caddy with the same last-hop IP, is throttled at /login. The same forged value is stored in sessions.ip and in the view_camera audit_log ip. The per-account lockout in the database (auth.authenticate) is keyed by account, not IP, so rotating IPs does not allow unlimited guessing against one account. Uvicorn sees a private peer address for directly published traffic when docker-proxy handles the connection (for example IPv6 clients reaching an IPv4-only bridge network, or rootless Docker or Docker Desktop port drivers), or when the client is a host on the server's private network. With the default Linux iptables DNAT path for IPv4 the real public source address is preserved and the forged header is ignored. Caddy adds no authentication, rate or size control (Caddyfile lines 1-3), so no application authorization is bypassed by skipping it. Fix: remove the backend ports entry or bind it to 127.0.0.1:8000:8000, and trust X-Forwarded-For and X-Forwarded-Proto only from Caddy's address.

**Causa raíz propuesta:** docker-compose.yml publishes the backend port on all host interfaces even though Caddy reaches the backend internally. backend/auth.py treats every RFC1918, loopback or ULA peer as the trusted Caddy proxy when reading X-Forwarded-For, and cookie_is_secure and the HSTS header assume TLS terminates at Caddy.

**Traza:**

- `entrypoint` [docker-compose.yml:10](docker-compose.yml) (services.backend.ports): "8000:8000" publishes the backend on every host interface, next to Caddy's 80/443, so clients can reach uvicorn without going through Caddy.
- `propagation` [backend/Dockerfile:25](backend/Dockerfile) (CMD uvicorn backend.app:app): uvicorn binds 0.0.0.0:8000 in plain HTTP with no proxy-header flags, so request.client.host is whatever peer address Docker presents.
- `propagation` [backend/auth.py:239](backend/auth.py) (_REDES_DE_CONFIANZA): The trusted networks include 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 and fc00::/7, which cover Docker bridge gateways and private LANs, not only Caddy's container address.
- `propagation` [backend/auth.py:260](backend/auth.py) (client_ip): If the peer is private, the last X-Forwarded-For element is returned as the client IP (line 263). A direct client that reaches uvicorn from a private address therefore controls this value.
- `sink` [backend/app.py:440](backend/app.py) (login_submit): The client-controlled IP keys the per-IP login throttle (checked at 440, failure registered at 449) and is stored as the session IP (454).

**Evidencia:**

- Caddyfile:2: Caddy proxies to backend:8000 over the compose network and applies no other control, so the host publication of 8000 is not needed for the intended ingress.
- backend/auth.py:257: The client_ip docstring states that an Internet client writing X-Forwarded-For by hand cannot pose as another IP. That holds only if no internet client reaches uvicorn from a private address.
- backend/auth.py:203: cookie_is_secure trusts the request scheme or X-Forwarded-Proto from any peer. A direct HTTP login without that header gets a session cookie without Secure (set_session_cookie, line 209).
- backend/app.py:157: Strict-Transport-Security is emitted only when cookie_is_secure is true, so direct-port responses never carry HSTS.
- backend/app.py:731: The audit_log ip recorded when a community_admin views another user's private camera comes from client_ip, so a direct-port client seen from a private address can forge it.
- backend/auth.py:283: throttled() blocks an IP after LOGIN_IP_MAX_FAILURES (default 30, line 34) failures in the window, keyed only by the client_ip string.
- docker-compose.yml:57: Caddy publishes 80 and 443, the intended TLS ingress for intivision.online.
- backend/templates/index.html:712: The UI mentions http://localhost:8000 only as a development path; nothing in the repository directs production users or edge devices to port 8000.

**Bloqueos (lo que falta saber):**

- The host firewall or cloud security-group state for TCP 8000 (IPv4 and IPv6) on the intivision.online host is not in the repository, so source cannot show whether the published port is reachable from the internet.
- The peer address uvicorn sees for published-port traffic depends on the Docker daemon setup (userland-proxy, iptables/ip6tables, IPv6 clients reaching an IPv4-only network, rootless or Desktop port drivers) and on which hosts share the server's private network. Only a private peer address lets client_ip accept attacker-supplied X-Forwarded-For.
- Whether any user or edge mini PC is configured to use http://<host>:8000 is a deployment fact; in-repo clients default to localhost or https://intivision.online.
- No OS-enforced sandbox was available, so the compose stack could not be run locally to observe the peer address or the Set-Cookie flags.

**Plan de validación:**

- *Local (en sandbox, datos ficticios):* On an isolated Docker host with external networking disabled, start only the backend service against a throwaway local Postgres with a dummy DATABASE_URL and one dummy user. From a separate network namespace, send one POST /login with dummy wrong credentials and the header 'X-Forwarded-For: 203.0.113.7' to the published port over IPv4 and, if the daemon has IPv6 enabled, over IPv6. Read the 'Acceso fallido desde ...' log line to see whether 203.0.113.7 (header trusted) or the real client address was recorded. Then perform one dummy successful login over the direct port and check that Set-Cookie for vigia_session lacks Secure and the response lacks Strict-Transport-Security.
- *Despliegue (lo observa el dueño, sin pruebas en vivo):* Owner, on the intivision.online host: run 'docker port guardian-backend' and 'ss -ltnp' to list the 8000 listeners, read /etc/docker/daemon.json (userland-proxy, ip6tables, rootless), and review host firewall and cloud security-group rules for TCP 8000 on IPv4 and IPv6. If 8000 is open externally, from an owner-controlled external machine send one GET /health to the server IP on :8000 and one failed login carrying 'X-Forwarded-For: 203.0.113.7', then read the guardian-backend log to see which IP was recorded.

---

## 7. client_ip trusts client-supplied X-Forwarded-For from any private or loopback peer, not only from Caddy

- **Fingerprint:** `backend/auth.py:client_ip:xff-trusted-from-any-private-peer`

client_ip supplies the key for the per-IP login throttle and the address stored in sessions.ip and in the admin view_camera audit entry. It returns the last X-Forwarded-For element whenever the TCP peer is in 127/8, 10/8, 172.16/12, 192.168/16, ::1 or fc00::/7. Through Caddy this is correct, because the last element is always the one Caddy sets from the real client. However, docker-compose.yml also publishes the backend directly on all host interfaces at port 8000. Docker may present a client of that port as a private gateway address, for example with userland-proxy handling of IPv6 clients when ip6tables is disabled, with rootless Docker, or with Docker Desktop. A client on the same private network (for example a cloud VPC) is also a private peer. In those cases the client's own X-Forwarded-For value is trusted. Each failed login can then claim a new address to escape the 30-failures-per-15-minutes throttle, leaving only the per-account lockout against password spraying. An attacker can also attribute failures to a victim's real address to block logins from it through Caddy, and can forge the IP recorded in sessions and audit_log.

**Causa raíz propuesta:** backend/auth.py client_ip decides whether to trust the proxy header by membership in broad private ranges (_REDES_DE_CONFIANZA) instead of the specific Caddy proxy address. Any peer that appears private, including a direct client of the published port when Docker rewrites the source address, therefore supplies the address used by security controls.

**Traza:**

- `entrypoint` [backend/app.py:438](backend/app.py) (login_submit): Anonymous POST /login with an arbitrary X-Forwarded-For header, reaching the backend directly on published port 8000.
- `propagation` [backend/app.py:439](backend/app.py) (login_submit): Client address derived with auth.client_ip(request).
- `propagation` [backend/auth.py:260](backend/auth.py) (client_ip): The peer address is checked only for membership in the private and loopback ranges defined at backend/auth.py:239-240.
- `propagation` [backend/auth.py:263](backend/auth.py) (client_ip): The last comma-separated X-Forwarded-For element is taken and returned if it parses as an IP address.
- `sink` [backend/app.py:440](backend/app.py) (login_submit): The per-IP throttle decision is keyed on the header-supplied address, and failures are recorded under it at backend/app.py:449.

**Evidencia:**

- docker-compose.yml:10: Backend port 8000 is published on all host interfaces in addition to the Caddy ingress.
- Caddyfile:2: Caddy's reverse_proxy to backend:8000 is the only intended forwarding proxy.
- backend/auth.py:239: Trusted networks are all RFC1918, loopback and ULA ranges rather than the proxy's address.
- backend/auth.py:283: The throttle compares the failure count for the derived address against LOGIN_IP_MAX_FAILURES.
- backend/app.py:454: create_session stores the derived address in sessions.ip.
- backend/app.py:731: The admin view_camera audit entry records auth.client_ip(request).

**Bloqueos (lo que falta saber):**

- Source does not show which address the backend container sees for a client connecting directly to published port 8000; this is a Docker networking fact. With iptables DNAT the public IPv4 address is preserved and the header is ignored. With userland-proxy handling (for example IPv6 clients without ip6tables), rootless Docker or Docker Desktop, the peer appears as a private gateway address and the header is trusted.
- Whether port 8000 is reachable from the Internet or from a shared private network at all (host or cloud firewall) is a deployment fact that was not observed.
- Target code was not executed, so the throttle bypass was not observed locally.

**Plan de validación:**

- *Local (en sandbox, datos ficticios):* In a sandbox, build a Starlette Request whose scope client is ('172.18.0.1', 40000) with header X-Forwarded-For: 203.0.113.7, and confirm that auth.client_ip returns '203.0.113.7'. Call register_failure 30 times for that address, then confirm throttled('203.0.113.7') is True and throttled('203.0.113.8') is False. Regression check: trust X-Forwarded-For only when the peer equals the configured Caddy address, or stop publishing port 8000.
- *Despliegue (lo observa el dueño, sin pruebas en vivo):* Without sending any test traffic, the owner checks on the production host: (1) whether port 8000 is open to the Internet or to other tenants of the private network (host firewall rules and the cloud security group); (2) how Docker publishes it (docker info for rootless mode, /etc/docker/daemon.json for userland-proxy and ip6tables, and whether the host has a public IPv6 address); (3) in the existing database, SELECT ip, count(*) FROM sessions GROUP BY ip, and in existing backend logs the 'Acceso fallido desde' lines, looking for Docker gateway addresses (for example 172.x.0.1), which would show that direct clients already appear as private peers.

---

## 8. Open /ws/live sockets keep working after logout, suspension, password reset or session expiry

- **Fingerprint:** `backend/app.py:ws_live:session-not-rechecked-after-accept`

ws_live validates the vigia_session cookie once, right after the handshake. It then loops indefinitely, processing frames with the user and camera captured at connect time. HTTP routes re-check the session on every request (revoked_at, expires_at and users.status). backend/manage.py set-status and set-password revoke all session rows so that a suspended or reset account loses its sessions, but nothing closes or re-validates sockets that are already open. A suspended resident keeps a working socket. So does someone using a stolen cookie whose sessions were revoked by a password reset. Each frame is still analysed. MEDIO/ALTO results are inserted as events attributed to that user and camera in the community's records, and dispatch_live_alert still drives the shared MQTT siren and the Telegram channel (subject only to the global per-level cooldown). Socket lifetime is also not tied to the 12-hour session expiry. No data from other users is disclosed over the socket.

**Causa raíz propuesta:** backend/app.py ws_live calls auth.user_from_websocket and cameras.resolve_camera only once, before the receive loop. The loop over receive_bytes and _process_frame never re-checks the session token, account status, expiry or camera binding, and the server keeps no registry of open sockets to close when sessions are revoked.

**Traza:**

- `entrypoint` [backend/app.py:538](backend/app.py) (ws_live): WebSocket /ws/live?camera= upgrade carrying the vigia_session cookie.
- `propagation` [backend/app.py:546](backend/app.py) (ws_live): Session resolved once via auth.user_from_websocket; revocation, expiry and status are checked only at this moment.
- `propagation` [backend/app.py:554](backend/app.py) (ws_live): Camera binding (owner, private, active) resolved once.
- `propagation` [backend/app.py:567](backend/app.py) (ws_live): Inside the while True loop, frames are received indefinitely with no session, status or expiry re-check.
- `propagation` [backend/app.py:569](backend/app.py) (ws_live): Each frame is processed with the user and camera captured at connect time.
- `propagation` [backend/app.py:533](backend/app.py) (_process_frame): insert_event writes an event and evidence media row attributed to the captured user, community and camera.
- `sink` [backend/app.py:534](backend/app.py) (_process_frame): dispatch_live_alert triggers the shared siren (ALTO) or warning (MEDIO) and sends a Telegram photo alert.

**Evidencia:**

- backend/auth.py:158: The HTTP session lookup requires revoked_at IS NULL, expires_at > now() and status 'active' on every request; the WebSocket path runs it only once.
- backend/manage.py:184: set-status documents that a suspended account cannot log in and loses its sessions; line 189 only updates session rows.
- backend/manage.py:178: set-password revokes all session rows to close open sessions; open sockets are unaffected.
- backend/app.py:463: Logout only revokes the session row; no socket is closed.
- backend/app.py:375: dispatch_live_alert triggers the siren for ALTO events, limited only by a global per-level cooldown (line 370).
- engine.py:146: LiveSession.process emits an event for MEDIO/ALTO frames, so continued frames keep producing events and evidence images.
- backend/Dockerfile:25: uvicorn runs with no WebSocket timeout or lifetime options, and the Caddyfile reverse_proxy (Caddyfile:2) sets no stream timeout, so source does not bound socket lifetime.

**Bloqueos (lo que falta saber):**

- Target code was not executed (no sandbox, PostgreSQL or YOLO models), so it was not observed that a socket opened before revocation keeps receiving, analysing and inserting frames after the session row is revoked or the user is suspended.
- How long such a socket can stay open in deployment depends on uvicorn/websockets keepalive defaults and Caddy's WebSocket proxy behaviour (for example closing streams on config reload). Source does not configure these and they were not observed.

**Plan de validación:**

- *Local (en sandbox, datos ficticios):* In a sandbox with a disposable local PostgreSQL (schema_vigia.sql) and one dummy resident with a private camera, use Starlette TestClient against backend.app. Patch get_engine to return a stub whose new_session().process returns a fixed ALTO event, patch cv2.imdecode to return a small array, and patch get_dispatcher to a recording mock. Log in, open /ws/live?camera=<id> and send one frame. Then run UPDATE users SET status='suspended' and UPDATE sessions SET revoked_at=now() for that user, as manage.py set-status does. Wait past the ALTO cooldown, send a second frame on the same socket, and count events rows and mock actuator.trigger calls. The claim holds if the second frame still inserts an event and calls the mock. Regression check: re-validate the token, status, expiry and camera per frame (or on a short interval) and close the socket with 4401.

---

## 9. Global per-level live alert cooldown lets one user's camera suppress other users' siren and Telegram alerts

- **Fingerprint:** `backend/app.py:dispatch_live_alert:global-per-level-cooldown-cross-tenant-suppression`

dispatch_live_alert keeps one module-level _last_alert dict keyed only by level ('MEDIO' or 'ALTO'). It returns early whenever the same level was dispatched anywhere in the process in the last 20 s (ALTO) or 45 s (MEDIO). The backend runs as a single uvicorn process (backend/Dockerfile:25), so every live session of every user and community shares this one budget. A resident fully controls the frames sent over /ws/live for their own private camera, and each LiveSession can emit an event every 5 s (config.yaml:42). With the committed config (no restricted zone, daytime), person (1) + loitering for 15 s of wall-clock time (2) + hands raised (3) scores 6 = ALTO, and person + loitering scores 3 = MEDIO. A resident who keeps re-sending such frames (for example, standing in front of their webcam with raised hands, using a second session for MEDIO) keeps both windows occupied. While they are, live MEDIO/ALTO events from every other user's camera are still stored in the database, but they produce no actuator command of their own and no Telegram photo. The chat receives only the attacker's evidence, so the people reading it never see the victims' frames. The siren is shared, so the attacker's own ALTO commands still sound it 10 s out of every 20 s; the victim-specific loss is mainly the Telegram evidence and any siren activation that falls in the off part of the window. Without an attacker, ordinary concurrent activity has the same effect, which on its own is rate limiting of a shared sink; the security question is a malicious resident using it to hide other residents' alerts. Not observed: no target code was executed.

**Causa raíz propuesta:** The live alert cooldown in backend/app.py (LEVEL_COOLDOWN and _last_alert, lines 359-372) is keyed by level alone, not by camera, owner or community. The rate-limit budget for the shared alert sinks can therefore be used up by any one principal, and other principals' live alerts are dropped while it is.

**Traza:**

- `entrypoint` [backend/app.py:539](backend/app.py) (ws_live): An authenticated resident (any community) opens /ws/live for their own private camera and streams frames they choose.
- `propagation` [backend/app.py:569](backend/app.py) (ws_live): Each received frame is passed to _process_frame in a worker thread.
- `propagation` [engine.py:146](engine.py) (LiveSession.process): A MEDIO/ALTO result yields an event at most every event_cooldown_s (5 s) per session; the session's own scorer measures loitering with time.time() (line 129).
- `propagation` [backend/app.py:534](backend/app.py) (_process_frame): The attacker's event is stored and passed to dispatch_live_alert.
- `propagation` [backend/app.py:372](backend/app.py) (dispatch_live_alert): _last_alert[level] is set process-wide by the attacker's event.
- `sink` [backend/app.py:370](backend/app.py) (dispatch_live_alert): A later live event of the same level from any other user's camera matches the same key and returns at line 371, before the actuator call (lines 374-377) and the Telegram send_photo (line 381).

**Evidencia:**

- backend/app.py:359: LEVEL_COOLDOWN = {'MEDIO': 45, 'ALTO': 20}, keyed by level only.
- backend/app.py:360: _last_alert is one module-level dict shared by all sessions, users and communities.
- backend/app.py:364: Docstring: the per-level cooldown exists so the external channels are not saturated.
- backend/Dockerfile:25: uvicorn is started without --workers, so one process (and one _last_alert) serves every user.
- config.yaml:42: event_cooldown_s: 5 allows one live event every 5 s per session.
- threat_scoring.py:82: hands_up adds 3; with person_present (1, line 64) and loitering (2, line 71) the score is 6, which meets the high threshold (config.yaml line 25).
- bd_intivision/schema_vigia.sql:237: notification_channels has a per-channel min_level, showing recipient-scoped alerting was intended; no Python code uses this table.

**Bloqueos (lo que falta saber):**

- No target code may be executed in this run (no OS-enforced sandbox), so the suppression of a second user's alert was not observed, even with a mocked dispatcher.
- Whether the deployment has more than one camera owner streaming live to the same backend process, so that one resident can suppress another resident's alerts, is a deployment fact not visible in source.

**Plan de validación:**

- *Local (en sandbox, datos ficticios):* In a sandbox with no network: replace backend.app.get_dispatcher with a stub returning enabled=True and recording actuator (trigger/warn) and notifier (send_photo) calls; patch backend.app.time.time to return controlled values; clear backend.app._last_alert. Call dispatch_live_alert with ALTO events tagged as camera A at t=0, 19 and 38, and with an ALTO event tagged as camera B (different owner and community) at t=10 and t=30. Expected if the claim holds: three trigger and three send_photo calls, all for camera A's images, none for camera B's.
- *Despliegue (lo observa el dueño, sin pruebas en vivo):* Owner checks, from the database and without generating traffic, whether more than one user or community has cameras with live sessions on this backend (for example, distinct owners in cameras.last_seen within the same period). If only one owner ever streams live, the shared cooldown only rate-limits that owner's own alerts.

---

## 10. Uploaded-video alerts bypass the live alert cooldown, so one resident can flood the shared siren and Telegram chat

- **Fingerprint:** `backend/app.py:dispatch_alerts:upload-alerts-bypass-cooldown`

dispatch_alerts, called from POST /analyze, publishes one actuator command (ON:10 for ALTO, WARN for MEDIO) for every MEDIO/ALTO event found in an uploaded video and sends one Telegram photo per upload. It applies no cooldown and no per-user or per-community limit, and it never consults LEVEL_COOLDOWN/_last_alert, which the live path uses so the shared channels are not saturated. Any resident or community_admin can upload video (up to MAX_UPLOAD_MB=200) against their own private camera. The event count is limited only by video time divided by 5 s, and video time is the frame index divided by the fps the container declares, which the uploader controls. One upload with many ALTO events therefore sends a burst of ON commands 0.5 s apart; each resets the ESP32 offAt, keeping the neighbourhood siren on for the whole dispatch. Repeated small uploads (about 15 frames of a person with raised hands is enough for MEDIO) each post an uploader-chosen frame to the shared chat as 'ALERTA DE SEGURIDAD' for the configured zone, at the upload rate instead of at most one ALTO per 20 s and one MEDIO per 45 s. If such bursts trigger Telegram per-chat throttling, other residents' alerts sent in that period fail and are only logged. The blocking of the event loop by dispatch_alerts' sleeps and publish waits is part of the separate root cause backend/app.py:analyze:blocking-video-analysis-on-event-loop and is not claimed here. Not observed: no target code was executed and Telegram was not contacted.

**Causa raíz propuesta:** The upload alert path (backend/app.py dispatch_alerts, lines 330-356) sends to the same shared actuator and Telegram sinks as the live path but has no rate control or deduplication across events or uploads, so it bypasses the anti-flood limit enforced on its live sibling (backend/app.py:359-372).

**Traza:**

- `entrypoint` [backend/app.py:577](backend/app.py) (analyze): An authenticated resident uploads an arbitrary video for their own private camera.
- `propagation` [backend/app.py:603](backend/app.py) (analyze): analyze_video returns every MEDIO/ALTO event found in the video.
- `propagation` [engine.py:85](engine.py) (GuardianEngine.analyze_video): At most one event per 5 s of video time, where video time = idx / container fps (lines 62, 76).
- `propagation` [backend/app.py:609](backend/app.py) (analyze): dispatch_alerts is called for every upload with all its events.
- `propagation` [backend/app.py:344](backend/app.py) (dispatch_alerts): Loops over every event with no cooldown, count limit or check of _last_alert.
- `propagation` [backend/app.py:348](backend/app.py) (dispatch_alerts): One siren ON command per ALTO event (one WARN per MEDIO event at line 350), sent to the shared topic via actuator.py:57.
- `sink` [backend/app.py:356](backend/app.py) (dispatch_alerts): One Telegram photo of the uploader-chosen frame per upload, independent of LEVEL_COOLDOWN.

**Evidencia:**

- backend/app.py:333: Docstring: one MQTT order per event, intended for viewing in Wokwi.
- backend/app.py:334: Docstring: only one Telegram per video 'to avoid filling the chat', so chat flooding is an intended concern, but nothing limits it across uploads.
- backend/app.py:359: LEVEL_COOLDOWN applies only to the live path (dispatch_live_alert).
- backend/app.py:33: Uploads accepted up to MAX_UPLOAD_MB=200.
- engine.py:62: fps comes from the uploaded container's metadata, so the uploader controls the video-time scale.
- notifier.py:65: A Telegram rejection (for example, rate limiting) is only logged; there is no retry.
- wokwi/sketch.ino:58: Each ON command resets offAt, so back-to-back commands keep the siren on continuously.

**Bloqueos (lo que falta saber):**

- No target code may be executed in this run (no OS-enforced sandbox), so the per-upload command count and per-upload Telegram sends were not observed, even with a mocked dispatcher.
- Whether bursts of uploads cause Telegram to throttle the configured chat and drop other residents' alerts is a third-party runtime fact; contacting Telegram is prohibited.

**Plan de validación:**

- *Local (en sandbox, datos ficticios):* In a sandbox with no network: replace backend.app.get_dispatcher with a stub returning enabled=True and recording actuator.trigger/warn and notifier.send_photo calls, and patch backend.app.time.sleep to a no-op. Call dispatch_alerts once with 50 synthetic ALTO events (video_time_s, level, score 6, reasons, image) and record 50 trigger calls and 1 send_photo call. Then call it 10 times with one MEDIO event each and record 10 send_photo calls. For comparison, call dispatch_live_alert 10 times with MEDIO events and backend.app.time.time patched to advance 1 s per call: expect 1 send_photo call.
- *Despliegue (lo observa el dueño, sin pruebas en vivo):* Owner checks the type of the configured TELEGRAM_CHAT_ID (group shared by several residents or communities, or a private chat) and reviews existing backend logs, without generating new uploads, for 'Telegram rechazó la foto' entries containing error_code 429 near bursts of /analyze requests.

---

## 11. Uploaded-video threat scoring reuses one ThreatScorer across users, so one upload can lower the next user's event levels

- **Fingerprint:** `engine.py:GuardianEngine.analyze_video:shared-scorer-state-across-uploads`

GuardianEngine is a process-wide singleton (backend/app.py:57-62, one uvicorn process per backend/Dockerfile:25). analyze_video scores every uploaded video with the single self.scorer created in __init__ and never resets it, whereas each live session gets its own ThreatScorer ('scorer propio por cliente'). ThreatScorer keeps a persistence history deque and presence_start between evaluate() calls. The analyze handler runs analyze_video synchronously with no await inside it, so uploads are analysed strictly one after another, and each video starts with the exact scorer state the previous video left. If resident A's video ends with a person present who first appeared at video time P (for example, P = 33 s in a 60 s video, or much larger with a low declared fps), presence_start stays P. In the next analysed video, from any user in any community, a person present from the first sampled frame gets loiter_time = t - P, which is negative or below 15 s until t >= P + 15, so the 2 loitering points are lost until the person is absent in a sampled frame. Outside the sensitive hours (22:00-06:00 at analysis time) and with no restricted zone (committed config), person + loitering (3, MEDIO) becomes 1 (BAJO): no event is stored and no WARN or Telegram is sent. Person + loitering + hands_up (6, ALTO) becomes 4 (MEDIO): WARN instead of the siren. The history carried from A also makes B's first frames immediately 'stable', which has no comparable effect. The attacker cannot see others' uploads, so they would need to upload repeatedly to be the preceding analysis. Only uploaded-video analysis is affected, not live streams. Not observed: no target code was executed.

**Causa raíz propuesta:** engine.py analyze_video (line 83) uses the engine-level self.scorer (line 46), whose temporal state (history and presence_start, threat_scoring.py:28-29 and 53-59) persists across analyses of different users' videos, instead of a fresh ThreatScorer per analysis as LiveSession does (engine.py:122).

**Traza:**

- `entrypoint` [backend/app.py:577](backend/app.py) (analyze): Resident A uploads a video ending with a person present; resident B (any community) uploads next.
- `propagation` [backend/app.py:603](backend/app.py) (analyze): Both uploads are analysed synchronously, one after the other, by the singleton engine from get_engine (lines 57-62).
- `propagation` [engine.py:46](engine.py) (GuardianEngine.__init__): self.scorer is created once per process.
- `propagation` [engine.py:83](engine.py) (GuardianEngine.analyze_video): self.scorer.evaluate is called for every video, with no reset between analyses.
- `propagation` [threat_scoring.py:54](threat_scoring.py) (ThreatScorer.evaluate): presence_start is set only when None, so A's value P is kept while B's person is present from the first sampled frame.
- `propagation` [threat_scoring.py:70](threat_scoring.py) (ThreatScorer.evaluate): loiter_time = t - P (line 56) stays below loitering_seconds until t >= P + 15, so no loitering points.
- `sink` [engine.py:85](engine.py) (GuardianEngine.analyze_video): B's event is created, or not, from the lowered level; with no event, nothing is stored or dispatched for that frame.

**Evidencia:**

- engine.py:122: LiveSession creates its own ThreatScorer per client, showing per-client state was intended.
- engine.py:110: Docstring: 'scorer propio por cliente' (one scorer per client).
- engine.py:62: fps comes from the uploaded container, so the uploader also controls the video-time scale of P.
- threat_scoring.py:28: The persistence history deque survives across analyze_video calls.
- backend/app.py:61: A single GuardianEngine serves all users.
- config.yaml:24: medium threshold 3 and high threshold 6 (line 25), with loitering worth 2 (line 17) and loitering_seconds 15 (line 26).

**Bloqueos (lo que falta saber):**

- No target code may be executed in this run (no OS-enforced sandbox), so the lowered level for the second video was not observed, even with a stubbed detector.
- Whether real victim footage keeps a person detected in every sampled frame long enough for the carried presence_start to matter depends on YOLO detections, which were not run.

**Plan de validación:**

- *Local (en sandbox, datos ficticios):* In a sandbox with no network: before constructing the engine, replace engine.PersonDetector with a stub whose detect(frame) returns [{'bbox': [10, 10, 50, 100], 'conf': 0.9}] when frame.mean() > 128 and [] otherwise; load config.yaml with pose.enabled set to false; patch engine.datetime so now() returns 12:00. Write with cv2.VideoWriter at 1 fps: video A, 60 frames 64x64, black for frames 1-30 and white for 31-60; video B, 30 white frames. On a fresh engine, analyze_video(B) should return MEDIO events (expected at video times 18, 24 and 30 s). On another fresh engine, run analyze_video(A) and then analyze_video(B): if the claim holds, B returns no events because presence_start stays at 33 s.

---

## 12. /analyze runs whole-video YOLO analysis synchronously on the backend event loop, which can stall the service for every community

- **Fingerprint:** `backend/app.py:analyze:blocking-video-analysis-on-event-loop`

The analyze handler is declared async def, but it calls get_engine().analyze_video(...), then insert_event for each event and dispatch_alerts, all synchronously and without asyncio.to_thread or an executor. analyze_video decodes every frame of the upload and runs YOLO (and pose when people are detected) on every sample_every-th frame, with no frame, duration or time cap. dispatch_alerts sleeps 0.5 s per event and can wait up to 5 s per MQTT publish and 10 s per Telegram request. The shipped backend image starts uvicorn with a single worker. Source therefore indicates that while one resident's upload is analysed, the single event loop cannot advance other users' async work: WebSocket live frames, /ingest, the /analyze session check and every middleware pass, which all route through the asyncio loop. Analysis time grows with the frame count of an upload of up to 200 MB, which the uploader controls. The length and breadth of the stall were not observed because no target code could be executed in this run.

**Causa raíz propuesta:** CPU-bound and network-bound blocking work (whole-video decode and YOLO inference, per-event DB inserts, MQTT and Telegram dispatch with sleeps) runs inline in an async FastAPI handler on the only event loop, with no worker offload, concurrency limit or frame/time budget.

**Traza:**

- `entrypoint` [backend/app.py:577](backend/app.py) (analyze): POST /analyze handler declared async def, so its body runs on the event loop thread; any authenticated resident or community_admin can call it.
- `propagation` [backend/app.py:603](backend/app.py) (analyze): get_engine().analyze_video(tmp_path, EVIDENCE_DIR, source_label=label) is called synchronously, with no asyncio.to_thread or run_in_executor.
- `propagation` [engine.py:69](engine.py) (GuardianEngine.analyze_video): while True: cap.read() iterates over every decoded frame with no frame-count or time cap.
- `sink` [engine.py:78](engine.py) (GuardianEngine.analyze_video): self.detector.detect(frame) runs YOLO inference for each sampled frame on the calling thread, which is the event loop thread.

**Evidencia:**

- backend/app.py:569: ws_live offloads per-frame inference with asyncio.to_thread, showing the intended pattern that /analyze does not follow.
- backend/app.py:608: insert_event (synchronous psycopg2 work) runs on the event loop for every detected event.
- backend/app.py:351: dispatch_alerts calls time.sleep(0.5) per event, on the event loop when called from analyze at line 609.
- actuator.py:59: MQTTActuator._send waits up to 5 s per siren/LED command (wait_for_publish(timeout=5)); with the tracked config.yaml, actuator.type is mqtt.
- notifier.py:61: The Telegram sendPhoto request in dispatch_alerts can block for up to 10 s when Telegram credentials are configured.
- backend/Dockerfile:25: uvicorn starts without --workers, so one process and one event loop serve every community; docker-compose.yml does not override the command.
- backend/app.py:33: MAX_UPLOAD_MB bounds bytes only; no frame, duration or time limit exists on the analysis path.
- engine.py:74: Inference is skipped only for frames that are not multiples of sample_every (3 in config.yaml), so inference count scales linearly with frame count.

**Bloqueos (lo que falta saber):**

- The claimed effect is a runtime event-loop stall; no target code could be executed in this run (no OS-enforced sandbox), so the stall was not observed and its duration for a given upload is unmeasured.
- The number of uvicorn worker processes actually running in the deployment is a deployment fact; source shows a single worker in backend/Dockerfile but an operator could run a different command.

**Plan de validación:**

- *Local (en sandbox, datos ficticios):* In a sandbox with CPU, memory and wall-clock limits and no external network, run the backend against a disposable PostgreSQL with one dummy resident, the shipped single-worker uvicorn command, actuator.type set to log and Telegram unset. Replace GuardianEngine.analyze_video with a stub that calls time.sleep(10) and returns an empty list (or use a 60-second synthetic MP4 with the real models). Start one POST /analyze as the dummy resident with a small MP4, and 1 s later send POST /ingest with a dummy edge token (or open /ws/live as a second dummy resident and send one JPEG). Record the second request's latency; stop once it is shown to wait for the analysis to finish.
- *Despliegue (lo observa el dueño, sin pruebas en vivo):* The owner checks the command the backend container actually runs (docker inspect of the container's Cmd/Args) to confirm whether uvicorn uses a single worker.

---

## 13. One resident can open unlimited /ws/live sessions and starve other tenants' live detection on the shared inference lock and default thread pool

- **Fingerprint:** `backend/app.py:ws_live:unbounded-concurrent-inference-sessions`

/ws/live accepts any number of concurrent connections per user and per camera, and processes every binary frame the client sends; the only pacing (one frame in flight) is in the dashboard JavaScript, which a scripted client need not follow. Each frame is handed to asyncio.to_thread, which uses the event loop's default executor (bounded by min(32, CPU count + 4) threads), and then waits on the single process-wide engine.infer_lock. Because each connection keeps one to_thread call in flight, a resident who opens enough connections and sends small valid JPEGs back to back can keep most default-executor threads busy waiting on infer_lock. Other residents' live sessions in any community then get a fraction of inference throughput, and other asyncio.to_thread work queues behind it: /ingest token lookups and inserts, WebSocket authentication and camera resolution, and the /analyze session check. Synchronous def routes run in anyio's separate thread pool and are not affected by the default-executor queue. The degree of starvation is a runtime property that was not observed.

**Causa raíz propuesta:** The server has no per-user or per-camera limit on concurrent live sessions or frame rate and no fair scheduling for the shared inference lock; pacing is delegated to the browser client.

**Traza:**

- `entrypoint` [backend/app.py:539](backend/app.py) (ws_live): Authenticated WebSocket handler; nothing counts or limits concurrent connections per user or camera.
- `propagation` [backend/app.py:554](backend/app.py) (ws_live): resolve_camera checks only that the camera is the user's own active private camera; it keeps no session accounting.
- `propagation` [backend/app.py:567](backend/app.py) (ws_live): receive_bytes loop with no rate, count or application-level size limit.
- `propagation` [backend/app.py:569](backend/app.py) (ws_live): Each frame is submitted with asyncio.to_thread to the shared default executor.
- `propagation` [backend/app.py:528](backend/app.py) (_process_frame): cv2.imdecode decodes client bytes in the worker thread before the lock.
- `sink` [engine.py:133](engine.py) (LiveSession.process): with eng.infer_lock: detect and pose run under one process-wide lock shared by every user and community.

**Evidencia:**

- engine.py:48: A single threading.Lock (no fairness guarantee) guards the shared YOLO models for all clients.
- backend/templates/index.html:761: The comment states one frame is sent and the reply awaited so the model never saturates; this pacing exists only in the client.
- backend/cameras.py:160: resolve_camera's query matches an own, active, private camera and records nothing about open connections.
- backend/app.py:623: /ingest's edge token lookup uses asyncio.to_thread, sharing the same default executor.
- backend/app.py:476: The limitar_subidas session lookup for /analyze uses asyncio.to_thread, sharing the same default executor.
- backend/Dockerfile:25: A single uvicorn process hosts every tenant's live sessions.
- Caddyfile:2: The repository reverse_proxy block sets no limit on concurrent WebSocket connections per client.

**Bloqueos (lo que falta saber):**

- The default executor size depends on the deployed host's CPU count, and per-frame inference latency depends on host CPU and torch threading; neither is visible in source, and nothing could be executed in this run.
- How unevenly threading.Lock acquisitions are distributed under contention, and therefore how much a victim session's throughput falls, is runtime behaviour that was not observed.
- Whether any layer in front of the backend (host firewall, CDN, or a different proxy configuration) limits concurrent WebSocket connections per client or user is a deployment fact; the repository Caddyfile sets none and port 8000 is published directly.

**Plan de validación:**

- *Local (en sandbox, datos ficticios):* In a sandbox with CPU, memory and wall-clock limits and no external network, run the backend against a disposable PostgreSQL with two dummy residents in two dummy communities, replacing detector.detect with a stub that sleeps 50 ms under infer_lock. Open one victim /ws/live connection that sends one small JPEG per reply, then N = 4, 8 and 16 attacker connections from the other dummy resident that send 1 KB valid JPEGs back to back. Record victim frames per second and the latency of one concurrent POST /ingest with a dummy edge token at each N. Stop once victim throughput clearly falls with N.
- *Despliegue (lo observa el dueño, sin pruebas en vivo):* The owner checks the backend host's CPU count, whether any proxy or firewall in front of intivision.online or port 8000 limits concurrent WebSocket connections per client IP or user, and the expected number of simultaneous live users.

---

## 14. Upload and ingest size limits rely only on the declared Content-Length, so chunked bodies are received in full before any limit applies

- **Fingerprint:** `backend/app.py:content-length-only-request-size-gates`

limitar_subidas is documented as rejecting oversized /analyze uploads before they are received, but upload_too_big reads only the Content-Length header and treats a missing header as '0'. A request sent with Transfer-Encoding: chunked carries no Content-Length and passes. The analyze handler takes an UploadFile parameter, so FastAPI parses the whole multipart body, spooling the file part to a temporary file, before the handler's own 200 MB counter runs; that counter only limits the second copy into UPLOAD_DIR. The repository Caddyfile sets no request body limit, and backend port 8000 is also published directly. An authenticated resident can therefore stream an arbitrarily large body into the container's temporary storage on the shared host disk for as long as the request lasts. /ingest has the same pattern for edge-device token holders: its 100 kB check reads only Content-Length and request.json() buffers the full body in memory. Whether the installed Starlette/python-multipart versions impose their own file-part limit, and what disk backs the spool, are not visible in source.

**Causa raíz propuesta:** Request body size is enforced from a client-declared header instead of counting bytes actually received during parsing, and neither the ASGI layer nor the repository proxy configuration applies a streaming body limit.

**Traza:**

- `entrypoint` [backend/app.py:472](backend/app.py) (limitar_subidas): HTTP middleware for POST /analyze: session check, then size check.
- `propagation` [backend/app.py:482](backend/app.py) (limitar_subidas): upload_too_big(request) is the only pre-receipt size control.
- `propagation` [backend/app.py:183](backend/app.py) (upload_too_big): int(request.headers.get('content-length', '0')): a chunked request without Content-Length is treated as 0 bytes.
- `propagation` [backend/app.py:485](backend/app.py) (limitar_subidas): call_next forwards the request; FastAPI parses the full multipart form to build the UploadFile before invoking the handler.
- `sink` [backend/app.py:577](backend/app.py) (analyze): The video: UploadFile parameter has already been spooled in full by the framework, with no application size limit, before the handler body runs.

**Evidencia:**

- backend/app.py:181: The docstring states the intent to reject huge uploads before receiving them, using Content-Length.
- backend/app.py:598: The handler's byte counter applies only while copying out of the already-parsed UploadFile.
- backend/app.py:627: /ingest checks size through the Content-Length header only.
- backend/app.py:632: request.json() reads the entire /ingest body into memory.
- Caddyfile:2: reverse_proxy has no request_body max_size.
- docker-compose.yml:10: Backend port 8000 is published directly, bypassing the proxy.
- requirements.txt:8: fastapi>=0.110.0 is unpinned (as is python-multipart at line 10), so the installed multipart spooling behaviour depends on the build.

**Bloqueos (lo que falta saber):**

- Starlette and python-multipart versions in the built image are unpinned; whether they impose any size limit on multipart file parts, and where they spool them, must be confirmed for the built image without executing it here.
- Whether Caddy and any host or CDN in front of it forward chunked request bodies to the backend without a bound is a deployment fact.
- The location of the container's temporary directory, its free space, and whether that filesystem is shared with ./events and PostgreSQL are deployment facts.

**Plan de validación:**

- *Local (en sandbox, datos ficticios):* In a sandbox with no external network, CPU/memory/time limits, TMPDIR on a 64 MB size-limited tmpfs and MAX_UPLOAD_MB=1, run the backend against a disposable PostgreSQL with a dummy resident session. POST /analyze with Transfer-Encoding: chunked and a 5 MB multipart file part; observe that limitar_subidas does not return 413 and that the spool file in TMPDIR grows past 1 MB before the handler returns its own 413. Repeat for /ingest with a dummy edge token and a 1 MB chunked JSON body, observing process RSS. Stop at the first exceedance.
- *Despliegue (lo observa el dueño, sin pruebas en vivo):* The owner confirms whether any request body limit exists in front of the backend (none is set in the repository Caddyfile), whether port 8000 is reachable from the Internet, and the free space and sharing of the filesystem backing the container's /tmp.

---

## 15. Evidence images are written to the shared evidence directory without per-user quota or retention, so one resident may exhaust storage for all communities

- **Fingerprint:** `engine.py:evidence-storage-without-quota-or-retention`

Every MEDIO/ALTO event writes a full-resolution JPEG of the decoded frame into the shared events/ directory, bind-mounted from host ./events. /analyze produces up to one image per 5 s of video time with no frame or duration cap, and /ws/live up to one per 5 s per session with unlimited sessions. No application code deletes evidence files, caps per-user totals, or enforces the per-plan quota the schema defines (plans.storage_gb and v_user_storage.quota_bytes are never referenced from Python). Image size follows the decoded frame resolution, which the uploader controls, and a compact long video of a static high-resolution scene with a person loitering could turn one upload of at most 200 MB into many large persistent evidence files. If the shared disk fills, other tenants' evidence writes fail silently (cv2.imwrite's return value is ignored while the event is still recorded), and anything else on that filesystem is affected. The bytes written per upload and the disk layout were not observed.

**Causa raíz propuesta:** Persisted evidence is not accounted per principal and has no quota, size cap or retention, and the write paths never enforce the schema-level quota (plans.storage_gb / v_user_storage).

**Traza:**

- `entrypoint` [backend/app.py:577](backend/app.py) (analyze): An authenticated resident uploads a video (live sessions reach the same kind of write at engine.py:153).
- `propagation` [engine.py:85](engine.py) (GuardianEngine.analyze_video): Every MEDIO/ALTO result at least event_cooldown_s (5 s) of video time after the previous one becomes an event, for the whole video.
- `sink` [engine.py:95](engine.py) (GuardianEngine.analyze_video): cv2.imwrite writes a full-resolution JPEG of the frame into the shared evidence directory, where nothing ever deletes it.

**Evidencia:**

- engine.py:153: Live sessions persist one evidence image per event per session into the same directory.
- backend/app.py:563: Live sessions are created with the shared EVIDENCE_DIR.
- bd_intivision/schema_vigia.sql:13: plans.storage_gb defines a per-plan storage quota.
- bd_intivision/schema_vigia.sql:311: The v_user_storage quota view exists, but no Python code references it or plans.storage_gb.
- docker-compose.yml:20: The evidence directory is a host bind mount shared by all communities.
- backend/app.py:33: MAX_UPLOAD_MB caps input bytes only, not decoded output volume.

**Bloqueos (lo que falta saber):**

- The bytes each event can write (decoded-resolution limits of the bundled FFmpeg/OpenCV and the JPEG size of attacker-chosen frames) can only be learned by running the decoder, which was not possible in this run.
- The host's disk capacity, whether ./events shares a filesystem with PostgreSQL data and Docker storage, and whether any operator cleanup job exists are deployment facts.

**Plan de validación:**

- *Local (en sandbox, datos ficticios):* In a sandbox with no external network, CPU/memory/time limits and a 256 MB size-limited filesystem for the events directory, run analyze_video with a stub detector and scorer that return ALTO for every sampled frame. Analyse one synthetic MP4 of at most 5 MB showing a static high-resolution scene, as a dummy resident through the local backend or directly via GuardianEngine. Measure total bytes written to events/ against the input size, and stop at the quota.
- *Despliegue (lo observa el dueño, sin pruebas en vivo):* The owner checks the disk capacity and filesystem layout behind ./events, whether PostgreSQL data shares that filesystem, and whether any retention or cleanup job runs.

---

## 16. Locked-account login branch skips bcrypt, so response time reveals whether an account exists

- **Fingerprint:** `backend/auth.py:authenticate:locked-path-skips-bcrypt-timing`

POST /login is meant to make unknown, inactive, wrong-password and locked accounts indistinguishable. Every failure returns the same 401 body, and unknown or inactive accounts run a dummy bcrypt check so they take as long as a real check. The locked branch in auth.authenticate returns before any bcrypt work. Every other failure path runs exactly one bcrypt verification at the same default cost. An anonymous visitor can send five wrong passwords for a candidate email or phone number and then time a sixth attempt. A registered, now-locked account answers without the bcrypt delay; an unregistered identifier answers after it. This reveals whether that email or phone belongs to a VIGIA user, and it locks the real account for LOCK_MINUTES as a side effect. The impact is limited to disclosure of account existence. The per-IP throttle (30 failures per 15 minutes by default) limits the rate to about five identifiers per source address per window.

**Causa raíz propuesta:** backend/auth.py authenticate returns 'locked' at line 107 before calling verify_password. The locked path therefore does no bcrypt work, while the unknown, inactive and wrong-password paths each run one bcrypt verification at the same default cost (bcrypt.gensalt() for both _DUMMY_HASH and user hashes).

**Traza:**

- `entrypoint` [backend/app.py:438](backend/app.py) (login_submit): Anonymous POST /login with form fields identifier and password.
- `propagation` [backend/app.py:445](backend/app.py) (login_submit): Calls auth.authenticate(identifier, password).
- `propagation` [backend/auth.py:112](backend/auth.py) (authenticate): Earlier failed attempts for an existing account: the fifth consecutive failure sets locked_until = now() + LOCK_MINUTES (CASE at line 116).
- `propagation` [backend/auth.py:96](backend/auth.py) (authenticate): The next attempt looks up the account by lower(email) or phone and computes the locked flag from locked_until.
- `sink` [backend/auth.py:107](backend/auth.py) (authenticate): Locked account: returns 'locked' immediately with no bcrypt work, so the response comes back roughly one bcrypt duration sooner than for an unknown identifier.

**Evidencia:**

- backend/auth.py:104: Unknown or inactive identifier: one bcrypt verification against _DUMMY_HASH, then returns 'invalid'.
- backend/auth.py:64: Comment states the intent: the dummy hash makes a nonexistent user take the same time, so account existence cannot be deduced from response time.
- backend/app.py:450: Comment states the intent: the same message and code for 'does not exist', 'wrong password' and 'locked', so nobody can find out which accounts exist.
- backend/app.py:453: Identical 401 response for every failure reason; only timing differs.
- backend/auth.py:66: The dummy hash is created with bcrypt.gensalt() at default cost.
- backend/auth.py:53: User password hashes (manage.py create-user and set-password call hash_password) also use bcrypt.gensalt() at default cost, so unknown and wrong-password paths take about the same time and only the locked path differs.
- backend/app.py:440: The per-IP throttle (defaults at backend/auth.py:34-35) limits the probe rate but does not remove the oracle.

**Bloqueos (lo que falta saber):**

- Target code was not executed (no OS-enforced sandbox, PostgreSQL instance or bcrypt runtime), so the response-time gap between the locked branch and the unknown-account branch has not been measured.

**Plan de validación:**

- *Local (en sandbox, datos ficticios):* In a sandbox with a disposable local PostgreSQL loaded from bd_intivision/schema_vigia.sql and one dummy active user, call backend.auth.authenticate directly. Make five calls with a wrong password for the dummy user, then time a sixth call. Separately, time six calls for an unregistered identifier. The claim holds if the dummy user's sixth call returns 'locked' in a few milliseconds while each call for the unregistered identifier takes about one bcrypt duration (hundreds of milliseconds at the default cost of 12). Regression check: call verify_password(password, _DUMMY_HASH) before returning 'locked' and confirm the timings converge.

---

