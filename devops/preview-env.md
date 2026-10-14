# Entorno de producción (preview) de INAMAP

Documento de operación del entorno que sigue la rama `production`: backend en el servidor Linux
de preview y frontends en Cloudflare Pages, ambos desplegados **por el propio servidor** al
detectar cambios en la rama. Estado verificado el 2026-10-13.

> Es **independiente** del staging del Mac (`*-staging.inamap.space`, rama `development`, pm2).
> Otra máquina, otra rama, otro túnel, otra base de datos y otros proyectos de Pages.

## 1. Resumen

| Pieza | Valor |
|---|---|
| Servidor | `fablabadmin@172.20.20.80` (Ubuntu 24.04, red interna; comparte equipo con kitchntabs y vanexa) |
| Rama que sigue | `production` en `inamap-backend-domain`, `inamap-frontend-domain` e `inamap-backend-docker` |
| Supervisor | systemd: `dash-watcher` (despliegue) y `cloudflared-inamap` (túnel). No hay pm2 en este equipo |
| API | `https://api.inamap.space` → `localhost:25200` |
| WebSocket (Reverb) | `https://ws.inamap.space` → `localhost:27001` (interno `6001`) |
| Web (tenants) | `https://inamap.space` y `https://www.inamap.space` → Pages `inamap-web` |
| App (visor público) | `https://app.inamap.space` → Pages `inamap-app` |
| System (plataforma) | `https://system.inamap.space` → Pages `inamap-system` |
| Túnel Cloudflare | `inamap-preview-server` (`408e7be2-971e-4e34-94ec-818ea71f95b4`), dedicado |
| Cuenta / zona Cloudflare | cuenta `08c4599e3c05798074bf0fc00642c6d1`, zona `inamap.space` `f37399060fa74fdd9ae8fdd142ad0b77` |

## 2. Arquitectura

```text
                     GitHub  (rama production)
                         │  poll cada 60 s
                         ▼
        ┌──────────── dash-watcher (systemd) ────────────┐
        │ inamap-backend-domain  → pull, composer,       │
        │                          migrate, reinicio     │
        │ inamap-frontend-domain → pull, build ×3,       │
        │                          wrangler pages deploy │
        └──────────┬───────────────────────┬─────────────┘
                   ▼                       ▼
        inamap_app (Docker)        Cloudflare Pages
        nginx+php-fpm, Horizon,    inamap-web / -system / -app
        Reverb, scheduler          (rama de producción: production)
                   ▲                       │
 api./ws.inamap.space                      ▼
   túnel inamap-preview-server    inamap.space, www., app., system.
   (cloudflared-inamap)
```

Los navegadores cargan el frontend desde Pages y hablan directamente con
`api.inamap.space` (REST) y `ws.inamap.space` (WebSocket). El backend permite esos cuatro
orígenes por CORS (sección 5.3).

## 3. Ramas y promoción

`production` se creó el 2026-10-13 a partir de `development` en los tres repositorios. Hoy son
alias; **no hay barrera de liberación** hasta que `development` avance.

Promover = avance rápido (fast-forward), nunca merge ni force:

```bash
R=inamap-backend-domain            # o inamap-frontend-domain
git -C $R fetch -q origin
git -C $R log --oneline origin/development..origin/production   # debe salir vacío
git -C $R log --oneline origin/production..origin/development    # lo que se va a publicar
NEW=$(git -C $R rev-parse origin/development)
git -C $R push origin "${NEW}:refs/heads/production"
```

En zsh escribir `${NEW}` con llaves: `"$NEW:refs/..."` lo interpreta el modificador `:r`.
**Empujar a `production` despliega** en un minuto. Avisar antes si alguien está probando.

## 4. Qué se despliega solo y qué no

