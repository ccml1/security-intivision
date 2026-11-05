# Auditoría de seguridad: intivision (run-1)

## 1. Alcance y cómo se hizo

- **Repositorio:** `AngelGabri3l/intivision`, rama `main`, commit `5999fbf`, árbol de trabajo limpio.
- **Perfil:** `quick`. Fue una sola ola de búsqueda más un crítico final de cobertura. **Es una pasada parcial y no garantiza haber encontrado todo.** Volver a correrla aumenta la cobertura.
- **Presupuesto:** no se fijó ninguno. Se usaron 4 agentes de reconocimiento, 5 de búsqueda, 1 crítico y 6 verificadores independientes.
- **Ejecución:** solo se revisó el código fuente. El equipo es Windows, Docker estaba apagado y no había un sandbox aislado, así que **no se ejecutó ni importó nada del proyecto**. Tampoco se contactó el sitio desplegado (`intivision.online`), el broker MQTT ni Telegram.
- **Corridas anteriores:** ninguna. Es la primera.
- **Cobertura diferida:** 5 unidades quedaron registradas pero sin revisar (sección 7).
- **Consecuencia:** como nada se pudo ejecutar, ningún hallazgo llegó a `confirmed`. Todo lo que depende del comportamiento en ejecución o del despliegue quedó como **pendiente de validar** (`needs_validation`).

## 2. Resumen de la postura de seguridad

La base de la aplicación web está bien hecha. Todo el SQL está parametrizado y no se encontró XSS: el HTML se escapa y la CSP usa nonce. Las contraseñas usan bcrypt, los tokens de sesión y de dispositivos se generan con aleatoriedad criptográfica y solo se guarda su hash. Hay protección CSRF (Origin + SameSite=Lax), y los permisos por rol en eventos, cámaras y evidencias son consistentes. El esquema de la base usa claves foráneas compuestas para aislar comunidades.

Los puntos débiles están alrededor de esa base:
- la **sirena** se controla por MQTT público sin autenticación;
- los **archivos de evidencia** comparten un mismo espacio de nombres entre todos los usuarios;
- las **alertas** (Telegram y sirena) son globales y no respetan las comunidades;
- hay un **puerto 8000 expuesto** que se salta Caddy;
- quedó un **hash de contraseña en el historial de git**;
- faltan **límites de recursos** en el análisis de video.

## 3. Hallazgos confirmados

Ninguno. No hubo forma de producir un resultado observado de forma segura (ver sección 1).

## 4. Hallazgos confirmados en detalle

No aplica.

## 5. Pendientes de validar (sin severidad)

Los 16 registros completos, con su traza, bloqueos y planes de validación, están en [NEEDS-VALIDATION.md](NEEDS-VALIDATION.md). El orden es una prioridad sugerida.

