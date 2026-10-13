# 02 · Desarrollo del backend (`inamap-backend-domain`)

El backend de INAMAP es un **dominio** del framework DASHADMIN: solo contiene lo propio de
INAMAP (mapas, geometría, rutas, búsqueda, sesiones de ruta). Autenticación, tenancy, planes,
facturación, correo y websockets los provee el core.

Requisito: el entorno Docker funcionando ([01 · Entorno Docker](01-entorno-docker.md)).

## 1. Cómo se conecta con el core

| El core carga… | Desde |
|---|---|
| El service provider del dominio (si existe la clase) | `app/Providers/AppDomainServiceProvider.php` |
| Las rutas API (todo archivo de la carpeta) | `routes/api/*.php` |
| Rutas web y canales | `routes/web.php`, `routes/channels.php` |
| Configuración (`config('<archivo>.*')`) | `config/*.php` |
| Traducciones (`__('inamap.*')`) | `resources/lang/{es,en}/` |
| Migraciones (y subcarpetas) | `database/migrations/` |
| Seeders | `database/seeders/<Carpeta>/<Clase>.php` |

Los namespaces son `Domain\App\…`, `Domain\Database\…` y `Domain\Tests\…`.

## 2. Estructura

```text
app/
├── Providers/              AppDomainServiceProvider (rate limiters, observers, settings, scheduler)
├── Models/Inamap/          Inamap, Layout, Zone, Structure, Category, Location, Path, PathLink, RouteSession
├── Http/Controllers/API/
│   ├── Inamap/             CRUD (base InamapBaseController) + Editor/LayoutGeometryController
│   ├── Public/             API pública del visor y sesiones de ruta
│   └── Prerelease/         Registro de interesados previo al lanzamiento
├── Http/Middleware/        InamapAccessToken (modos de acceso, token, credencial de dispositivo)
├── Services/Inamap/        Geometry, Routing, Search, Session, Qr, Diagnostics, Plan, Tenancy
├── Rules/, Observers/, Jobs/, Console/Commands/, Mail/
config/                     inamap.php, tenant_settings.php, prerelease.php
database/                   migrations, seeders/ (ver §4.1), data/permissions.json
routes/api/                 inamap.php (autenticada), inamap_public.php (pública), prerelease.php
tests/                      Unit/Inamap (motor puro), Feature/API/Inamap (HTTP)
```

## 3. Ciclo de trabajo

1. Editar el código en `inamap-backend-domain` (se ve al instante en el contenedor).
2. Si se agregó una migración: `./inamap.sh migrate`.
3. Si se agregaron rutas o config: `./inamap.sh restart`.
4. Probar: `./inamap.sh test` y/o con `curl` contra http://localhost:25200/api.
5. Revisar logs: `./inamap.sh logs`.

## 4. Agregar un recurso CRUD

Todos los recursos siguen el mismo patrón (ver `app/Http/Controllers/API/Inamap/`):

1. **Migración** en `database/migrations/`: PK `uuid('id')->primary()`, `tenancy_id`, `tenant_id`,
   `softDeletes()`.
2. **Modelo** en `app/Models/Inamap/` que extienda `InamapModel` (UUID, filtros, borrado lógico y
   el scope `visibleTo($user)` para el aislamiento por tenant).
3. **Carpeta del recurso** `app/Http/Controllers/API/Inamap/<Entidad>/` con:
   - `<Entidad>Controller` que extienda `InamapBaseController` y declare `$resource`,
     `$requestValidator`, `$modelFilter`, `$_model`;
   - `<Entidad>Request` (extiende `InamapRequest`; `req()` da `required` al crear y
     `sometimes|required` al actualizar);
   - `<Entidad>Resource` (forma del JSON) y `<Entidad>Filter` (filtros de listado).
4. **Rutas** en `routes/api/inamap.php` con `$registerCrud(Controller::class)` dentro del grupo
   `['access', 'auth:sanctum']`. El nombre de ruta (`api.inamap.<entidad>.<acción>`) es el
   permiso.
5. **Permisos**: agregar los `route_name` a `database/data/permissions.json` y a los archivos de
   `database/data/rolePermissions/` según el rol. En una base ya sembrada, agregar una migración
   de permisos (ver skill `dashadmin-permissions` del framework) o volver a correr los seeders
   `PermissionSeeder` y `RoleSeeder`.
6. **Pruebas** en `tests/Feature/API/Inamap/`, usando el trait `InteractsWithInamap`.

### 4.1 Seeders

El `DatabaseSeeder` del core corre sus seeders globales (permisos, roles, usuarios, tenants) y
luego **descubre solo** cada `database/seeders/<Carpeta>/<Clase>.php` del dominio
(`Domain\Database\Seeders\<Carpeta>\<Clase>`). Los datos de referencia **no** vienen del core:
los aporta el dominio.

| Carpeta | Seeder | Qué siembra |
|---|---|---|
| `Extended/` | `PermissionSeeder`, `RoleSeeder` | Permisos de `data/permissions.json` y su asignación por rol. |
| `Common/` | `CurrencySeeder` | Monedas CLP (por defecto) y USD. El registro valida `primary_currency` contra esta tabla. |
| `Common/` | `LanguageSeeder` | Idiomas `es` y `en`. |
| `Subscription/` | `TrialSubscriptionPlanSeeder` | Plan `trial` (por defecto, 30 días, límites del plan de prueba: 1 tenant, 1 inamap). Precio 0 hasta definir la oferta. |

