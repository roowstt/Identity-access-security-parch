# Auditoría de seguridad — Identity Access Lab MVP

## Índice y alcance

1. Resumen ejecutivo
2. Archivos revisados y límites
3. Hallazgos priorizados
4. Controles positivos
5. Plan de remediación
6. Checklist listo para presentar
7. Checklist antes de datos reales

Documentos técnicos complementarios: [`docs/ARQUITECTURA.md`](docs/ARQUITECTURA.md), [`docs/API.md`](docs/API.md), [`docs/MODELO_DE_DATOS.md`](docs/MODELO_DE_DATOS.md), [`docs/ROLES_Y_PERMISOS.md`](docs/ROLES_Y_PERMISOS.md), [`docs/OPERACION.md`](docs/OPERACION.md) y [`docs/MODELO_DE_AMENAZAS.md`](docs/MODELO_DE_AMENAZAS.md).

## Resumen ejecutivo

El código revisado implementa controles útiles para un laboratorio: sesión corta, cookie HttpOnly, CSRF en POST protegidos, autorización consultada en servidor, validaciones estructurales y pruebas unitarias de autorización, SCIM, NFC y respaldos. Sin embargo, la configuración actual sirve solo para un laboratorio local controlado: HTTP, credenciales de cámara predeterminadas, Frigate sin autenticación, publicaciones de puertos sin dirección de host restringida y URI de redirección OIDC amplias. No usar con datos reales ni exponer a redes no confiables. (app/wsgi.py:33-45,167-215; compose.yaml:3-21,31-50,66-106; frigate/config/config.yml:10-11)

El hallazgo de mayor prioridad es la autorización del permiso `roles`: una cuenta con ese permiso puede cambiar los permisos de cualquier cuenta, incluso la propia, y también modificar la matriz de cualquier rol distinto de `superadmin`. La validación de `superadmin` impide asignar ese rol a usuarios no reservados, pero no frena la promoción a `admin` ni la concesión de permisos extra. Los tests confirman el comportamiento previsto para el SOC, pero no prueban una autoelevación. (app/wsgi.py:478-492,553-564; app/store.py:129-163,207-213; tests/test_routes.py:138-159)

También falta una salvaguarda común en operaciones sobre cuentas protegidas: permisos `usuarios` o `nfc` bastan para cambiar el estado, restablecer contraseña o asignar badge a cualquier usuario, incluidos los `superadmin` reservados. Un usuario con esos permisos puede resetear la contraseña del superadmin mediante la credencial administrativa de Keycloak y luego autenticarse como esa identidad local de máximo privilegio. (app/wsgi.py:494-546; app/store.py:30,134-149; app/seed.py:21-29)

**Conclusión:** adecuado como demostración local, con redes privadas bajo control y datos ficticios. El riesgo real aumenta sustancialmente si el host es accesible desde LAN: OIDC, cookie y servicios de vídeo viajan por HTTP; además, las credenciales de cámara pueden ser `admin/admin`. (compose.yaml:20-21,38,45,66-68,89-106; keycloak/identity-access-realm.json:3-4)

## Archivos revisados y límites

Se leyeron los archivos de aplicación `app/wsgi.py`, `app/store.py`, `app/seed.py`, `app/provision_directory.py`; `compose.yaml`, `Dockerfile`, `requirements.txt`; `keycloak/identity-access-realm.json`; `frigate/config/config.yml`; `tools/nfc-bridge.ps1`, `iniciar.ps1`, `actualizar-ip.ps1`, `iniciar-movil.ps1`; `modules/mobile_flutter/lib/main.dart`, `modules/mobile_flutter/tools/serve_web.py`, `modules/mobile_flutter/pubspec.yaml` y `modules/mobile_flutter/ios/RunnerTests/RunnerTests.swift`; `modules/video/compose.yml`, `modules/video/config.example.yml`; `tests/test_routes.py`, `tests/test_store.py`, `.github/workflows/ci.yml`, `README.md` y `docs/ARQUITECTURA.md`. La lista de rutas se contrastó directamente con los decoradores en `app/wsgi.py:220-697`.

