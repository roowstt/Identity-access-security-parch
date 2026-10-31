# Modelo de amenazas

## Índice

- [Alcance](#alcance)
- [Activos](#activos)
- [Actores y confianza](#actores-y-confianza)
- [Superficies de ataque](#superficies-de-ataque)
- [Amenazas y mitigaciones observadas](#amenazas-y-mitigaciones-observadas)
- [Supuestos y riesgos residuales](#supuestos-y-riesgos-residuales)

## Alcance

Modelo cualitativo del MVP en la configuración del repositorio. Incluye Flask/Gunicorn, Keycloak, SQLite, SCIM, NFC y vídeo Frigate. No se inspeccionaron secretos/directorio/backups ni infraestructura externa; red y versiones efectivamente desplegadas son **NO VERIFICADAS**. (compose.yaml:1-110; app/wsgi.py:1-697)

## Activos

| Activo | Ubicación / protección observada |
|---|---|
| Identidad y estados/roles locales | JSON de `users.payload` en SQLite; rol y extras afectan permisos de cada request. (app/store.py:68-75,139-181,207-213) |
| Claves OIDC, SCIM y sesión | Variables de entorno; cookie de sesión firmada por `APP_SECRET`; configuración de cliente Keycloak. (app/wsgi.py:32-57,167-174; compose.yaml:31-50) |
| Contraseñas guardadas para NFC | `user_credentials.secret`, cifrado Fernet con clave derivada de `APP_SECRET`; fuentes alternativas de credencial en env/archivo de directorio. (app/store.py:306-347; app/wsgi.py:75-98) |
| UID NFC y relación badge-persona | `nfc_cards`, taps pendientes `nfc_taps`; UID normalizado. (app/store.py:68-75,232-303) |
| Logs de acceso y turnos | `events` SQLite; máximo 500 registros. (app/store.py:216-223) |
| Vídeo y grabaciones | Frigate, volumen host `frigate/storage`, cámara Android/IP Webcam. (compose.yaml:94-99; frigate/config/config.yml:16-25,44-93) |
| Configuración/secretos de despliegue | `.env`, `.directory/accounts.json`, backups; contenido no examinado y considerado privado según el alcance. (compose.yaml:31-65; iniciar.ps1:13-16) |

## Actores y confianza

| Actor | Capacidades o condición |
|---|---|
| Usuario autenticado | Tiene rol local y puede solicitar endpoints; permisos revisados en cada acción protegida. (app/wsgi.py:183-198; app/store.py:207-213) |
| Gestor con `roles` | Puede asignar rol/permisos y cambiar matriz de roles; diseño actual permite concesiones no acotadas por autoridad del actor (H-01). (app/wsgi.py:478-492,553-564) |
| Cliente SCIM que conoce `SCIM_TOKEN` | Puede listar todos los usuarios y mutar/perder cuentas; la tokenización está compartida y global. (app/wsgi.py:167-174,632-694) |
| Poseedor de badge o emulador UID | Puede producir UID; puente envía UID, app lo compara con whitelist y el UID no prueba posesión criptográfica de una clave. (tools/nfc-bridge.ps1:83-115; app/wsgi.py:332-350) |
| Usuario de red con acceso a puertos | Puede alcanzar servicios según firewall y red; Compose no vincula puertos a loopback explícitamente. (compose.yaml:20-21,66-68,101-106) |
| Administrador de host/volúmenes | Puede leer/escribir SQLite, configuración y logs; no existe protección de integridad criptográfica de events en el store. (compose.yaml:22-24,63-65,94-99; app/store.py:216-223) |

## Superficies de ataque

1. Login OIDC, redirect URI, callback y cookie en puertos 8000/8081. (app/wsgi.py:41-57,252-277; compose.yaml:20-21,66-68)
2. SCIM bearer para `/Users` y `/Groups`, incluido acceso a lista y mutaciones. (app/wsgi.py:167-174,632-697)
3. Admin de usuarios/roles, cambios de credenciales, respaldo y restauración. (app/wsgi.py:417-618)
4. `/nfc/tap` con bearer bridge, `/nfc/poll` con sesión/CSRF, lector local y base de taps. (app/wsgi.py:332-373; tools/nfc-bridge.ps1:22-27)
5. API/UI Frigate HTTP 5000, RTSP 8554, WebRTC 8555 y UI HTTPS 8971; además proxy Flutter CORS `*` en 9000. (compose.yaml:101-106; modules/mobile_flutter/tools/serve_web.py:16-19,26-48,67-71)
6. Docker volumes, bind mounts, `.env`, configuración y scripts PowerShell que reescriben archivos o reinician volúmenes. (compose.yaml:22-24,63-65,94-99; actualizar-ip.ps1:20-64)

## Amenazas y mitigaciones observadas

| Amenaza | Controles presentes | Riesgo residual / acción |
|---|---|---|
| Robar/reusar sesión | Cookie HttpOnly, SameSite=Lax, duración 15 min; Secure solo con HTTPS. | HTTP LAN expone cookie; activar TLS y Secure obligatorio. (app/wsgi.py:33-45) |
| CSRF | Token aleatorio guardado en sesión y comparación `compare_digest`; POST protegidos y logout revisados. | Sin `require` no aplica igual: NFC tap usa bearer, poll valida CSRF; pruebas adicionales faltantes para todas las rutas. (app/wsgi.py:176-215,332-373,620-630) |
| Escalamiento por roles | Roles/extras se validan a un conjunto permitido; superadmin reservado; UI limita alta RRHH sin `roles`. | Gestor de roles puede editar matriz y autoasignarse; bloquear escalamiento por actor y cubrirlo con tests. (app/store.py:129-163; app/wsgi.py:441-476,478-492,553-564) |
| Abuso de SCIM | Token comparado en tiempo constante; filtro de usuarios restringido y tamaño de request 2 MiB. | Token global accede a todas las cuentas y cambia estado/roles; TLS, scopes, rotación, logging y segmentación. (app/wsgi.py:39,167-174,632-694) |
| Robo de credenciales de NFC | Cifrado en reposo Fernet; longitud mínima 32 para `APP_SECRET`. | Clave deriva directamente de secreto app; backups exportan password en claro y NFC usa password grant; separar claves y eliminar copia reversible. (app/wsgi.py:41-45,100-135; app/store.py:306-379) |
| Clonación/replay badge | UID debe estar registrado, cuenta activa, bridge token requerido; TTL y consumo una vez por registro. | UID clonable; tap HTTP con bearer puede repetirse creando nuevos taps; usar challenge-response, TLS/loopback y nonce. (app/wsgi.py:332-369; app/store.py:266-303) |
| Acceso a vídeo no autorizado | Frigate tiene puerto 8971 descrito como HTTPS/API autenticada; contenedor tiene no-new-privileges. | Config actual `auth.enabled:false`; HTTP/RTSP/WebRTC publicados y proxy CORS abierto; cerrar y autenticar. (compose.yaml:81-106; frigate/config/config.yml:10-11; modules/mobile_flutter/tools/serve_web.py:16-19) |
| Alteración de auditoría | Store registra denegaciones, login y cambios; retiene los últimos 500. | SQLite local editable, sin cadena de integridad ni exportación remota; eventos de turno no prueban veracidad; enviar a sink append-only. (app/store.py:216-223; app/wsgi.py:311-330) |
| Fuerza bruta/abuso de endpoints | Password mínima 8, content limit global, algunos errores devuelven 401/403. | No se observa rate-limit/cooldown; validar IdP, limitar errores y auditar detección. (app/wsgi.py:33-39,443-461; requirements.txt:1-4) |
| Inyección/config corrupta por script | Sustituciones sobre lista finita de archivos; algunos pasos revisan exit code de Docker. | `Ip`/`CamIp` no validados; regex toca source/config; ResetKeycloak borra volúmenes; validar, hacer backup, diff y confirmación. (actualizar-ip.ps1:1,20-59,62-64) |

## Supuestos y riesgos residuales

- Se asume que el laboratorio se usa con cuentas ficticias, cámaras de prueba y redes controladas; no se ha comprobado el firewall real ni el alcance de publicación de puertos.
- En la auditoría no se leyeron archivos secretos ni datos del volumen. Exposición de credenciales reales, fuerza del secreto, estado de backup y cuentas contenidas permanecen **NO VERIFICADOS**.
- No se ejecutó escáner de dependencias ni se verificó CVE específico. Las versiones fijadas en `requirements.txt` y el tag Frigate son hechos del repositorio, no certificación de ausencia de vulnerabilidades. (requirements.txt:1-4; compose.yaml:81)
- `health` es una respuesta estática y no certifica el estado funcional de OIDC, SQLite, SCIM o vídeo. (app/wsgi.py:220-222)
- No usar esta arquitectura para tratar datos personales, autenticación física real o vídeo sensible hasta completar requisitos del checklist en `AUDITORIA.md`.
