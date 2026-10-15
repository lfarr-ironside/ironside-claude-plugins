# Plugins de Claude — Ironside

Marketplace interno para conectar Claude Code y Cowork con la base de
reportes de Ironside.

## Instalación

```
/plugin marketplace add lfarr-ironside/ironside-claude-plugins
/plugin install ironside-db@ironside-mcp
```

Reiniciá Claude. La primera vez te va a pedir iniciar sesión con tu cuenta
Microsoft de Ironside.

## Cuál de los dos

| | `ironside-db` | `azure-pg` |
|---|---|---|
| **Para quién** | cualquiera que necesite consultar datos | equipo de datos |
| **Hay que instalar** | nada | Node.js + Azure CLI (`az login`) |
| **Dónde corre el servidor** | en Azure | en tu máquina |
| **Qué ve** | 5 vistas de solo lectura | toda la base, según tus permisos |
| **Costos y márgenes** | no | sí |

Si solo necesitás consultar ventas, stock o productos, usá **`ironside-db`**:
no requiere instalar nada y se configura en dos comandos.

`azure-pg` queda para quienes necesitan acceso completo, incluidos costos y
márgenes, y están dispuestos a mantener Azure CLI configurado.

## Acceso

Para `ironside-db` hay que estar en el grupo **BBDD Reader Claude** de
Microsoft Entra. Pedir el alta a lfarr@ironside.cl.

El grupo controla todo: quien no está adentro no pasa el login, y quitar a
alguien del grupo le corta el acceso sin tocar la base de datos.

## Qué se puede consultar con ironside-db

| Vista | Contenido |
|---|---|
| `ventas` | Facturación: cliente, producto, cantidades, montos, estado de pago |
| `stock_bodega` | Unidades por bodega |
| `stock_ubicacion` | Unidades por ubicación dentro de cada bodega |
| `productos` | Catálogo con precio de lista |
| `categorias` | Árbol de categorías |

Son de **solo lectura** y **no incluyen costos ni márgenes**.

## Estructura del repo

```
.claude-plugin/marketplace.json   el catálogo que lee Claude
ironside-db/                       plugin remoto (recomendado)
azure-pg/                          plugin local (equipo de datos)
```

Cada plugin lleva su propio `.mcp.json`, sus skills y su README.

Reemplaza a `postgresql-mcp-connector`, que quedó archivado. Aquel repo era
privado, y por eso Claude no podía sincronizarlo: fallaba con "Error al
sincronizar el marketplace" sin decir que el problema era de permisos.