No se inspeccionaron `.env`, `.directory/accounts.json`, `backups/` ni los datos persistidos del NVR. Sus valores y contenidos quedan **NO VERIFICADOS** y no se reproducen. No se ejecutó el proyecto ni scripts de despliegue; tampoco se hizo un escaneo de CVE/dependencias o una prueba de intrusión. La existencia de las pruebas está verificada por su fuente, pero sus resultados de ejecución no se afirman.

## Hallazgos priorizados

Esfuerzo: XS < 1 día; S 1–2 días; M 3–5 días; L más de 5 días. La severidad distingue riesgo operativo de laboratorio frente a uso real.

| ID | Severidad | Componente | Archivo:línea | Descripción | Impacto | Evidencia | Recomendación | Esfuerzo |
|---|---|---|---|---|---|---|---|---|
| H-01 | Alta | Autorización de roles | `app/wsgi.py:478-492,553-564`; `app/store.py:129-163,207-213` | Cualquier usuario con `roles` puede cambiar rol/permisos extra de cualquier cuenta (incluyéndose) y editar la matriz de cualquier rol salvo restricciones de `superadmin`. | El SOC o una cuenta con `roles` puede autoelevarse con `admin` o conceder permisos sensibles; alterar la matriz puede ampliar privilegios de un conjunto de usuarios. | El endpoint de usuario solo requiere `roles`; `_prepare` valida que el rol/extras existan, no que el actor pueda concederlos. La edición de matriz solo protege al rol `superadmin`. | Separar “asignar permisos acotados” de “conceder privilegios”; denegar autoelevación y reservar asignación de `admin`, permisos sensibles y cambios de matriz a una autoridad superior. Añadir tests negativos. | M |
| H-02 | Alta | Transporte / sesión | `compose.yaml:20-21,38,45,66-68`; `keycloak/identity-access-realm.json:3-4`; `app/wsgi.py:33-45` | Se publica app/Keycloak por HTTP y el realm no exige TLS; la app habilita `AUTHLIB_INSECURE_TRANSPORT=1`. La cookie `Secure` queda apagada para `http://`. | En una LAN no confiable se pueden observar o modificar credenciales OIDC, tokens o cookies; una cookie capturada da acceso según los permisos de la cuenta. | Mapeos Compose sin IP host, realm `sslRequired: none`, variable insegura y `SESSION_COOKIE_SECURE` ligada a URL HTTPS. | En LAN: proxy TLS con hostnames/callbacks exactos, retirar la excepción de Authlib, `Secure`/HSTS y restricciones de red. Para demo local, ligar puertos a loopback. | M |
| H-03 | Alta | Frigate / CCTV | `frigate/config/config.yml:10-11`; `compose.yaml:81-106`; `modules/mobile_flutter/tools/serve_web.py:16-19,26-48` | Frigate tiene autenticación desactivada; Compose publica la UI/API HTTP 5000, RTSP 8554 y WebRTC 8555. El proxy móvil añade CORS `*` y reenvía `/api/` sin autenticación. | Usuarios de red con acceso a esos puertos pueden consultar API/streams y potencialmente vídeo/grabaciones. | Config `auth.enabled: false`, puertos publicados y proxy abierto. | Habilitar autenticación, no publicar API/RTSP/WebRTC directamente; exponer solo proxy TLS autenticado y limitar por firewall/VLAN. Revisar impacto en clientes móviles antes del cambio. | M |
| H-04 | Alta | Protección de superadmin / cuentas | `app/wsgi.py:390-405,494-546`; `app/store.py:30,134-149,232-245`; `app/seed.py:21-29` | Las rutas de estado, credencial y badge no comprueban si el objetivo es `superadmin` ni si el actor tiene autoridad equivalente. El rol protegido evita cambiar su título, no restablecer su contraseña, desactivarlo o registrar/quitar su tarjeta. | Un gestor con `usuarios` (por ejemplo, RRHH seeded) puede restablecer en Keycloak la contraseña de un superadmin y tomar su identidad; `nfc` puede registrar un UID del atacante para el superadmin. | `admin_user_credential`, `admin_user_status`, `admin_user_card` y las rutas genéricas de tarjetas comprueban permiso/UID, no jerarquía de objetivo. | Aplicar un guard central en todas las mutaciones de cuenta privilegiada: solo una sesión superadmin puede gestionar cuentas superadmin; proteger también cambios de rol/extras y caminos NFC genéricos. Añadir tests de no autorización para RRHH/SOC. | M |
| M-01 | Media | OIDC / URL base | `app/wsgi.py:41-45,252-256`; `keycloak/identity-access-realm.json:34-60`; `actualizar-ip.ps1:41-59` | `APP_BASE_URL` se valida con `startswith`: acepta cualquier URL que empiece por `https://` y prefijos `http://192.168.`, `http://10.` o `http://172.` sin verificar octetos/rango de IP. El realm mantiene comodines de redirect URIs para múltiples prefijos de redes privadas y `webOrigins: ["+"]`. | Configuración errónea o no confiable puede abrir callbacks/orígenes más amplios de los necesarios y facilitar redirecciones indebidas. La explotabilidad exacta depende de las reglas de coincidencia de Keycloak y el DNS/red, **NO VERIFICADA**. | Validación de prefijos, URI wildcard y origen `+` visibles en los archivos. | Validar esquema/host/puerto con parser, aceptar solo localhost o IP validada como privada y registrar URI exacta de callback/origen para el entorno. | S |
| M-02 | Media | Credenciales | `app/store.py:306-347`; `app/wsgi.py:32-45,75-98`; `keycloak/identity-access-realm.json:79-80,99-100,119-120` | `user_credentials.secret` usa Fernet (cifrado autenticado; perfil Fernet AES-128-CBC + HMAC-SHA256) con clave determinista `base64url(SHA256(APP_SECRET))`; no hay sal ni versionado/rotación. Al cambiar `APP_SECRET`, el ciphertext existente no se descifra y `credential()` devuelve `None`; también deja inválidas las sesiones firmadas previas. | Si se expone el proceso o el secreto de app, se pueden recuperar todas las contraseñas custodiadas; rotar sin migración rompe NFC y cierra sesiones vigentes. | Derivación y descifrado en Store; claves iniciales del realm son placeholders `${DEMO_PASSWORD_*}`, no valores literales. | Usar gestor de secretos/KMS y clave aleatoria independiente/versionada; planificar rotación y reestablecer credenciales en vez de guardar contraseñas reutilizables cuando sea posible. | M |
| M-03 | Media | Backup / restore | `app/store.py:358-379,384-478`; `app/wsgi.py:582-618`; `tests/test_routes.py:161-190` | El JSON exportado incluye contraseñas descifradas en claro. `replace` da de baja usuarios ausentes y borra tarjetas/credenciales antes de reponer las del bundle; la operación no está dentro de una transacción global ni se revierte ante fallo intermedio. | Robo del archivo equivale a robo de credenciales; importación errónea puede dejar estado parcial y deshabilitar cuentas o credenciales. | Exportación arma `credentials` con `password`; el import aplica cambios por pasos y borra tablas en modo replace. | Cifrar y firmar backups, proteger descarga/almacenamiento, exigir confirmación reforzada para `replace`, simular diff, backup previo y realizar import transaccional con rollback y sincronización remota ordenada. | M |
| M-04 | Media | NFC | `app/wsgi.py:60-135,332-373`; `app/store.py:266-303`; `tools/nfc-bridge.ps1:83-115` | La tarjeta se autentica por UID leído/enviado; el UID es clonable. El bearer del puente protege `/nfc/tap`, pero no hay desafío criptográfico de tarjeta ni nonce/replay control del mensaje de tap. TTL por defecto es 15 s y el código solo impone mínimo de 5 s, sin máximo explícito; cada repetición autorizada del endpoint crea un nuevo tap. | Una tarjeta emulada o token del puente reutilizado puede iniciar sesión como el usuario mapeado, con sus privilegios; mayor impacto para cuentas de alto privilegio. | El puente obtiene UID y lo envía; app hace grant de contraseña con credenciales recuperadas; el endpoint crea `record_tap`. | No tratar UID como factor. Usar tarjeta con challenge-response o segundo factor; limitar bridge a host local/TLS, rotar token, nonce con expiración y exigir MFA para cuentas privilegiadas. | L |
| M-05 | Media | SCIM / autorización | `app/wsgi.py:167-174,632-694`; `app/store.py:139-181` | Un único `SCIM_TOKEN` autoriza listar usuarios y crear, modificar o desactivar perfiles, incluyendo `title` y permisos extra aceptados por `save`. El token no está sujeto a rol local ni a scopes por operación. | Filtración del token permite cambio de roles/estado y exposición de directorio; `user_id` por sí mismo no supera el bearer, pero cualquier portador tiene acceso global (no es un IDOR anónimo). | `scim_auth` compara un único bearer; PATCH admite título y activo, y `save` valida datos pero no autoridad del llamante. | Rotación/almacenamiento gestionado, TLS, scopes y autorización mínima, auditoría de identidad cliente, límites de lectura y filtros/paginación robustos. | M |
| M-06 | Media | Frigate / cámaras | `compose.yaml:89-92`; `frigate/config/config.yml:44-57` | Si no se sobreescriben, usuario y contraseña de cámara se inicializan ambos como `admin`. | Una cámara que conserve credenciales de fábrica puede ser accedida desde la red; las URLs de flujo podrían filtrar credenciales en logs/config. | Defaults Compose y credenciales interpoladas en URI de fuente. | Hacer obligatorias variables fuertes/únicas; cambiar credenciales en teléfono/cámara, restringir red y evitar registrar URL completa. | S |
| M-07 | Media | Trazabilidad | `app/store.py:216-223`; `app/wsgi.py:183-198,311-330`; `modules/mobile_flutter/lib/main.dart:99-120` | Eventos viven en la misma SQLite, son modificables por quien controla el archivo/proceso y se recortan a los últimos 500. Un usuario con `mantenimiento` puede registrar declaraciones de turno/estado/cámaras; el cliente Flutter también contiene eventos de muestra locales. | No hay garantía de inmutabilidad, integridad ni no repudio; no usar como evidencia forense. | Inserción y retención limitada; el POST de turno requiere `mantenimiento` y admite valores de cámara provistos por el cliente; Flutter inicializa una lista local fija. | Enviar eventos a colector remoto append-only con identidad/tiempo firmado, control de acceso independiente, retención y separar eventos simulados de los reales. | M |
| M-08 | Media | Scripts / operación | `actualizar-ip.ps1:1,20-59,62-64,68-83`; `iniciar.ps1:13-16,29-38` | La actualización sustituye texto con regex en archivos, escribe `.env` y realm; `-Ip`/`-CamIp` no tienen validación de formato explícita. `-ResetKeycloak` elimina los volúmenes app y Keycloak; las esperas suprimen excepciones. El script de inicio lanza puente con `ExecutionPolicy Bypass`. | Configuración corrupta, datos borrados accidentalmente o fallos de arranque ocultos; no se encontró `git reset --hard` ni ejecución dinámica tipo `Invoke-Expression` en los scripts revisados (**no presente en los archivos revisados**). No se confirma inyección de comandos en estos flujos. | Comandos/funciones indicados; reset requiere switch explícito. | Validar direcciones IPv4 con parser, escribir archivos de forma atómica con copia previa, mostrar diff/confirmar borrado, propagar errores y firmar/limitar scripts; retirar Bypass cuando no sea imprescindible. | S |
| M-09 | Media | Seguridad web | `app/wsgi.py:33-45,176-215`; `requirements.txt:1-4` | Existe CSRF para POST en rutas protegidas y logout, pero no se observa rate limiting ni configuración explícita de CSP, HSTS, X-Frame-Options o X-Content-Type-Options. Contraseña mínima 8 caracteres; no se ve limitación de fallos OIDC. | Fuerza bruta y riesgos de clickjacking/XSS/inyección se mitigan menos de lo deseable fuera de un entorno local. | Configuración de app y dependencias revisadas; no se agregó extensión de rate limiting. No equivale a demostrar que un proxy externo no añada cabeceras. | Añadir limitación de intentos en IdP y app, cabeceras en proxy/Flask, política de contraseñas y revisión de plantillas/contextos de salida. | M |
| I-01 | Informativa | Contenedores / cadena de suministro | `Dockerfile:1-14`; `compose.yaml:1-110`; `modules/video/compose.yml:1-22`; `requirements.txt:1-4`; `.github/workflows/ci.yml:1-22` | App corre como UID 10001 no root; Frigate tiene `no-new-privileges`; Keycloak y Frigate no especifican `user`. Requisitos Python están fijados por versión pero no hay evidencia de lock/hash ni escaneo CVE. El servicio raíz de Frigate tiene default 0.14.1 con override; el Compose alternativo de `modules/video/` referencia 0.17.1 y no forma parte del Compose raíz. | El estado de CVE vigente y usuarios efectivos de imágenes Keycloak/Frigate quedan **NO VERIFICADOS**; no afirmar vulnerabilidades específicas sin escaneo. | Definiciones de imagen y CI; pipeline ejecuta unit tests, valida JSON y Compose, no escanea dependencias. | Escaneo SCA de versiones resueltas y de imágenes en CI, fijar digest, y comprobar usuarios/capacidades efectivas antes de publicación. | S |
| I-02 | Informativa | Contraseñas demo | `app/wsgi.py:15-24`; `keycloak/identity-access-realm.json:68-235`; `README.md` (sección de inicio rápido) | En el código se usan nombres de variables para credenciales demo y el realm contiene placeholders de variables con `temporary:false`; no se revisaron archivos `.env` ni `.env.example`. Los valores exactos y su fortaleza son **NO VERIFICADOS**. | No puede descartarse reutilización o exposición de claves desde archivos locales ignorados. | El realm evidencia placeholders, no literales; la app resuelve variables de entorno. | Usar únicamente valores aleatorios temporales para demo y rotarlos; no reutilizar contraseñas ni publicar `.env`/archivos privados. | XS |