| # | Pista | Traza principal | Qué falta saber | Corrección sugerida |
|---|---|---|---|---|
| 1 | **La sirena se controla sin autenticación** en un broker MQTT público (`broker.hivemq.com`, tópico `guardian/demo7k3x9/siren` escrito en `config.yaml` y en el sketch). El ESP32 obedece `ON:<n>` con cualquier duración, y `OFF`/`WARN` la apagan. | `wokwi/sketch.ino:41-57`, `config.yaml:48-52` | Si hay un dispositivo físico suscrito (el sketch usa la WiFi del simulador Wokwi) y si el broker admite clientes anónimos. | Usar un broker propio con usuario, contraseña, ACL y TLS. Poder cambiar host y tópico por variables de entorno. Limitar la duración en el firmware. |
| 2 | **`/ingest` acepta cualquier nombre de imagen** y lo registra como evidencia de la comunidad del dispositivo, aunque el archivo no exista. El admin de esa comunidad puede luego leer un archivo que aparezca después con ese nombre, incluso de otra comunidad. | `backend/app.py:104, 248, 664, 683` | Ejecutarlo con una base de datos de prueba. El escenario entre comunidades requiere adivinar el nombre de archivo de la víctima. | No aceptar nombres de imagen desde el borde, o exigir que el archivo exista y pertenezca al dispositivo. |
| 3 | **Los nombres de evidencia no son únicos por dueño** (`evt_<archivo>_<ms>.jpg`). Una subida posterior sobrescribe el archivo, y `/evidence` lo entrega a quien registró el nombre primero. | `engine.py:88, 95`, `backend/app.py:248, 683` | Que dos subidas coincidan en nombre y marca de tiempo. Requiere ejecución. | Usar nombres aleatorios (UUID) o rutas por comunidad y evento, y guardar el `sha256`. |
| 4 | **Telegram y la sirena son globales:** fotos de cámaras privadas de todas las comunidades llegan a un solo chat, aunque el dueño haya ocultado la cámara al admin. | `backend/app.py:322, 375, 381`, `notifier.py:59` | Quién está en el chat de Telegram y cuántas comunidades hay. | Enrutar por comunidad (las tablas `notification_channels` ya existen) y respetar `admin_can_view`. |
| 5 | **Hash bcrypt del `basic_auth` de Caddy (usuario `angel`) en el historial de git** (commits `b99bce4`…`41b30dc`) y en la rama remota `chore/aplanar-bd_intivision`. | `Caddyfile` (historial) | Si el repo es público, si el hash sigue desplegado y si la contraseña se reutiliza en otro lado. | Cambiar esa contraseña, borrar la rama y, si hace falta, reescribir el historial. |
| 6 | **El puerto 8000 está publicado** en todas las interfaces y se salta Caddy: sin TLS, cookie sin `Secure` y sin HSTS. | `docker-compose.yml:10` | Firewall o grupo de seguridad del servidor. | Quitar `ports` del backend o usar `127.0.0.1:8000:8000`. |
| 7 | **`client_ip` confía en `X-Forwarded-For`** de cualquier IP privada, no solo de Caddy. Así se puede saltar el límite de intentos de login por IP o falsear la IP de auditoría. | `backend/auth.py:239-263` | Qué IP ve Docker para clientes que entran directo al puerto 8000. | Confiar en XFF solo si viene de la IP de Caddy. |
| 8 | **Los WebSockets `/ws/live` abiertos siguen funcionando** tras logout, suspensión o cambio de contraseña. | `backend/app.py:546, 566-571` | Observarlo en ejecución y saber cuánto vive un socket. | Revalidar la sesión periódicamente y cerrar el socket con 4401. |
| 9 | **El cooldown de alertas en vivo es global por nivel:** las alertas de un usuario silencian las de otros. | `backend/app.py:359-372` | Observarlo, y si hay varios usuarios transmitiendo en vivo. | Aplicar el cooldown por cámara o por comunidad. |
| 10 | **Las alertas de videos subidos no tienen cooldown:** un residente puede inundar la sirena y el chat. | `backend/app.py:344-356` | Observarlo, y los límites de Telegram. | Usar el mismo limitador que la ruta en vivo. |
| 11 | **Un único `ThreatScorer` se reutiliza entre subidas de distintos usuarios:** un video puede bajar el nivel del siguiente. | `engine.py:46, 83`, `threat_scoring.py:54-70` | Observarlo con detecciones reales. | Crear un `ThreatScorer` nuevo por análisis. |
| 12 | **`/analyze` analiza el video de forma síncrona dentro de un handler `async`:** bloquea el servicio para todos mientras dura. | `backend/app.py:577, 603` | Observarlo, y el número de workers. | Usar `asyncio.to_thread` o una cola, con límites de frames y tiempo. |
| 13 | **Sesiones `/ws/live` ilimitadas por usuario** compiten por un único `infer_lock`. | `backend/app.py:539-569`, `engine.py:133` | CPU, latencia y límites delante del backend. | Limitar las sesiones y los frames por segundo por usuario. |
| 14 | **El límite de tamaño solo mira `Content-Length`:** un cuerpo con chunked encoding se recibe completo. | `backend/app.py:183, 472-485` | Versiones de Starlette y multipart, y el comportamiento de Caddy. | Usar `request_body max_size` en Caddy y contar los bytes al leer. |
| 15 | **Las evidencias no tienen cuota ni retención:** un residente podría llenar el disco compartido. | `engine.py:85-95` | Disco del servidor y tamaño real por evento. | Aplicar `plans.storage_gb` y una tarea de limpieza. |
| 16 | **Un login a una cuenta bloqueada no ejecuta bcrypt:** el tiempo de respuesta revela si la cuenta existe. | `backend/auth.py:107` | Medir la diferencia de tiempo. | Verificar contra `_DUMMY_HASH` antes de devolver `locked`. |

Hubo 1 candidato rechazado. No es un hallazgo: era un duplicado de la pista 3 y se fusionó con ella (`engine.py:analyze_video:evidence-filename-not-owner-unique`).

## 6. Mejoras recomendadas y buenas prácticas observadas

