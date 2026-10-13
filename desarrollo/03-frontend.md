# 03 · Desarrollo del frontend (`inamap-frontend-domain`)

El frontend es un workspace pnpm aislado con tres apps y tres paquetes del dominio. Las apps de
gestión usan el framework **DashAdmin** (`@dashadmin/*`, publicado en npm); el visor público no.

| App | Para quién | Puerto local |
|---|---|---|
| `inamap-web` | Clientes (tenants): registro, inamaps, editor de mapas, categorías, usuarios, suscripción | 5181 |
| `inamap-system` | Administración de plataforma: tenancies, tenants, planes, usuarios, pasarelas | 5182 |
| `inamap-app` | Visitantes: visor público, búsqueda, rutas, QR | 5180 |

| Paquete | Contenido |
|---|---|
| `ina-maps` | Recursos DashAdmin (inamaps, categorías, reportes) y el editor de mapas. |
| `ina-viewer` | Dibujo del mapa en SVG, geometría, cliente de la API pública y visor. Sin react-admin. |
| `ina-common` | Piezas de plataforma compartidas: providers, páginas de cuenta, imagen del tenant. |

## 1. Requisitos

- Node.js ≥ 20.19 (recomendado 22 o 24) y pnpm 11: `corepack enable`.
- Backend levantado ([01 · Entorno Docker](01-entorno-docker.md)).

## 2. Primer arranque

```bash
cd inamap-frontend-domain
pnpm install

cp apps/inamap-web/.env.inamap-web.local.example       apps/inamap-web/.env.inamap-web.local
cp apps/inamap-system/.env.inamap-system.local.example apps/inamap-system/.env.inamap-system.local
cp apps/inamap-app/.env.example                         apps/inamap-app/.env.local
```

En los dos primeros completar `VITE_APP_SOCKETS_KEY` con `REVERB_APP_KEY` de
`inamap-backend-docker/.env.app.local`.

```bash
pnpm dev:web      # http://localhost:5181
pnpm dev:system   # http://localhost:5182
pnpm dev:app      # http://localhost:5180
```

Iniciar sesión con los usuarios de prueba del backend (`.env.app.local` del docker).

## 3. Variables de entorno

| App | Archivo | Claves principales |
|---|---|---|
| `inamap-web`, `inamap-system` | `apps/<app>/.env.<app>.<entorno>` (se elige con `CUSTOM_MODE`) | `VITE_APP_ADMIN_API_URL`, `VITE_APP_BACKEND_URL`, `VITE_APP_FRONTEND_URL`, `VITE_APP_SOCKETS_*`, `VITE_DEV_PORT`, `VITE_APP_RELEASE_STAGE` |
| `inamap-app` | `apps/inamap-app/.env.local` | `VITE_API_URL`, `VITE_DEV_PORT` |

Los scripts del `package.json` raíz ya pasan `CUSTOM_MODE=<app>.local`. Para otro entorno:
crear `apps/inamap-web/.env.inamap-web.staging` y compilar con
`INAMAP_ENV=staging pnpm build:web`.

`VITE_APP_RELEASE_STAGE=pre-release` muestra la portada de "próximamente" y pide código de
invitación al registrarse; `release` (por defecto) abre el registro.

## 4. Desarrollo

- **Local** (sin túnel): `pnpm dev:all` o `dev:app` / `dev:web` / `dev:system`.
- **Público** (Cloudflare Tunnel): `pnpm tunnel:init` una vez y `pnpm dev:tunnel:all` publica
  `app-dev` / `web-dev` / `system-dev` `.inamap.space` contra `api-dev` / `ws-dev`; requiere el backend con
  `./inamap.sh up --tunnel`. Los dos modos usan los mismos puertos (5180-5182): uno u otro. Ver
  `inamap-frontend-domain/TUNNELS.md`.

- En `dev`, los paquetes `ina-*` se resuelven a su **código fuente**
  (`vite-plugins/localPackageSrcAliases.mts`): editar un paquete se ve al instante en la app.
- Las apps DashAdmin cargan recursos con un **manifiesto** (`src/resources/WebPrivateResources.tsx`
  a nivel tenancy y `WebTenantPrivateResources.tsx` a nivel tenant, para el TenantSwitcher).
- Llamadas a la API autenticada desde código propio: `useInamapApi()` (`ina-maps`), basado en
  `useAxios()` de `dash-axios-hook`, que agrega el token y el `X-Tenant-Id` del tenant activo.