### Parche sugerido para H-01 (Alta)

Es un ejemplo de endurecimiento, no está aplicado al código. Mantiene el permiso de consulta de matriz, reserva cambios de matriz para `admin` y bloquea autoasignación y concesiones sensibles a actores que no sean `admin`. Si la política del laboratorio necesita que SOC pueda asignar roles, implementar una lista permitida explícita en lugar de ampliar esta regla.

```diff
@@
     @app.post("/admin/usuarios/<user_id>/rol")
     @require("roles")
     def admin_user_role(user_id):
         target = store.get(user_id)
         if not target:
             abort(404)
+        new_title = request.form.get("title", "")
+        new_extras = set(request.form.getlist("extraPermissions"))
+        privileged = {"admin", "usuarios", "roles", "nfc", "respaldo"}
+        if user_id == session["uid"] and (
+            new_title != target["title"]
+            or new_extras != set(target.get("extraPermissions") or [])
+        ):
+            abort(403)
+        if not store.permitted(session["uid"], "admin") and (
+            new_title in {"admin", "superadmin"} or new_extras & privileged
+        ):
+            abort(403)
         try:
             store.save(
-                {"title": request.form.get("title", ""), "extraPermissions": request.form.getlist("extraPermissions")},
+                {"title": new_title, "extraPermissions": sorted(new_extras)},
                 user_id,
             )
@@
     @app.post("/admin/roles/<role>")
-    @require("roles")
+    @require("admin")
     def admin_role_permissions(role):
```

