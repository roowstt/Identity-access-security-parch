# API HTTP

## Índice

- [Convenciones](#convenciones)
- [Rutas de aplicación](#rutas-de-aplicación)
- [Rutas SCIM](#rutas-scim)
- [Códigos y notas](#códigos-y-notas)

## Convenciones

- Todas las rutas corresponden a `app/wsgi.py:220-697`. Parámetros de path y query se indican entre `< >`.
- “Permiso” es el control comprobado por el servidor, no el control visual de la plantilla. POST decorados requieren token CSRF; respuestas `403` indican permiso insuficiente y `400` CSRF inválido. (app/wsgi.py:176-215)
- Cookies de sesión son Flask firmadas, con nombre `identity_access_lab_session`, HttpOnly y SameSite=Lax; Secure solo con base HTTPS. (app/wsgi.py:33-45)
- No se declara formato estable de plantilla HTML ni códigos HTTP de validadores de extensiones. Donde el código devuelve redirect/HTML se documenta el resultado visible, sin inventar esquema.

## Rutas de aplicación

| Método y ruta | Autenticación / permiso | Parámetros / cuerpo | Éxito | Errores relevantes |
|---|---|---|---|---|
| `GET /health` | Público | Ninguno | JSON `{"status":"ok"}`; 200 | No se maneja error de dependencia; el handler solo devuelve estado estático. |
| `GET /oidc/metadata` | Público | Ninguno | JSON de metadata OIDC con token/userinfo/JWKS reescritos a base interna; 200 | 404 si `OIDC_INTERNAL_METADATA_URL` no está configurado; 503 si falla solicitud al IdP. |
| `GET /` | Cookie opcional | Ninguno | HTML login o dashboard; 200 | Si la sesión apunta a usuario inactivo, se limpia. |
| `GET /login` | Público | Host HTTP esperado según `APP_BASE_URL` | Redirect al proveedor OIDC | Si host no coincide, redirect a `APP_BASE_URL/login`; errores OIDC no están gestionados en esta ruta. |
| `GET /auth/callback` | OIDC | Parámetros de callback procesados por Authlib | Asigna `uid`, `csrf`, sesión permanente y redirige `/` | 403 si identidad incompleta/desconocida/inactiva/no vinculable o excepción. |
| `GET /consulta` | `consulta` | Ninguno | HTML; 200 | 302 anónimo a `/login`; 403 sin permiso. |
| `POST /mantenimiento` | `mantenimiento` + CSRF | Token form `csrf` o header `X-CSRF-Token` | HTML; 200 | 302 anónimo; 400 CSRF; 403 permiso. |
| `POST /admin` | `admin` + CSRF | Token CSRF | HTML; 200 | 302/400/403 como arriba. |
| `GET /registros` | `registros` | Ninguno | HTML con últimos eventos; 200 | 302/403. |
| `GET /turno` | `consulta` | Ninguno | HTML de cambio de turno; 200 | 302/403. |
| `GET /api/turno` | `consulta` | Ninguno | JSON `{"events":[...]}` | 302/403. |
| `POST /api/turno` | **`mantenimiento`** + CSRF | JSON o form: `shift` ∈ `mañana/tarde/noche`, `status` ∈ `success/warning/danger`, `cams` lista de strings (máximo 10 se incluyen en el texto del evento) | JSON `{"ok":true}`; evento de turno | 400 si shift/status/lista inválidos o CSRF; 302/403 sin sesión/permisos. |
| `POST /nfc/tap` | Bearer `NFC_BRIDGE_TOKEN`; NFC habilitado | JSON `uid` y `source` opcional | JSON `{"accepted":true,"user":"..."}`; 200 | 404 NFC off; 401 token incorrecto; 403 UID no registrado o cuenta inactiva. |
| `POST /nfc/poll` | Cookie/sesión con CSRF; no exige permiso `nfc` | JSON `{"csrf":"..."}` | `{"login":false}` si no hay tap; o `{"login":true,"user":"..."}` | 404-like payload sólo en NFC off: JSON `{"login":false}` con 200; 400 CSRF; 401 fallo de credenciales/IdP. |
| `GET /nfc` | `nfc` | Ninguno | HTML de gestión NFC; 200 | 302/403. |
| `POST /nfc/tarjetas` | `nfc` + CSRF | Form `uid`, `username`, `label` opcional | Redirect a `/nfc` | 302 con query `error` si UID/usuario inválido; 400 CSRF; 403 permiso. |
| `POST /nfc/tarjetas/eliminar` | `nfc` + CSRF | Form `uid` | Redirect `/nfc` | 400 CSRF; 403 permiso. |
| `GET /admin/usuarios` | `usuarios` **o** `roles` | Ninguno | HTML usuarios; 200 | 302/403. |
| `POST /admin/usuarios` | `usuarios` + CSRF | `userName`, `displayName`, `department`, `employeeNumber`, `active`, opcional `title`/`extraPermissions` si actor tiene `roles`, `password`, `nfc_uid`, `nfc_label` | Pantalla HTML que expone contraseña inicial/temporal; 200 | Redirect con error si faltan datos, password <8, UID incorrecto o duplicado; 503 Keycloak fuera de servicio; 400 CSRF/403 permiso. |
| `POST /admin/usuarios/<user_id>/rol` | `roles` + CSRF | `title`, lista `extraPermissions` | Redirect `/admin/usuarios` | 404 usuario no existe; 400 dato inválido; 400 CSRF/403 permiso. |
| `POST /admin/usuarios/<user_id>/estado` | `usuarios` + CSRF | `active=true/false` | Redirect `/admin/usuarios` | 404 no existe; impedir baja propia devuelve redirect con error; 400 CSRF/403 permiso. |
| `POST /admin/usuarios/<user_id>/credencial` | `usuarios` + CSRF | `password` opcional (generada si vacía; mínimo 8) | HTML muestra credencial actualizada; 200 | 404 no existe; redirect si inválida; 503 Keycloak; 400 CSRF/403 permiso. |
| `POST /admin/usuarios/<user_id>/tarjeta` | `nfc` + CSRF | `uid`, `label`, `action=remove` para quitar | Redirect `/admin/usuarios` | 404 usuario; redirect error UID inválido; 400 CSRF/403 permiso. |
| `GET /admin/roles` | `roles` | Ninguno | HTML matriz; 200 | 302/403. |
| `POST /admin/roles/<role>` | `roles` + CSRF | `permissions[]` | Redirect `/admin/roles` | 400 rol/permisos inválidos o intento de modificar matriz `superadmin`; 400 CSRF/403 permiso. |
| `GET /admin/respaldo` | `respaldo` | Ninguno | HTML conteos de usuarios/tarjetas/credenciales; 200 | 302/403. |
| `POST /admin/respaldo/exportar` | `respaldo` + CSRF | Ninguno | Archivo JSON `identity-access-respaldo-<fecha>.json`; 200 | 400 CSRF/403 permiso. El JSON contiene contraseñas descifradas. |
| `POST /admin/respaldo/importar` | `respaldo` + CSRF | multipart `file`, `mode=merge` o `replace` | Redirect con resumen | Redirect error por archivo/JSON/schema inválido; 503 si Keycloak falla; 400 CSRF/403 permiso. |
| `POST /logout` | Cookie y CSRF, sin decorador `require` | Form `csrf` | Limpia taps no consumidos y sesión; redirect `/` | 400 si sesión o token CSRF faltan/son inválidos. |

Fuentes de handlers y permisos: `app/wsgi.py:220-630`.

## Rutas SCIM

Todas exigen `Authorization: Bearer <SCIM_TOKEN>`, comparado en tiempo constante. El token es global para las operaciones; no se aplica permiso de rol local en estos handlers. (app/wsgi.py:167-174,632-697)

| Método y ruta | Parámetros / semántica | Éxito | Errores relevantes |
|---|---|---|---|
| `GET /scim/v2/Users` | Query opcional `filter=userName eq "correo"`; `startIndex` default 1; `count` default 100, acotado 0–100. Sin filter lista todos. | SCIM ListResponse con `totalResults`, `startIndex`, `itemsPerPage`, `Resources`; 200. | 401 token inválido; 400 filter no admitido. |
| `POST /scim/v2/Users` | JSON recurso usuario; `save` valida correo/rol/estado/permisos y reserva superadmin. | Recurso creado; 201 y header Location. | 401; 409 correo duplicado; 400 datos no válidos. |
| `GET /scim/v2/Users/<user_id>` | ID interno de usuario | Recurso actual; 200 | 401; 404 usuario inexistente. |
| `PUT /scim/v2/Users/<user_id>` | JSON recurso; se fusiona sobre perfil existente via `save` (no implementa reemplazo SCIM completo). | Recurso actualizado; 200 | 401; 404; 409 duplicado; 400 datos no válidos. |
| `PATCH /scim/v2/Users/<user_id>` | `Operations[]`, solo op `replace`/`add`; paths `active`, `title`, `userName`, o ruta ausente + objeto `value`. | Recurso actualizado; 200 | 401; 404; 400 estructura/ruta/op no admitida o datos inválidos; 409 duplicado. |
| `DELETE /scim/v2/Users/<user_id>` | ID interno | Desactiva (no elimina fila), 204 | 401; 404. |
| `GET /scim/v2/Groups` | Ninguno | Lista vacía SCIM; 200 | 401. |

Fuentes: `app/wsgi.py:632-697`; validación de recurso `app/store.py:139-181`.

## Códigos y notas

- Decoradores de permiso: 302 para login requerido y 403 para permiso faltante; CSRF inválido aborta 400. (app/wsgi.py:176-215)
- El tamaño global máximo de request es 2 MiB. (app/wsgi.py:39)
- Métodos no listados en cada endpoint quedan sujetos a la respuesta estándar de Flask/Werkzeug; no se ha probado su código en ejecución (**NO VERIFICADO**).
- El comportamiento de proxies/firewalls externos y TLS no forma parte del código Flask inspeccionado (**NO VERIFICADO**).
