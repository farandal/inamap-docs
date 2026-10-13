# 04 · Comandos rápidos

Referencia para copiar y pegar. Backend: desde `inamap-backend-docker/`. Frontend: desde
`inamap-frontend-domain/`.

## Primera vez

```bash
# Backend
cd inamap-backend-docker
./inamap.sh init                 # luego editar .env → DASH_IMAGE=...
./inamap.sh up

# Frontend
cd ../inamap-frontend-domain
pnpm install
cp apps/inamap-web/.env.inamap-web.local.example       apps/inamap-web/.env.inamap-web.local
cp apps/inamap-system/.env.inamap-system.local.example apps/inamap-system/.env.inamap-system.local
cp apps/inamap-app/.env.example                         apps/inamap-app/.env.local
```

## Día a día

| Tarea | Comando |
|---|---|
| Levantar backend | `./inamap.sh up` |
| Detener backend (conserva datos) | `./inamap.sh down` |
| Estado de servicios | `./inamap.sh status` |
| Log de Laravel | `./inamap.sh logs` |
| Logs de colas / websockets / nginx | `./inamap.sh logs horizon` · `reverb` · `nginx` |
| Aplicar migraciones nuevas | `./inamap.sh migrate` |
| Datos de referencia (monedas, idiomas, plan de prueba) | `./inamap.sh seed:base` |
| Recargar rutas y config | `./inamap.sh restart` |
| Pruebas del backend | `./inamap.sh test` |
| Una prueba | `./inamap.sh test --filter=RouteSessionTest` |
| Rutas del dominio | `./inamap.sh artisan route:list --path=inamap` |
| Shell / Tinker | `./inamap.sh shell` · `./inamap.sh tinker` |
| Limpiar sesiones de ruta vencidas | `./inamap.sh artisan inamap:prune-route-sessions` |
| Panel de clientes | `pnpm dev:web` → http://localhost:5181 |
| Administración de plataforma | `pnpm dev:system` → http://localhost:5182 |
| Visor público | `pnpm dev:app` → http://localhost:5180 |
| Las tres apps (local) | `pnpm dev:all` |
| Backend público (api-dev / ws-dev .inamap.space) | `./inamap.sh tunnel:init` (una vez) · `./inamap.sh up --tunnel` |
| Frontends públicos (app-dev / web-dev / system-dev .inamap.space) | `pnpm tunnel:init` (una vez) · `pnpm dev:tunnel:all` |
| Pruebas / tipos del frontend | `pnpm test` · `pnpm typecheck` |
| Build completo del frontend | `pnpm build` |

## Direcciones locales

| Qué | Dónde |
|---|---|
| API | http://localhost:25200/api |
| Reverb | ws://localhost:27001 |
| Mailhog | http://localhost:25030 |
| PostgreSQL | `psql -h localhost -p 25434 -U im -d im_dev_db` |
| Redis | `redis-cli -p 25389` |
| Visor de un inamap | http://localhost:5180/inamap/{id} |

En modo túnel: API `https://api-dev.inamap.space/api`, Reverb `wss://ws-dev.inamap.space`, apps
`https://app-dev.inamap.space` · `web-dev` · `system-dev`.

## Credenciales de desarrollo

| Qué | Dónde está |
|---|---|
| Usuarios de prueba | `inamap-backend-docker/.env.app.local` (`SYSTEM_ADMIN_*`, `TENANT_ADMIN_*`, `NORMAL_USER_*`) |
| Contraseña de la base | `inamap-backend-docker/.env` (`DB_PASSWORD`) |
| Clave de websockets para el frontend | `REVERB_APP_KEY` de `.env.app.local` → `VITE_APP_SOCKETS_KEY` |

## Prohibido en desarrollo

```text
php artisan migrate:fresh     php artisan migrate:refresh     php artisan db:wipe
```

Para empezar de cero a propósito: `./inamap.sh down`, borrar `storage/pgsql-data` y
`storage/.seeded`, `./inamap.sh up`.
