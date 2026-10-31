# Operación del laboratorio

## Índice

- [Requisitos e instalación](#requisitos-e-instalación)
- [Encendido y apagado](#encendido-y-apagado)
- [Cambio de IP](#cambio-de-ip)
- [Respaldo y restauración](#respaldo-y-restauración)
- [Solución de problemas](#solución-de-problemas)
- [Límites de seguridad](#límites-de-seguridad)

## Requisitos e instalación

Requiere Docker Desktop/Engine con `docker compose`; el MVP también describe Flutter y Python para el servidor web opcional. Compose espera variables en `.env` para secretos obligatorios de Keycloak, app y SCIM. Los valores del `.env` existente y `.env.example` no se leyeron; no copiar ni mostrar valores secretos. (README.md, sección Inicio rápido; compose.yaml:6-18,31-50)

Desde PowerShell, en la raíz:

```powershell
Copy-Item .env.example .env
# Completa variables localmente en un gestor/editor seguro; no las pegues en chat.
docker compose up --build -d
docker compose ps
```

El servicio app se construye desde `Dockerfile`; al inicio ejecuta `app.seed`, `app.provision_directory` y Gunicorn. Keycloak importa el realm al inicializarse y app espera healthcheck de Keycloak. (Dockerfile:1-14; compose.yaml:3-27,68-73)

## Encendido y apagado

```powershell
.\iniciar.ps1
```

El script comprueba Docker, detecta una IP LAN, archiva el volumen app si existe, compara IP guardada en `.env` y ejecuta Compose o `actualizar-ip.ps1`; espera hasta 200 segundos a `/health`. Si NFC está habilitado, abre otra consola PowerShell con `ExecutionPolicy Bypass` y el token del puente. No ejecutar el script con archivos de proyecto modificados sin revisar primero su diff: puede modificar configuración y reiniciar los servicios. (iniciar.ps1:1-39; actualizar-ip.ps1:20-64)

Apagado conservando volúmenes:

```powershell
docker compose down
```

`docker compose down` no borra volúmenes nombrados por defecto. `docker compose down -v` elimina datos locales de app y Keycloak y es destructivo; no usar salvo que se quiera reiniciar completamente el laboratorio. (README.md, sección Comandos útiles; compose.yaml:108-110)

Servidor Flutter Web opcional:

```powershell
.\iniciar-movil.ps1
```

Usa Flutter para `pub get` y `build web --release`, y sirve build web en 9000 por defecto con proxy `/api/` hacia `http://127.0.0.1:5000`. El script escucha el servidor de Python en todas las interfaces, por lo que la página puede ser accesible en la LAN. (iniciar-movil.ps1:1-37; modules/mobile_flutter/tools/serve_web.py:7-19,26-48,67-71)

## Cambio de IP

```powershell
.\actualizar-ip.ps1 -Ip 192.168.1.50
# Opcional, solo si también cambia la IP del teléfono cámara:
.\actualizar-ip.ps1 -Ip 192.168.1.50 -CamIp 192.168.1.51
```

La dirección de ejemplo es ilustrativa. El script acepta `-Ip`/`-CamIp`, pero no valida formalmente que sean IPv4 antes de insertar texto. Actualiza coincidencias de IP en archivos, cambia `APP_BASE_URL` en `.env`, añade URI comodín al realm y después ejecuta `docker compose down` y `up -d --build --force-recreate`. Revisar cambios antes del reinicio y respaldar `.env`/realm de forma segura. (actualizar-ip.ps1:1-59,62-64)

`-ResetKeycloak` además elimina los volúmenes `identity-access-lab-mvp_keycloak_data` y `identity-access-lab-mvp_app_data`; elimina tanto identidad como SQLite. Es destructivo. (actualizar-ip.ps1:63)

## Respaldo y restauración

Hay dos mecanismos distintos:

1. `iniciar.ps1` crea `backups\app_data.tgz` a partir del volumen `app_data` si el volumen existe. El archivo puede contener el SQLite y los ciphertexts de credenciales; debe protegerse y excluirse de Git. No inspeccioné ni leí backups. (iniciar.ps1:13-16; compose.yaml:63-65; app/store.py:68-75)
2. La UI `/admin/respaldo`, permiso `respaldo`, permite exportar JSON e importar `merge` (predeterminado) o `replace`. El bundle incluye perfiles, roles, tarjetas y contraseñas descifradas; el JSON se considera material secreto. `replace` desactiva usuarios que no aparecen, reemplaza tarjetas y credenciales y actualiza la matriz de roles. (app/wsgi.py:566-618; app/store.py:358-478)

**Restauración:** no se encontró un script específico para extraer `app_data.tgz` de vuelta en un volumen. La restauración funcional documentada en código es importar JSON mediante la UI. La restauración de un tar de volumen es **NO VERIFICADA**; validar cualquier procedimiento en un entorno desechable y mantener copia previa. Keycloak se sincroniza vía API para usuarios con contraseña en el bundle antes de aplicar store, por lo que un fallo puede dejar sistemas remotos y SQLite desalineados. (app/wsgi.py:592-618; app/store.py:442-478)

Antes de importar:

- Confirmar origen, versión (`identity-access-lab`, version 1), número/correspondencia de usuarios, cuentas reservadas y modalidad.
- No importar archivos de origen desconocido: validar bundle evita tipos/roles/UID malformados, pero las credenciales son secretos en claro. (app/store.py:384-440)
- Preferir ensayo controlado en `merge`; para `replace`, tener respaldo validado y plan de rollback. La importación no es una transacción global. (app/store.py:442-478)

## Solución de problemas

| Síntoma | Verificación segura basada en el código |
|---|---|
| `Invalid parameter: redirect_uri` | Comparar `APP_BASE_URL/auth/callback` con las URI del cliente realm. `/login` genera callback usando base configurada; el realm tiene varias URI, incluidas comodines. Alinear una URI exacta para host/puerto usados. No ampliar con comodines para resolverlo. (app/wsgi.py:41-45,252-256; keycloak/identity-access-realm.json:34-60) |
| `Account is not fully set up` | **Causa exacta NO VERIFICADA** sin inspeccionar el estado Keycloak. El realm marca credenciales placeholder de demo como `temporary:false`; usuarios creados por la app también reciben password `temporary:false`. Revisar en Keycloak si usuario habilitado tiene credencial inicial y completar acciones pendientes requeridas por el realm. No reproducir valores de contraseña en logs. (keycloak/identity-access-realm.json:68-80; app/provision_directory.py:37-54) |
| Cámara sin imagen | Verificar IP/puerto y credenciales de la app de cámara, ruta de stream, cámara `cam_entrada` habilitada y `cam_interior` desactivada por defecto. `cam_entrada` usa HTTP `/video` vía go2rtc en config actual; interior usa RTSP. La configuración Frigate actual no pudo validarse contra un dispositivo real. (compose.yaml:89-92; frigate/config/config.yml:44-57,66-93) |
| `/health` responde pero servicios no están listos | `/health` devuelve `{"status":"ok"}` sin comprobar IdP/SQLite/Frigate; inspeccionar `docker compose ps` y logs locales sin publicar secretos. (app/wsgi.py:220-222; compose.yaml:25-27,68-73) |
| NFC no abre sesión | Verificar NFC enabled, bearer del puente, tarjeta en lista, usuario activo, y disponibilidad de credencial en las fuentes que busca la app. Cada toque se acepta solo dentro del TTL y poll requiere CSRF. No compartir token ni contraseñas para diagnosticar. (app/wsgi.py:60-98,332-373) |
| Cambio IP reseteó o dejó estado incoherente | No usar `-ResetKeycloak` si se busca conservar datos. El script puede editar archivos y recrear servicios; revisar diff y logs antes de volver a ejecutar. (actualizar-ip.ps1:20-64) |

## Límites de seguridad

Compose usa Keycloak `start-dev`, publica servicios HTTP, permite `AUTHLIB_INSECURE_TRANSPORT=1`, tiene Frigate con auth apagada y valores default `admin/admin` para cámara. Tratar solo como demo local, con datos ficticios y red aislada. (compose.yaml:3-4,20-21,38,45,66-68,89-106; frigate/config/config.yml:10-11)