### Parche sugerido para H-02 (Alta)

Para uso local, limitar los puertos directos a loopback. Para clientes de LAN, reemplazar estos mapeos por un reverse proxy HTTPS con certificado confiable y callbacks/orígenes OIDC exactos; un cambio de puerto por sí solo no cifra tráfico.

```diff
 services:
   keycloak:
     ports:
-      - "8081:8080"
+      - "127.0.0.1:8081:8080"
   app:
     environment:
-      AUTHLIB_INSECURE_TRANSPORT: "1"
      APP_BASE_URL: ${APP_BASE_URL:?Define un URL HTTPS para el entorno}
     ports:
-      - "8000:8000"
+      - "127.0.0.1:8000:8000"
```

Al activar HTTPS, retirar también en el realm `sslRequired: none` y las URI comodín; fijar `SESSION_COOKIE_SECURE=True` obligatoriamente para el despliegue HTTPS. La corrección del realm y del proxy debe desplegarse coordinada para no romper el callback. (keycloak/identity-access-realm.json:3-4,34-60; app/wsgi.py:33-45)

### Parche sugerido para H-03 (Alta)

Este diff reduce exposición directa; las aplicaciones deben migrar a un proxy Frigate autenticado antes de retirar los puertos que consumen. También evita un fallback conocido de credenciales de cámara.