| Cambio en | Qué hace el servidor | Tiempo |
|---|---|---|
| `inamap-backend-domain` | `git pull --ff-only`, `composer install`, `migrate --force`, `optimize:clear` y **reinicio del contenedor** `inamap_app` | ~1 min de API caída |
| `inamap-frontend-domain` | `git pull --ff-only`, `pnpm install --frozen-lockfile`, build de los 3 paquetes/apps, `wrangler pages deploy` ×3 | varios minutos, sin caída |
| `dash-backend` (core) | Reconstruye `local/dash-backend-core:latest`; **no** recrea `inamap_app` | ver 8 |
| `inamap-backend-docker` | Nada: compose y capas de configuración se actualizan a mano | manual |
| Archivos `.env*` del servidor | Nada | manual |

Si un pull no es posible (otra rama, árbol sucio, divergencia) el watcher **omite** el repositorio
y lo anota como `WARN`; no resuelve nada solo. Si el árbol de `inamap-frontend-domain` queda
sucio el frontend deja de desplegarse: los builds de producción usan `--frozen-lockfile` y sus
archivos `.env` generados están en `.gitignore` justamente para evitarlo.

## 5. Backend

### 5.1 Contenedores y puertos

Proyecto compose `inamap` en `~/inamap-backend-docker`. Imagen del core:
`local/dash-backend-core:latest` (la misma que usan kitchntabs y vanexa en este servidor).

| Contenedor | Función | Puerto en el host |
|---|---|---|
| `inamap_app` | nginx + php-fpm, Horizon, Reverb, scheduler | `25200` (HTTP), `27001` (Reverb) |
| `inamap_pgsql` | PostgreSQL 17 (`im_dev_db`, usuario `im`) | `25434` |
| `inamap_pgsql_setup` | crea la base de pruebas | — |
| `inamap_redis` | colas, caché, broadcasting | `25389` |
| `inamap_mailhog` | correo de pruebas | `25029` / `25030` |

Todos con `restart: unless-stopped`. Los puertos no chocan con kitchntabs (`250xx`, `25432`,
`25379`) ni vanexa (`251xx`, `25433`, `25388`, `26001`).

### 5.2 Directorios del servidor (hermanos de `dash-backend-docker`)

| Ruta | Contenido |
|---|---|
| `~/inamap-backend-docker` | compose, `.env` y `.env.app.production` (modo `640`, **no** versionados) |
| `~/inamap-backend-domain` | dominio, montado en el contenedor; lo actualiza el watcher |
| `~/inamap-frontend-domain` | frontend; el watcher compila aquí |
| `~/dash-backend-docker` | watcher y scripts. **Está en la rama `development`** (ver 11) |
| `~/.local/node` | Node 22 + pnpm 11.17.0 de usuario, solo para compilar el frontend |
| `/etc/cloudflared/inamap-preview-token` | token del conector del túnel (modo `600`) |
| `/etc/systemd/system/cloudflared-inamap.service` | unidad del conector |

### 5.3 Configuración

`.env` (compose): `DASH_IMAGE`, `APP_ENV_FILE=./.env.app.production`, `DB_PASSWORD`, puertos.

`.env.app.production` (Laravel): `APP_ENV=local`, `APP_DEBUG=true`, `LOG_LEVEL=debug` (igual que
los demás entornos previos a producción), `APP_URL=https://api.inamap.space`,
`REVERB_HOST=ws.inamap.space`, `REVERB_PORT=443`, `REVERB_SCHEME=https`,
`REVERB_APP_ID/KEY/SECRET` replicados en `PUSHER_APP_*`, `FRONTEND_URL=https://inamap.space`,
`INAMAP_APP_URL=https://app.inamap.space` (destino de enlaces y QR).

Los secretos (`APP_KEY`, contraseña de la base, claves de Reverb, contraseñas de los usuarios
sembrados) se generaron en el servidor y viven solo en esos dos archivos. Correos sembrados:
`admin@inamap.local`, `tenant@inamap.local`, `user@inamap.local`.

Reglas para editarlos:

