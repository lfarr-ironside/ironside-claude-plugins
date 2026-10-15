# ironside-db — consultar la base de reportes desde Claude

Permite preguntarle a Claude por las ventas, el stock y el catálogo de
Ironside en lenguaje natural.

**No hay que instalar nada.** Ni Node.js, ni Azure CLI, ni configurar un
`.env`. El servidor corre en Azure y la autenticación es con tu cuenta
Microsoft de Ironside.

## Instalación

Desde Claude Code o Cowork:

```
/plugin marketplace add https://github.com/lfarr-ironside/postgresql-mcp-connector.git
/plugin install ironside-db@ironside-mcp
```

Reiniciá Claude. La primera vez que uses el conector te va a pedir iniciar
sesión: se abre el navegador, entrás con tu cuenta de Ironside y listo.

## Requisito de acceso

Tu cuenta tiene que estar en el grupo **BBDD Reader Claude** de Entra. Si no
lo está, el login va a fallar con un error de permisos. Pedí el alta a
Leopoldo Farr (lfarr@ironside.cl).

## Qué podés consultar

| Vista | Contenido |
|---|---|
| `ventas` | Facturación: cliente, producto, cantidades, montos, estado de pago |
| `stock_bodega` | Unidades por bodega |
| `stock_ubicacion` | Unidades por ubicación física dentro de cada bodega |
| `productos` | Catálogo con precio de lista |
| `categorias` | Árbol de categorías |

Ejemplos de preguntas:

> ¿Cuánto vendimos en Chile en septiembre?

> ¿Qué productos tienen stock en la bodega de Marketing?

> Dame los 10 SKU más vendidos en México este año

## Qué NO incluye

Estas vistas **no exponen costos ni márgenes**. Son de solo lectura: no se
puede escribir en la base.

Si necesitás analizar rentabilidad, pedí acceso al equipo de datos; es otro
nivel de permisos, no algo que se pueda sacar desde acá.

## Si algo no funciona

**"No autorizado" o falla el login** — tu cuenta no está en el grupo
`BBDD Reader Claude`.

**Claude no encuentra las herramientas** — reiniciá Claude después de
instalar el plugin.

**Una consulta devuelve "permission denied"** — estás pidiendo una tabla
fuera de las vistas permitidas. Usá `listar_vistas` para ver cuáles hay.