```diff
 # frigate/config/config.yml
 auth:
-  enabled: false
+  enabled: true
```

```diff
 # compose.yaml
-      FRIGATE_CAM_USER: ${FRIGATE_CAM_USER:-admin}
-      FRIGATE_CAM_PASS: ${FRIGATE_CAM_PASS:-admin}
+      FRIGATE_CAM_USER: ${FRIGATE_CAM_USER:?Define un usuario de cámara único}
+      FRIGATE_CAM_PASS: ${FRIGATE_CAM_PASS:?Define una contraseña única}
@@
     ports:
-      - "8971:8971"
-      - "5000:5000"
-      - "8554:8554"
-      - "8555:8555/tcp"
-      - "8555:8555/udp"
+      - "127.0.0.1:8971:8971"
```

### Parche sugerido para H-04 (Alta)

El ejemplo centraliza la decisión y la aplica a las rutas de estado, restablecimiento de credencial, cambio de rol/extras y tarjetas identificadas por usuario/UID.

```diff
@@
     def scim_error(detail, status):
         return jsonify({"schemas": ["urn:ietf:params:scim:api:messages:2.0:Error"], "detail": detail, "status": str(status)}), status

+    def protect_superadmin_target(target):
+        if target and target.get("title") == "superadmin":
+            actor = store.get(session.get("uid"))
+            if not actor or actor.get("title") != "superadmin":
+                abort(403)
@@
     def nfc_card_create():
         try:
-            store.register_card(request.form.get("uid", ""), request.form.get("username", ""), request.form.get("label", ""))
+            username = request.form.get("username", "")
+            protect_superadmin_target(store.by_name(username))
+            store.register_card(request.form.get("uid", ""), username, request.form.get("label", ""))
@@
     def nfc_card_delete():
-        store.remove_card(request.form.get("uid", ""))
+        uid = request.form.get("uid", "")
+        card = store.card(uid)
+        if card:
+            protect_superadmin_target(store.by_name(card["username"]))
+        store.remove_card(uid)
@@
     def admin_user_role(user_id):
         target = store.get(user_id)
         if not target:
             abort(404)
+        protect_superadmin_target(target)
@@
     def admin_user_status(user_id):
         target = store.get(user_id)
         if not target:
             abort(404)
+        protect_superadmin_target(target)
@@
     def admin_user_credential(user_id):
         target = store.get(user_id)
         if not target:
             abort(404)
+        protect_superadmin_target(target)
@@
     def admin_user_card(user_id):
         target = store.get(user_id)
         if not target:
             abort(404)
+        protect_superadmin_target(target)
```

