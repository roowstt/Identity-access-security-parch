# Roles y permisos

## Índice

- [Matriz de permisos](#matriz-de-permisos)
- [Reglas de autorización](#reglas-de-autorización)
- [Reglas de escalamiento y superadmin](#reglas-de-escalamiento-y-superadmin)
- [Permisos por ruta](#permisos-por-ruta)

## Matriz de permisos

Los valores iniciales de la matriz se cargan en `app/store.py:15-22`. Cambios persistidos en `role_permissions` sustituyen los defaults al leer roles; cada usuario suma `extraPermissions` al permiso del rol. (app/store.py:122-126,207-213)

| Rol | consulta | mantenimiento | admin | registros | usuarios | roles | nfc | respaldo |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| `superadmin` | Sí | Sí | Sí | Sí | Sí | Sí | Sí | Sí |
| `admin` | Sí | Sí | Sí | Sí | Sí | Sí | Sí | Sí |
| `mantenimiento` | Sí | Sí | — | — | — | — | — | — |
| `auditor` | Sí | — | — | Sí | — | — | — | — |
| `consulta` | Sí | — | — | — | — | — | — | — |
| `sin_acceso` | — | — | — | — | — | — | — | — |

Nota: el código declara migraciones de permisos para instalaciones existentes (`ROLE_PERMISSIONS_MIGRATIONS`), por lo que la matriz efectiva de una base ya migrada puede diferir del default literal. (app/store.py:24-26,78-92)

## Reglas de autorización

- Decorador `require` redirige a `/login` si falta sesión; consulta usuario activo y permiso en Store; deniega con 403 y registra evento si no autorizado. En POST protegido exige CSRF. (app/wsgi.py:176-198)
- `require_any` autoriza si el usuario tiene al menos uno de los permisos indicados; `/admin/usuarios` usa `usuarios` o `roles`. (app/wsgi.py:200-215,417-420)
- Los permisos se evalúan desde la cuenta local en cada solicitud. Una desactivación o cambio de rol afecta a la sesión existente. (app/store.py:207-213)
- Los permisos extra se agregan (no sustituyen) a los permisos del rol. Los nombres se validan frente al conjunto `ALLOWED_PERMISSIONS`. (app/store.py:13,155-163,205-213)
- El rol `superadmin` está reservado a tres correos de roster; el perfil de cualquier otro no puede tener ese título y las cuentas reservadas no pueden cambiarse a otro rol por `save`. Su matriz debe retener `usuarios` y `roles`. (app/store.py:30,134-149)
- El claim de Keycloak `lab_role` se exporta como claim de protocolo, pero las rutas de app consultan el rol local en SQLite; no se usa el claim para autorizar rutas. (keycloak/identity-access-realm.json:18-32; app/wsgi.py:183-198; app/store.py:207-213)

## Reglas de escalamiento y superadmin

El alta de usuario solo permite que quien tenga `roles` elija rol/permisos; sin ese permiso, la ruta fija `title=consulta` y `extraPermissions=[]`. Esto protege el formulario de alta de RRHH, pero el endpoint separado `/admin/usuarios/<id>/rol` solo requiere `roles` y no compara el permiso que el actor puede delegar ni bloquea su propio ID. (app/wsgi.py:441-476,478-492)

La operación de editar matriz `/admin/roles/<role>` también requiere solo `roles`, aunque el store únicamente restringe al rol `superadmin`; el resto de matrices puede cambiarse para cualquier permiso conocido. Esto es el hallazgo H-01 de la auditoría. (app/wsgi.py:553-564; app/store.py:129-137)

En teoría de implementación, `/scim/v2/Users` usa token global y puede modificar `title`/`extraPermissions`, por lo que SCIM también es autoridad de aprovisionamiento local de alto impacto. SCIM no usa la matriz de roles de interfaz. (app/wsgi.py:167-174,632-694; app/store.py:165-181)

## Permisos por ruta

Resumen de puertas de autorización; respuestas exactas y parámetros están en [`API.md`](API.md).

| Permiso / control | Operaciones |
|---|---|
| Público | `/`, `/health`, `/oidc/metadata`, `/login`, `/auth/callback`; `/nfc/tap` solo bearer puente; `/nfc/poll` CSRF de sesión; grupos/usuarios SCIM solo bearer SCIM. |
| `consulta` | `/consulta`, `/turno`, GET `/api/turno` |
| `mantenimiento` | POST `/mantenimiento`; POST `/api/turno` tiene excepción explícita para exigir `mantenimiento`. |
| `admin` | POST `/admin` |
| `registros` | GET `/registros` |
| `nfc` | `/nfc`, `/nfc/tarjetas`, `/nfc/tarjetas/eliminar`, `/admin/usuarios/<id>/tarjeta` |
| `usuarios` o `roles` | GET `/admin/usuarios` |
| `usuarios` | alta, estado y restablecimiento de credencial de usuarios |
| `roles` | asignación de rol/permisos y edición de matriz |
| `respaldo` | pantalla, exportar e importar JSON |
| CSRF | Todas las rutas POST protegidas por `require`/`require_any` y `/logout`; `/nfc/poll` requiere CSRF aunque no usa esos decoradores. |

Fuente: decoradores de ruta `app/wsgi.py:279-697`.
