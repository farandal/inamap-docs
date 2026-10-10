# 02 · Modos de acceso del visor

Cada inamap define quién puede abrir su visor público (`inamap-app`) en
**MAPA › Configuración › Modo de acceso** (`settings.access_mode`).

| Modo | Para qué sirve | Comportamiento |
|---|---|---|
| **`open`** (por defecto) | Mapas que cualquiera puede consultar: campus, ferias, centros comerciales | Cualquiera con el enlace ve el mapa, busca y calcula rutas. El token es opcional y solo sirve para llevar una ruta concreta al celular. |
| **`token_required`** | Recintos que no quieren su plano expuesto públicamente | Para ver el mapa, buscar o calcular rutas hace falta un **token de sesión** válido de ese inamap, que normalmente llega escaneando un QR del recinto. |

## Qué exige cada endpoint público

| Endpoint | `open` | `token_required` |
|---|---|---|
| `GET api/inamap/{id}` (dibujo del mapa) | Libre | `X-Inamap-Token` o `X-Inamap-Access-Key` |
| `GET api/inamap/search` | Libre | `X-Inamap-Token` o `X-Inamap-Access-Key` |
| `GET api/inamap/route` | Libre | `X-Inamap-Token` o `X-Inamap-Access-Key` |
| `POST api/inamap/route-sessions` | Libre | Credencial de dispositivo, administrador o sesión vigente |
| `GET api/inamap/route-sessions/current` | Token de la sesión | Token de la sesión |

Respuestas cuando falta o falla el token: `401 TOKEN_REQUIRED` (no se envió),
`401 TOKEN_EXPIRED` (venció) y `401 TOKEN_INVALID` (no existe o es de otro inamap).

## Cómo llega el token al visitante

1. Un kiosco con [credencial de dispositivo](01-credencial-de-dispositivo.md) crea una
   sesión de ruta y muestra el QR.
2. El QR abre `…/inamap/{id}#t=<token>`. El token va en el **fragmento** (`#`), que el
   navegador no envía al servidor ni queda en registros de acceso.
3. El visor lee el token y lo envía en `X-Inamap-Token` en cada llamada.

## Caché

En modo `open` el payload del mapa se sirve con `Cache-Control: public` y `ETag` (60 s). En
`token_required` se sirve como `private, no-store`, para que ninguna caché intermedia
entregue el plano a quien no tiene token.
