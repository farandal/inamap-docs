# INAMAP — Documentación de funcionalidades

Documentación funcional y técnica de cada funcionalidad de INAMAP, escrita para quienes
configuran, integran o desarrollan sobre la plataforma. La especificación completa del
proyecto (arquitectura, modelo de datos, requerimientos) está en
[`inamap-backend-domain/README.md`](../inamap-backend-domain/README.md); esta sección
explica **cómo funciona y cómo se usa** cada pieza.

## Desarrollo

| # | Guía | Para quién |
|---|---|---|
| 01 | [Entorno local con Docker](desarrollo/01-entorno-docker.md) | Todo desarrollador: levantar el backend, puertos, comandos, problemas frecuentes |
| 02 | [Desarrollo del backend](desarrollo/02-backend.md) | Backend: estructura, agregar recursos, permisos, configuración, pruebas, depuración |
| 03 | [Desarrollo del frontend](desarrollo/03-frontend.md) | Frontend: apps, paquetes `ina-*`, variables, editor de mapas, i18n, verificación |
| 04 | [Comandos rápidos](desarrollo/04-comandos-rapidos.md) | Referencia para copiar y pegar |

## DevOps

| Guía | Para quién |
|---|---|
| [Entorno de producción (preview)](devops/preview-env.md) | Operación: backend en el servidor, frontends en Cloudflare Pages, despliegue automático por rama `production`, verificación, rollback y pendientes |

Repositorios del proyecto (carpetas hermanas):

| Repositorio | Contenido |
|---|---|
| `inamap-backend-domain` | Backend del dominio + especificación técnica (`README.md`) |
| `inamap-backend-docker` | Entorno local del backend (Docker) |
| `inamap-frontend-domain` | Apps `inamap-web`, `inamap-system`, `inamap-app` y paquetes `ina-*` |
| `inamap-documentation` | Esta documentación |

## Funcionalidades

| # | Funcionalidad | Estado del documento |
|---|---|---|
| 01 | [Credencial de dispositivo (kioscos y pantallas)](funcionalidades/01-credencial-de-dispositivo.md) | Completo |
| 02 | [Modos de acceso del visor](funcionalidades/02-modos-de-acceso.md) | Completo |
| 03 | [Sesiones de ruta y código QR](funcionalidades/03-sesiones-de-ruta-y-qr.md) | Completo |
| 03b | [Integración con Vanexa (kioscos con agente de voz)](funcionalidades/03-vanexa.md) | Completo |
| 04 | Editor de mapas: layouts, zonas, estructuras, ubicaciones | Pendiente |
| 05 | Caminos, enlaces y enlaces verticales | Pendiente |
| 06 | Cálculo de rutas y rutas accesibles | Pendiente |
| 07 | Búsqueda difusa y categorías | Pendiente |
| 08 | Diagnóstico de conectividad | Pendiente |
| 09 | Publicación, enlace público e inserción (iframe) | Pendiente |
| 10 | Planes, límites y velocidad peatonal | Pendiente |
| 11 | Roles y permisos | Pendiente |

## Convenciones

- Las rutas de API son relativas a la base de la API (local: `http://localhost:25200/api`).
- "Inamap" (en minúscula) es la entidad: un recinto con sus pisos y su geometría.
- Las capturas y nombres de botones corresponden a la app de gestión `inamap-web`.