- **Reescribir en el lugar** (`cat nuevo > archivo`). `sed -i` cambia el inodo y el contenedor
  sigue leyendo el archivo viejo (es un bind mount de un solo archivo). Luego `docker restart inamap_app`.
- Modo **`640`**. Con `600` PHP no lo lee (500 "No application encryption key"); `644` expone secretos.
- No dejar `DB_DATABASE_TEST` en blanco: Compose lo reemplaza por un valor por defecto y
  `pgsql_setup` queda sin salud con `app` en `Created`.

**CORS.** El core fija sus orígenes permitidos en `config/cors.php`; los dominios los amplían con
capas en `domain-config-layers/`. Los orígenes de producción están en una capa **local del
servidor**, fuera de git (`.git/info/exclude`), como los `.env*`:

```text
~/inamap-backend-docker/domain-config-layers/90-server-production/cors.php
  → https://inamap.space, https://www.inamap.space, https://app.inamap.space, https://system.inamap.space
```

Si se agrega otro hostname de frontend hay que editar ese archivo y reiniciar `inamap_app`.

## 6. Frontend

### 6.1 Proyectos de Cloudflare Pages

Subida directa (sin integración Git de Cloudflare). Rama de producción de cada proyecto:
`production`.

| Proyecto | Dominios personalizados | App |
|---|---|---|
| `inamap-web` | `inamap.space`, `www.inamap.space` | `apps/inamap-web` (tenants) |
| `inamap-system` | `system.inamap.space` | `apps/inamap-system` (plataforma) |
| `inamap-app` | `app.inamap.space` | `apps/inamap-app` (visor público) |

Cada dominio tiene un CNAME proxied en la zona hacia `<proyecto>.pages.dev` (el ápice usa
CNAME flattening de Cloudflare). Los SPA resuelven rutas profundas con `public/_redirects`
(`/* /index.html 200`).

### 6.2 Cómo se compila para producción

`scripts/inamap-frontend-deploy.js` (en `dash-backend-docker`) elige un **perfil** con
`INAMAP_FRONTEND_ENV`; sin la variable usa `staging`, así que el Mac no cambia.

| | `staging` (Mac) | `production` (este servidor) |
|---|---|---|
| Rama | `development` | `production` |
| Proyectos Pages | `inamap-{web,system,app}-staging` | `inamap-{web,system,app}` |
| Archivos `.env` de las apps | existen en el Mac (gitignored) | **se generan en cada ejecución** |
| Credenciales Cloudflare | `.env.vanexa` | `.env.inamap-pages` |
| `pnpm install` | normal | `--frozen-lockfile` |

En producción el script escribe, a partir de los hostnames de la sección 1 y de
`REVERB_APP_KEY` leída de `~/inamap-backend-docker/.env.app.production`:

```text
apps/inamap-web/.env.inamap-web.production
apps/inamap-system/.env.inamap-system.production
apps/inamap-app/.env.production                      (VITE_API_URL=https://api.inamap.space/api)
```

Variables relevantes de web/system: `VITE_APP_BACKEND_URL`, `VITE_APP_ADMIN_API_URL`,
`VITE_APP_INAMAP_APP_URL`, `VITE_APP_SOCKETS_HOST/PORT/SCHEME/KEY`. La clave de Reverb es la
clave **pública** que el navegador envía, no el secreto. `VITE_APP_RELEASE_STAGE` queda en
`release`; se cambia con `INAMAP_RELEASE_STAGE=pre-release` (portada "próximamente" e invitación
al registrarse).

Otras variables del script: `INAMAP_FRONTEND_BUILD_ONLY=1` (compila sin publicar ni pedir
credenciales), `INAMAP_FRONTEND_DIR`, `INAMAP_BACKEND_ENV_FILE`, `INAMAP_TOOLCHAIN_BIN`.

Compilar tres apps Vite no es liviano: se ejecuta con `nice -n 10` y
`NODE_OPTIONS=--max-old-space-size=3072`. En el primer build del servidor (instalación de
dependencias en frío) la memoria disponible no bajó de ~9 GB y los contenedores vecinos no se
vieron afectados.