La aplicación de producción debería evitar contraseñas reversibles/compartidas y exigir MFA o un flujo de recuperación independiente para estos usuarios; el guard evita que un permiso de área sea suficiente para tomar la identidad.

## Controles positivos observados

- CSRF con token de sesión en POST protegidos y comparación con `hmac.compare_digest`; logout exige token. (app/wsgi.py:176-215,620-630)
- Cookies HttpOnly y SameSite=Lax, duración de 15 minutos; límite de cuerpo de 2 MiB. (app/wsgi.py:33-39)
- Revisión servidor de usuario activo/permisos en cada ruta decorada. Desactivar un usuario invalida el acceso aun con sesión previa. (app/wsgi.py:183-198; app/store.py:188-213)
- SQL parametrizado en consultas del store; validación de rol, permisos conocidos, UID hexadecimal y token SCIM en tiempo constante. (app/store.py:129-163,232-250; app/wsgi.py:167-174)
- Cuenta `superadmin` reservada a tres identificadores de roster y su matriz debe conservar `usuarios` y `roles`. (app/store.py:30,134-149)
- La app Docker corre como `appuser` UID 10001; el directorio de cuentas se monta read-only y Frigate tiene `no-new-privileges`. (Dockerfile:8-14; compose.yaml:63-65,86)
- Tests existentes cubren permisos básicos, CSRF/NFC, SCIM, backup y validaciones de store. (tests/test_routes.py:51-190,224-387; tests/test_store.py:18-154)