**Mejoras recomendadas** (no son vulnerabilidades demostradas):
- `mosquitto.conf` tiene `allow_anonymous true` y el puerto 1883 está publicado. Hoy no se usa porque la app usa HiveMQ, pero conviene corregirlo.
- Los contenedores corren como root (no hay `USER` en los Dockerfiles).
- La app se conecta a Postgres como superusuario (`vigia`). Conviene crear un rol con permisos mínimos.
- `cookie_is_secure` y HSTS confían en `X-Forwarded-Proto` venga de quien venga.
- El registro de auditoría de `/evidence` guarda la IP de Caddy en lugar de `client_ip`.
- `notifier.py` puede escribir el token del bot de Telegram en los logs cuando hay errores de conexión.
- `.dockerignore` no excluye `*.dump`, `backups/` ni `*.mp4`. `.gitignore` no cubre `.env.*`, `*.pem` ni `*.key`.
- Ni las dependencias ni las imágenes Docker tienen versión fija (`>=`, tags flotantes), y los pesos de YOLO se descargan sin verificar su hash.
- El bloqueo por cuenta (5 intentos → 15 min) permite que cualquiera bloquee la cuenta de otro. Es una decisión documentada, pero conviene usar retrasos progresivos.
- `cross_site_request` deja pasar las peticiones que no traen ni `Origin` ni `Sec-Fetch-Site` (solo afecta a navegadores antiguos). Además, la comparación de origen ignora el esquema.
- Al crear una cámara con un nombre ya usado, el mensaje revela que existe una cámara privada de otro residente con ese nombre.
- `/evidence` se cachea 10 minutos (`private, max-age=600`).
- Si el broker MQTT no responde al arrancar, la sirena se desactiva en silencio (cae a `LogActuator`).
- El handler `WARN` del ESP32 llama a `noTone` y podría apagar una sirena ALTO activa. Es una pista nueva que reportó un verificador; no se validó y quedó diferida.

**Buenas prácticas observadas:**
- SQL 100 % parametrizado.
- Escapado HTML consistente y CSP con nonce, sin `unsafe-inline`.
- bcrypt con usuario ficticio para no revelar si una cuenta existe por tiempo (salvo la pista 16).
- Tokens de 256 bits de los que solo se guarda el hash.
- Cookie `HttpOnly` y `SameSite=Lax`.
- Documentación de la API (`/docs`) desactivada por defecto.
- Errores genéricos, sin trazas.
- Claves foráneas compuestas por comunidad.
- Auditoría de los accesos del admin a recursos privados.
- No se encontraron otros secretos en el historial: no hay `.env`, tokens ni claves.

## 7. Cobertura

Ledger (`coverage-ledger.json`, 20 unidades):

| Estado | Unidades |
|---|---|
| Cubiertas, sin hallazgos | 4: CSRF; permisos de cámaras; listado de eventos; XSS e inyección SQL |
| Con candidato | 10 |
| No aplica | 1: CLI `manage.py`, solo para operadores |
| Diferidas por el perfil `quick` | 5 |

Las unidades diferidas son:
- agotamiento del pool de PostgreSQL por peticiones anónimas a login, ingest y health;
- cuerpos de formulario sin límite de tamaño antes de la autenticación;
- FFmpeg/OpenCV abriendo videos subidos sin una lista blanca de demuxers (posible SSRF o lectura de archivos locales);
- imágenes muy comprimidas que se descomprimen a dimensiones enormes (`cv2.imdecode`);
- el `WARN` del ESP32 que puede silenciar una sirena ALTO.

El crítico final rechazó 4 hallazgos que los cazadores dejaron fuera de su alcance, porque ya estaban cubiertos o eran duplicados.

**No se afirma cobertura completa.** La siguiente corrida debería empezar por las unidades diferidas y, si hay un entorno aislado (Docker con `--network none` y una base de datos de prueba), validar las pistas 2, 3, 8, 9, 11 y 16 con los planes locales de NEEDS-VALIDATION.md.

## 8. Notas sobre el proceso

- Los validadores del skill no funcionan en Windows, porque no existe `O_NOFOLLOW`. Se ejecutaron las mismas funciones `validateDocument()` mediante un envoltorio, y tanto `findings.json` como `coverage-ledger.json` son válidos.
- Los 16 candidatos se repartieron entre 6 verificadores frescos, cada uno distinto del cazador original, en lugar de usar un verificador por candidato.