### 6.3 Reintentos y estado

Al terminar bien, el script anota el commit publicado en
`~/dash-backend-docker/.inamap-frontend-production.sha` (ignorado por git). El watcher despliega
cuando la rama se mueve **o** cuando ese archivo no coincide con `HEAD` (publicación fallida o
pendiente), con espera de 15 min tras un fallo para no recompilar cada minuto. Sin archivo de
estado solo despliega ante un commit nuevo.

### 6.4 Token de Cloudflare (pendiente de colocar)

El servidor **no** tiene aún una credencial de Cloudflare Pages: el primer lanzamiento se
publicó desde la estación de trabajo y el servidor tiene sin llenar
`~/dash-backend-docker/.env.inamap-pages`. Sin token, un cambio nuevo hace que el watcher falle
rápido (`missing CF_API_TOKEN`, sin compilar) y reintente cada 15 min.

Crear un token **solo para Pages** (no reutilizar el de staging, que además toca DNS y túneles):

1. Cloudflare → My Profile → API Tokens → Create Token → *Create Custom Token*.
2. Permisos: **Account → Cloudflare Pages → Edit**. Nada más (ni zona, ni DNS).
3. Account Resources: *Include → la cuenta de INAMAP*.
4. Colocarlo en el servidor, sin dejarlo en el historial:

   ```bash
   ssh fablabadmin@172.20.20.80
   cd ~/dash-backend-docker && read -rs T && printf 'CF_ACCOUNT_ID=08c4599e3c05798074bf0fc00642c6d1\nCF_API_TOKEN=%s\n' "$T" > .env.inamap-pages; unset T
   chmod 600 .env.inamap-pages
   ```

5. Probar sin esperar un commit:
   `set -a; . scripts/systemd/dash-watcher.env; set +a; node scripts/inamap-frontend-deploy.js`

## 7. Configuración del watcher

`~/dash-backend-docker/scripts/systemd/dash-watcher.env` (unidad `dash-watcher`):

```text
DASH_WATCHER_BRANCH=production
INAMAP_FRONTEND_ENV=production
INAMAP_TOOLCHAIN_BIN=/home/fablabadmin/.local/node/bin
#DASH_WATCHER_SKIP_FRONTEND=1     # desactiva la publicación del frontend
```

El archivo se lee al **iniciar** el proceso: tras cambiarlo o tras actualizar el código del
watcher, `sudo systemctl restart dash-watcher`.

Proyectos activos: `~/dash-backend-docker/.env.projects` con
`ACTIVE_PROJECTS=kitchntabs,vanexa,inamap` (se lee en cada ciclo). Para liberar recursos quitar
`inamap` de la lista y hacer `docker compose stop` en `~/inamap-backend-docker`. No usar
`scripts/toggle-project.sh` en este servidor: termina en `pm2 save`.

## 8. Operación

```bash
ssh fablabadmin@172.20.20.80

# estado general
systemctl is-active dash-watcher cloudflared-inamap
docker ps --format 'table {{.Names}}\t{{.Status}}' | grep inamap
free -h
journalctl -u dash-watcher -n 40 --no-pager         # "no changes" = al día u omitido (buscar WARN)
journalctl -u dash-watcher -f                       # seguir un despliegue

# backend
cd ~/inamap-backend-docker
docker logs -f inamap_app
docker compose exec -u dash app php artisan migrate:status
docker compose exec -u dash app tail -f storage/logs/laravel.log

# frontend: publicar a mano el commit actual de production
cd ~/dash-backend-docker && set -a && . scripts/systemd/dash-watcher.env && set +a
node scripts/inamap-frontend-deploy.js
```

Usar siempre `-u dash` con `artisan`: como root crea archivos que php-fpm no puede escribir.