## Pruebas y cobertura cualitativa

No se ejecutaron pruebas y no se asignan porcentajes. La cobertura aproximada se expresa cualitativamente a partir de los casos existentes en `tests/` y la configuración CI. (tests/test_routes.py; tests/test_store.py; .github/workflows/ci.yml:10-22)

| Módulo | Cobertura observada | Huecos principales |
|---|---|---|
| Store / modelo | Media: roles básicos, desactivación, vinculación OIDC, tarjetas, TTL, Fernet, backup merge/replace y datos inválidos. (tests/test_store.py:18-154) | Rotación de APP_SECRET, concurrencia, integridad/auditoría, fallo a mitad de importación y atomicidad. |
| Rutas Flask / autorización | Media y selectiva: acceso anónimo, permisos admin/RRHH/SOC, alta, cambio estado, export/import, turno y parte de errores. (tests/test_routes.py:51-190,368-387) | Autoelevación, reset de superadmin por RRHH, modificación de matriz privilegiada, cada ruta/método, cabeceras, rate limiting, callback OIDC real. |
| SCIM | Parcial: bearer requerido, listado y PATCH de estado local. (tests/test_routes.py:56-70) | POST/PUT/DELETE/Groups, scopes, delegación de privilegios y paginación/filtros extremos. |
| NFC | Parcial/representativa: token, UID no registrado, usuario inactivo, tap, CSRF, grant mockeado, apagado, tarjeta, logout y TTL. (tests/test_routes.py:224-366; tests/test_store.py:61-75) | UID clonado, replay con token, carrera de taps, integración real lector/Keycloak y abuso de credenciales. |
| Backup | Parcial: export/import merge y replace desactiva usuario ausente. (tests/test_routes.py:161-190; tests/test_store.py:121-148) | Reemplazo destructivo completo, rollback/errores intermedios, archivo grande y exposición en tránsito/almacenamiento. |
| PowerShell, Docker, Frigate y vídeo | Sin prueba funcional observada en `tests/`; CI valida la configuración Compose y el JSON del realm, no levanta los servicios ni prueba video/scripts. (actualizar-ip.ps1; iniciar.ps1; compose.yaml; frigate/config/config.yml; .github/workflows/ci.yml:15-22) | Pruebas de configuración, exposición de red, arranque, permisos de contenedor, cámaras y procedimientos de backup/restauración. |
| Flutter | Sin prueba de comportamiento de la app en la suite revisada. `RunnerTests.swift` solo contiene el test de ejemplo vacío; CI ejecuta unittest Python y no Flutter test. (modules/mobile_flutter/ios/RunnerTests/RunnerTests.swift:4-9; .github/workflows/ci.yml:10-22) | Tests Dart de API/proxy, estados de cámara, turnos y seguridad móvil/web. |

## Plan de remediación

### Fase 1 — Inmediato

1. Deshabilitar exposición de Frigate 5000/8554/8555 a redes no confiables; habilitar auth y reemplazar credenciales predeterminadas de cámaras. (compose.yaml:89-106; frigate/config/config.yml:10-11)
2. Restringir roles: corregir autoasignación, asignación de privilegios y cambios de matriz; crear regresiones para SOC/administrador. (app/wsgi.py:478-492,553-564)
3. Proteger estado, reset de credencial y alta/baja de tarjetas para superadmins; agregar pruebas de denegación para RRHH/SOC. (app/wsgi.py:494-546; app/seed.py:21-29)
4. Mantener el despliegue solo en loopback/VLAN aislada hasta contar con HTTPS, y no reutilizar contraseñas ni valores reales. (compose.yaml:20-21,66-68; keycloak/identity-access-realm.json:3-4)
5. Tratar exportaciones y tar de volumen como secretos; confirmar respaldo previo y realizar una prueba de restauración en entorno desechable antes de usar `replace`. (app/wsgi.py:582-618; app/store.py:358-478; iniciar.ps1:13-16)

