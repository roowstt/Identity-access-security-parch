# Resumen de revisión de seguridad

**Proyecto:** Identity Access Lab MVP
**Rama revisada:** `main`
**Alcance:** revisión estática del código disponible; no se ejecutó el proyecto, scripts, escáner de dependencias ni prueba de intrusión.
**Archivos privados excluidos:** `.env`, `.directory/accounts.json`, `backups/` y datos persistidos del NVR. Su contenido y los valores de secretos son **NO VERIFICADOS** y no se reproducen.

La configuración revisada es apropiada únicamente para una demostración local aislada. No debe exponerse a redes no confiables ni usarse con datos personales/reales hasta resolver los hallazgos altos, proteger los respaldos y desplegar transporte/autenticación apropiados. La revisión completa, evidencia, parches sugeridos y plan de remediación están en [`AUDITORIA.md`](AUDITORIA.md).

| ID | Severidad | Componente | Evidencia | Resumen |
|---|---|---|---|---|
| H-01 | Alta | Roles y permisos | `app/wsgi.py:478-492,553-564`; `app/store.py:129-163,207-213` | Un actor con `roles` puede autoelevarse, conceder privilegios y editar matrices de otros roles. |
| H-02 | Alta | Transporte y sesión | `compose.yaml:20-21,38,45,66-68`; `keycloak/identity-access-realm.json:3-4`; `app/wsgi.py:33-45` | HTTP, `AUTHLIB_INSECURE_TRANSPORT=1` y cookies sin Secure sobre HTTP exponen el flujo a interceptación en redes no confiables. |
| H-03 | Alta | Frigate / CCTV | `frigate/config/config.yml:10-11`; `compose.yaml:81-106`; `modules/mobile_flutter/tools/serve_web.py:16-19,26-48` | Frigate sin autenticación; API, RTSP y WebRTC publicados; el proxy móvil reenvía API sin autenticación. |
| H-04 | Alta | Protección de superadmin | `app/wsgi.py:390-405,494-546`; `app/store.py:30,134-149,232-245`; `app/seed.py:21-29` | Permisos `usuarios`/`nfc` pueden resetear, desactivar o enlazar tarjetas a cuentas superadmin. |
| M-01 | Media | OIDC / URL base | `app/wsgi.py:41-45,252-256`; `keycloak/identity-access-realm.json:34-60`; `actualizar-ip.ps1:41-59` | Validación de `APP_BASE_URL` por prefijo y URI de redirección comodín; impacto de explotación exacto **NO VERIFICADO**. |
| M-02 | Media | Credenciales | `app/store.py:306-347`; `app/wsgi.py:32-45,75-98` | Fernet usa clave derivada directamente de `APP_SECRET`; falta rotación/versionado y el cambio de clave invalida credenciales cifradas y sesiones existentes. |
| M-03 | Media | Respaldo/restauración | `app/store.py:358-478`; `app/wsgi.py:582-618` | La exportación JSON contiene contraseñas en claro; `replace` puede dejar cambios parciales y desactivar cuentas. |
| M-04 | Media | NFC | `app/wsgi.py:60-135,332-373`; `app/store.py:266-303`; `tools/nfc-bridge.ps1:83-115` | UID clonable; no hay challenge-response ni protección de replay por nonce; TTL predeterminado 15 s, mínimo 5 s y sin máximo explícito. |
| M-05 | Media | SCIM | `app/wsgi.py:167-174,632-694`; `app/store.py:139-181` | Un token global permite listar usuarios y crear/modificar/desactivar perfiles, incluidos roles y permisos aceptados por `save`. |
| M-06 | Media | Credenciales de cámara | `compose.yaml:89-92`; `frigate/config/config.yml:44-57` | Usuario y contraseña predeterminados de la cámara son ambos `admin` si no se sobrescriben. |
| M-07 | Media | Auditoría | `app/store.py:216-223`; `app/wsgi.py:183-198,311-330`; `modules/mobile_flutter/lib/main.dart:99-120` | Eventos editables en SQLite, retención máxima 500 y entradas de turno autodeclaradas por usuario con `mantenimiento`; no son evidencia inmutable. |
| M-08 | Media | Scripts de operación | `actualizar-ip.ps1:1,20-59,62-64,68-83`; `iniciar.ps1:13-16,29-38` | Reemplazos regex, IP sin validación formal, eliminación explícita de volúmenes con `-ResetKeycloak`, errores suprimidos y `ExecutionPolicy Bypass`. No se halló `git reset --hard` ni `Invoke-Expression`; inyección de comandos no confirmada. |
| M-09 | Media | Seguridad web | `app/wsgi.py:33-45,176-215`; `requirements.txt:1-4` | No se observa rate limiting ni configuración explícita de cabeceras CSP/HSTS/frame; controles externos **NO VERIFICADOS**. |
| I-01 | Informativa | Contenedores / dependencias | `Dockerfile:1-14`; `compose.yaml:1-110`; `modules/video/compose.yml:1-22`; `requirements.txt:1-4`; `.github/workflows/ci.yml:1-22` | App corre como UID 10001; usuarios efectivos de Keycloak/Frigate y estado de CVE **NO VERIFICADOS**. El Compose alternativo de video referencia Frigate 0.17.1; el principal usa 0.14.1 por defecto. |
| I-02 | Informativa | Contraseñas demo | `app/wsgi.py:15-24`; `keycloak/identity-access-realm.json:68-235` | El código y realm usan variables/placeholders; valores reales y fortaleza **NO VERIFICADOS**, ya que `.env` y `.env.example` no se leyeron. |

No se ejecutaron los tests; su existencia y cobertura cualitativa se describen en `AUDITORIA.md`. No se afirma que la cobertura sea porcentual ni que los servicios se hayan probado en runtime.