### Agregar un recurso al menú

1. Crear `packages/ina-maps/src/resources/<recurso>Resource.tsx` con su `schema` (campos) y la
   configuración del recurso (`model` = ruta de la API, `group: 'ina_maps.groups.maps'`, `roles`).
2. Agregarlo al arreglo de `inamapResources.tsx` (o crear un archivo propio y sumarlo a los
   manifiestos de las apps).
3. Agregar sus textos en `packages/ina-maps/src/i18n/{es,en}.ts`.
4. El backend debe tener la ruta y los permisos (ver [02 · Backend](02-backend.md)).

### Textos (i18n)

- Las claves viven en el paquete que las usa (`ina_maps.*` en `ina-maps`). Se registran solas al
  cargar el paquete (`i18n/register.ts`); no hay que tocar las apps.
- Textos propios de una app: `apps/<app>/src/i18n/{es,en}.tsx`.

### Contrato con `dash-admin`

Cada app DashAdmin debe proveer `src/DASHApp.tsx` y `src/assets/logo-horizontal.png` /
`login-back.png`, porque el framework los importa con el alias `@app`.

### MUI 9

- Sin "system props": `alignItems`, `justifyContent`, `fontWeight`, `textAlign`… van dentro de
  `sx`.
- En `TextField`, `InputProps` pasó a `slotProps={{ input: … }}`; en `ListItemText`,
  `primaryTypographyProps` pasó a `slotProps={{ primary: … }}`.
- Algunos íconos cambiaron de nombre (por ejemplo `DeleteOutlineOutlined`).

## 5. Editor de mapas

Código en `packages/ina-maps/src/components/editor/`:

| Archivo | Rol |
|---|---|
| `InamapEditor.tsx` | Orquesta todo: herramientas, edición, guardado, publicación. |
| `EditorSidebar.tsx` | Secciones MAPA / HERRAMIENTAS / CAPAS dentro del menú lateral de la app (portal). |
| `PropertiesPanel.tsx` | Propiedades del elemento seleccionado (drawer derecho). |
| `LayoutsPanel.tsx` | Selector flotante de pisos y gestión de layouts (drawer). |
| `SettingsDrawer.tsx` | Información y configuración del inamap (drawer). |
| `EditorDialogs.tsx` | Tipo de enlace vertical y "Probar ruta". |
| `useEditorDocument.ts` | Documento del piso con deshacer/rehacer. |

El dibujo lo hace `MapCanvas` (`ina-viewer`), el mismo que usa el visor público.

**Zonas solapadas.** Por defecto las zonas no pueden solaparse: al cerrar una zona que cruza otra
el editor avisa y no la crea, y las zonas en conflicto se resaltan en rojo. Cada inamap puede
permitirlo con el interruptor **Permitir zonas solapadas** (drawer *Configuración* →
`settings.allow_zone_overlap`, apagado por defecto); con él encendido el editor no valida ni
resalta cruces de zonas, la ayuda de la herramienta lo indica y el servidor tampoco las rechaza
(ver [02 · Backend](02-backend.md) §6.1).

## 6. Verificación

```bash
pnpm test             # pruebas de geometría (vitest)
pnpm typecheck        # tipos de ina-* e inamap-app
pnpm build            # paquetes + las tres apps
```

## 7. Problemas frecuentes

| Síntoma | Causa / solución |
|---|---|
| Pantalla en blanco, error de red en la consola | La API no responde: revisar `./inamap.sh status` y `VITE_APP_ADMIN_API_URL`. |
| Error de CORS (`No 'Access-Control-Allow-Origin' header`) | El origen no está permitido: agregarlo en `inamap-backend-docker/domain-config-layers/10-inamap-local/cors.php` y `./inamap.sh restart`. Los puertos 5180–5182 ya están incluidos. |
| Aparece una clave de traducción en vez del texto | Falta la clave en `i18n/{es,en}` del paquete o la app. |
| Las notificaciones en tiempo real no llegan | `VITE_APP_SOCKETS_KEY` distinto de `REVERB_APP_KEY`, o puerto de Reverb incorrecto. |
| El build falla resolviendo `ina-maps/...` | Ejecutar `pnpm build:packages` antes del build de producción. |
| `pnpm install` se queda detenido | Ejecutar en una terminal interactiva (puede pedir aprobar scripts de instalación). |