### Fase 2 — Corto plazo

1. Implementar TLS, callbacks/orígenes OIDC exactos, cookie Secure obligatoria y quitar `AUTHLIB_INSECURE_TRANSPORT`. (app/wsgi.py:33-45; compose.yaml:45; keycloak/identity-access-realm.json:34-60)
2. Endurecer SCIM: scopes mínimos por cliente, rotación de token, acceso por red y limitar operaciones con elevación de privilegios. (app/wsgi.py:167-174,632-694)
3. Versionar claves de cifrado con rotación soportada o transición a gestor de secretos; decidir no conservar contraseñas de usuarios si el flujo NFC puede cambiarse. (app/store.py:306-347)
4. Añadir rate limiting, cabeceras de seguridad, logging remoto inmutable y validación de IP en el script de cambio. (app/wsgi.py:33-45; actualizar-ip.ps1:1,20-59)
5. Ejecutar escáner SCA e imágenes, fijar versiones/digest y documentar usuarios/capacidades de cada imagen; el estado CVE es **NO VERIFICADO** en esta auditoría. (requirements.txt:1-4; compose.yaml:1-110; .github/workflows/ci.yml:1-22)

### Fase 3 — Largo plazo

1. Migrar SQLite a un servicio de datos con cifrado, backups protegidos y restauración probada/atómica; la app actual guarda todos los datos locales en un volumen. (compose.yaml:63-65; app/store.py:68-75)
2. Sustituir NFC UID por challenge-response y añadir MFA reforzado a cuentas privilegiadas. (tools/nfc-bridge.ps1:83-115; app/wsgi.py:100-135)
3. Añadir integración E2E, tests negativos de autorización, tests de importación parcial, cabeceras, límites de fuerza bruta y verificación del flujo de vídeo autenticado. (tests/test_routes.py; tests/test_store.py; .github/workflows/ci.yml:10-22)
4. Definir gestión de identidades privilegiadas, respuesta a incidentes, retención legal y monitoreo de acceso a vídeo; no se encontró evidencia de esos controles en los archivos revisados. (compose.yaml:1-110; app/store.py:216-223)

## Checklist listo para presentar

- [ ] Todas las pruebas automatizadas se ejecutaron y sus resultados quedaron documentados; esta auditoría no las ejecutó.
- [ ] Las cuentas de demostración usan credenciales conocidas por el equipo, temporales y no reutilizadas; valores actuales **NO VERIFICADOS**.
- [ ] No se incluyen `.env`, `.directory/accounts.json`, bases de datos, imágenes de cámara ni backups en la entrega.
- [ ] Los hallazgos H-01/H-02/H-03 están explicados como riesgos de seguridad y las limitaciones de demostración están visibles.
- [ ] H-04 está explicado: roles `usuarios`/`nfc` no deben poder cambiar credenciales/estado/badges de cuentas privilegiadas.
- [ ] La documentación refleja los puertos y permisos del código actual y no afirma que las pruebas hayan corrido.

## Requisitos antes de usar con datos reales

- [ ] HTTPS end-to-end, callbacks y origins exactos, cookie Secure, sin `AUTHLIB_INSECURE_TRANSPORT`.
- [ ] Frigate auth habilitada; cámaras con credenciales únicas, sin defaults; API, RTSP y WebRTC segregados y autenticados.
- [ ] Correcciones H-01 y H-04 aprobadas y probadas; cuentas de mayor privilegio con MFA y separación de funciones.
- [ ] SCIM con scopes, rotación y controles de red; `SCIM_TOKEN` y `APP_SECRET` de alta entropía en gestor de secretos.
- [ ] Backups cifrados/firmados, protección de archivos y prueba de recuperación/rollback; reemplazo con confirmación y reconciliación externa.
- [ ] Plan de rotación de claves Fernet y migración/restablecimiento de credenciales.
- [ ] Revisión de privacidad/consentimiento, retención de video, logging inmutable y respuesta a incidentes.
- [ ] Escaneo SCA/CVE vigente, imágenes fijadas por digest, usuarios no root verificados y prueba de penetración independiente.

**No habilitar uso con datos reales mientras alguna casilla crítica permanezca pendiente.**