**Actualizar la imagen del core.** El watcher reconstruye `local/dash-backend-core:latest` pero
no recrea `inamap_app` (usa una imagen distinta de la de los otros proyectos). Para tomarla:
`cd ~/inamap-backend-docker && docker compose up -d --force-recreate app`.

## 9. Verificación

```bash
# backend
curl -s -o /dev/null -w '%{http_code}\n' https://api.inamap.space/        # 200
curl -s -o /dev/null -w '%{http_code}\n' https://ws.inamap.space/         # 404 (correcto)
curl -s --http1.1 -i -H 'Connection: Upgrade' -H 'Upgrade: websocket' \
  -H 'Sec-WebSocket-Version: 13' -H 'Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==' \
  "https://ws.inamap.space/app/<REVERB_APP_KEY>?protocol=7" | head -1     # 101

# CORS: debe devolver access-control-allow-origin con el mismo origen
curl -s -o /dev/null -D - -X OPTIONS https://api.inamap.space/api/login \
  -H 'Origin: https://system.inamap.space' -H 'Access-Control-Request-Method: POST' | grep -i allow-origin

# frontends
for u in inamap.space www.inamap.space app.inamap.space system.inamap.space; do
  curl -s -o /dev/null -w "$u %{http_code}\n" https://$u/; done
```

El WebSocket se prueba por **HTTP/1.1**: por HTTP/2 el handshake devuelve 500 y parece una caída.
Una app recién creada o reiniciada responde 502 cerca de un minuto (el entrypoint migra antes de
iniciar nginx).

Verificado el 2026-10-13:

- 150 migraciones aplicadas, ninguna pendiente; seed ejecutado (4 usuarios, 1 tenant).
- `POST /api/login` con el administrador sembrado devuelve token.
- Los cuatro hostnames responden 200, con *fallback* de SPA en rutas profundas; los dominios
  personalizados de Pages figuran `active`.
- Carga en navegador sin errores de consola: `inamap.space` (portada con Ingresar/Registrarse),
  `system.inamap.space` (formulario de ingreso) y `app.inamap.space` (visor con campo de enlace/QR).
- Los bundles solo referencian `api.inamap.space` y `ws.inamap.space`: ni staging, ni dev, ni localhost.
- Preflight CORS permitido para los cuatro orígenes y sin cabecera para uno ajeno.

## 10. Rollback

No hay rollback automático ni se revierten migraciones.

1. **Frontend:** en Cloudflare Pages, proyecto → Deployments → *Rollback to this deployment*
   (inmediato). Para que no vuelva a avanzar, revertir el commit en `development`, promoverlo y
   dejar que el watcher publique. Ojo: el watcher publica de nuevo si la rama sigue por delante
   del estado anotado.
2. **Backend:** el watcher haría avance rápido hasta el commit malo de nuevo. Detenerlo primero
   (`sudo systemctl stop dash-watcher`), corregir con un commit de reversión en `development`,
   promoverlo y reanudar. Las migraciones se deshacen a mano si hace falta.

## 11. Instalación desde cero (resumen reproducible)

1. **Ramas** `production` en los tres repositorios (sección 3).
2. **Toolchain** (Node 22 + pnpm 11.17.0 sin sudo; el Node del sistema es 18):
   descargar el tarball de `nodejs.org/dist/latest-v22.x/` verificando `SHASUMS256.txt`,
   extraer en `~/.local/node` y `npm install -g pnpm@11.17.0`.
3. **Repositorios** hermanos de `dash-backend-docker`, en `production`, con la misma forma de
   remoto que ya usan los demás clones de este servidor.
4. **Backend:** `cp .env.example .env`; `cp .env.app.example .env.app.production`; generar
   secretos (`openssl rand`); `chmod 640`; `docker compose up -d`; chown de
   `storage/app-storage` a `1000:33` (sin tocar `pgsql-data`); `php artisan db:seed`.
   La primera descarga de `postgres:17` tarda varios minutos.