Todos son idempotentes (`updateOrCreate`). Para correr uno sobre una base existente:
`./inamap.sh artisan db:seed --class="Domain\Database\Seeders\Common\CurrencySeeder"`, o los
tres de referencia juntos con `./inamap.sh seed:base`.

## 5. Convenciones

| Tema | Regla |
|---|---|
| Claves primarias | Siempre UUID. |
| Aislamiento | Toda tabla de negocio lleva `tenancy_id` y `tenant_id`; las consultas pasan por `visibleTo()`. Un TenancyAdmin con `X-Tenant-Id` ve solo ese tenant (validado). |
| Nombres | Entidad raíz `inamaps`; el resto bajo `inamap/`. Tablas `inamaps` e `inamap_<entidad>`. |
| Geometría | En la unidad del inamap (v1: metros), JSON `[[x,y],…]`, origen arriba-izquierda. Sin PostGIS. |
| Límites de plan | Claves propias del dominio con `InamapPlanLimits`, no con `PlanLimitsService::enforceLimit()` (el core cuenta 0 para claves que no conoce). |
| Textos | Español e inglés en `resources/lang/{es,en}/inamap.php`. |
| Errores públicos | Formato `{ "error": { "code": "…", "message": "…" } }`. |

## 6. Configuración del dominio (`config/inamap.php`)

| Clave | Por defecto | Para qué |
|---|---|---|
| `units` | `m` habilitado | Registro de unidades (extensible). |
| `vertical_types` | ascensor, rampa, escalera, escalera mecánica | Accesibilidad y costo de cada tipo de enlace vertical. |
| `snap_tolerance_m` | `0.30` (`INAMAP_SNAP_TOLERANCE_M`) | Distancia a la que dos vértices se consideran el mismo nodo. |
| `default_walking_speed_mps` | `1.2` | Velocidad si el tenant no define la suya. |
| `graph_cache_ttl` | `3600` | Caché del grafo de rutas (se invalida al editar). |
| `route_sessions.default_ttl_minutes` / `max_ttl_minutes` | 60 / 1440 | Duración de las sesiones de ruta. |
| `route_sessions.app_url` | `INAMAP_APP_URL` | URL del visor público usada en QR y enlaces. |
| `public_rate_limit` / `session_rate_limit` | 120 / 20 por minuto | Protección de la API pública. |
| `plan_limit_defaults` | `max_inamaps = 1` | Límites cuando el plan no los define. |

### 6.1 Ajustes por inamap (`inamaps.settings`)

Se guardan como JSON en el propio inamap y se editan desde el drawer de **Configuración** del
editor o desde la ficha del inamap. Una clave omitida nunca borra la existente (se fusiona).

| Clave | Por defecto | Efecto |
|---|---|---|
| `access_mode` | `open` | `open` o `token_required` (credencial de dispositivo). |
| `session_ttl_minutes` | 60 | Vida de las sesiones de ruta. |
| `initial_layout_id`, `initial_zoom`, `primary_color` | — | Cómo abre el visor. |
| `allow_zone_overlap` | `false` | **Zonas solapadas.** Apagado, las zonas de un layout no pueden solaparse (RF-06): el editor las marca en rojo y el servidor responde 422 en `POST/PUT inamap/zones` y en `PUT inamap/layouts/{id}/geometry`. Encendido, ninguna de las dos comprobaciones rige para ese inamap (el borde compartido siempre se permite). Estructuras: sin cambios, solo advierten. Apagarlo no modifica las zonas ya guardadas, pero las nuevas ediciones vuelven a validarse. `Inamap::allowsZoneOverlap()`. |

## 7. Pruebas

```bash
./inamap.sh test                               # toda la suite Domain
./inamap.sh test --filter=RouteEngineTest      # una clase
./inamap.sh test --filter=accessible           # por nombre de método
```

- `tests/Unit/Inamap`: geometría y motor de rutas, en PHP puro (sin base de datos).
- `tests/Feature/API/Inamap`: endpoints HTTP con base de datos. El `TestCase` del core migra y
  siembra `im_dev_db_test` automáticamente; nunca toca la base de desarrollo.

## 8. Probar la API a mano

```bash
API=http://localhost:25200/api

# Login (correo y contraseña de TENANT_ADMIN_* en .env.app.local)
TOKEN=$(curl -s -X POST $API/login -H 'Content-Type: application/json' \
  -d '{"email":"tenant@inamap.local","password":"<contraseña>"}' | jq -r '.data.token // .token')

# Crear un inamap
curl -s -X POST $API/inamaps -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"name":"Campus Central"}' | jq

# API pública (sin token, inamap publicado y en modo open)
curl -s "$API/inamap/<inamapId>" | jq '.data.layouts'
curl -s "$API/inamap/search?inamap=<inamapId>&q=baños" | jq
```

> La forma exacta de la respuesta de login la define el core; si el token no aparece en
> `.data.token`, revisar la respuesta completa.

## 9. Depuración

| Necesito ver… | Comando |
|---|---|
| Errores de Laravel | `./inamap.sh logs` |
| Si una petición llegó al servidor | `./inamap.sh logs nginx` |
| Trabajos en cola | `./inamap.sh logs horizon` |
| Websockets | `./inamap.sh logs reverb` |
| Rutas registradas del dominio | `./inamap.sh artisan route:list --path=inamap` |
| Probar código | `./inamap.sh tinker` |