5. **Túnel** `inamap-preview-server` por la API de Cloudflare (ingress `api.` → `:25200`,
   `ws.` → `:27001`, `404` final), CNAME proxied por hostname, token del conector en
   `/etc/cloudflared/inamap-preview-token` y la unidad `cloudflared-inamap`. Un solo conector
   por túnel: dos producen 502 intermitentes.
6. **Pages:** `POST /accounts/<cuenta>/pages/projects` con `{"name":..., "production_branch":"production"}`;
   `POST .../pages/projects/<p>/domains` con `{"name":"<hostname>"}`; CNAME proxied del
   hostname a `<p>.pages.dev`.
7. **CORS:** capa local `90-server-production` (sección 5.3) y reinicio de `inamap_app`.
8. **Watcher:** variables de la sección 7, `.env.inamap-pages` (sección 6.4) y
   `sudo systemctl restart dash-watcher`.

## 12. Decisiones y problemas encontrados

- **Imagen del core.** Se usa `local/dash-backend-core:latest` y no `farandal/dash-backend:latest-core`:
  es más reciente y es la que ya corren kitchntabs y vanexa. Se verificó que las 15 clases del core
  que importa el dominio de inamap existen en la rama `production` de `dash-backend`.
- **El core de `production` no tiene `App\IntegrationCore`** ni el complemento de plan
  `integration_inamap` (22 commits detrás de `development`). El dominio de inamap no los usa hoy;
  las integraciones externas no funcionarán aquí hasta promover el core y reconstruir la imagen.
- **`dash-backend-docker` del servidor está en `development`**, no en `production`: el
  watcher y el script de publicación se desarrollan allí y el repositorio no lo vigila el watcher
  (es la elección manual que documenta `SETUP.md` §2.2). Volver: `git checkout production`
  una vez promovidos esos commits.
- **Estado del despliegue fuera de `storage/`.** `storage/` del servidor pertenece al usuario del
  contenedor; el archivo `.inamap-frontend-production.sha` vive junto a los scripts.
- **El Node del sistema (18) no sirve para compilar** (el repositorio exige ≥ 20.19). El script
  antepone `INAMAP_TOOLCHAIN_BIN` al `PATH` solo de sus procesos hijos; el watcher sigue en el Node 18.
- **php-fpm no se recarga solo** (`opcache.validate_timestamps=Off`): por eso el watcher reinicia
  el contenedor tras cada despliegue de dominio, con ~1 min de API caída aceptado.
- **El perfil por defecto del script es `staging`.** Con el frontend activo en el watcher y sin
  `INAMAP_FRONTEND_ENV=production` (ni `DASH_WATCHER_SKIP_FRONTEND=1`), un commit nuevo habría
  publicado sobre los proyectos de staging. Por eso ambas variables están documentadas juntas.
- **Memoria.** El servidor tiene 15 GB y se recomiendan 32 GB para dos stacks completos. Con los
  tres stacks quedan ~10 GB disponibles en reposo y ~9 GB durante un build del frontend.

## 13. Pendiente

1. **Token de Pages en el servidor** (sección 6.4): hasta entonces los cambios de frontend no se
   publican solos. El resto del flujo está probado.
2. **Rotar la contraseña SSH** de `fablabadmin`: se compartió por chat durante la instalación.
3. **Token de Cloudflare propio** para el túnel y los DNS: se crearon con el de staging desde la
   estación de trabajo (el servidor solo guarda el token del conector del túnel).
4. **Promover `dash-backend`** a `production` si se necesitan integraciones externas (sección 12).
5. **Alinear `dash-backend-docker`**: promover `development` a `production` o decidir que el
   servidor siga en `development`.
6. **Prueba de reinicio** (`sudo reboot`): `inamap_app` y `cloudflared-inamap` deben volver sin
   sesión iniciada. Contenedores y unidades ya están configurados para ello.
7. **Barrera de liberación:** `production` es alias de `development`; decidir cuándo empieza a
   promoverse de forma deliberada.
