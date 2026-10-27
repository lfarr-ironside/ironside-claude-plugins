# ironside-db-full — acceso completo a la base de reportes

Permite consultar **todas las tablas** de la base de reportes de Ironside en
lenguaje natural, incluidos costos y márgenes.

**No hay que instalar nada.** Ni Node.js, ni Azure CLI, ni configurar un
`.env`. El servidor corre en Azure y la autenticación es con tu cuenta
Microsoft de Ironside.

## ¿Este o `ironside-db`?

| | `ironside-db` | `ironside-db-full` |
|---|---|---|
| **Qué ve** | 6 vistas de solo lectura | las 30 tablas de `public` |
| **Costos y márgenes** | no | **sí** |
| **Compras y proveedores** | no | sí |
| **Apuntes contables** | no | sí |
| **Quién** | grupo `BBDD Reader Claude` | grupo `BBDD Reader Full Claude` |

Si solo necesitas consultar ventas, stock o productos, usa `ironside-db`:
es más simple y sus vistas ya vienen con los nombres ordenados.

Este plugin es para quienes necesitan analizar rentabilidad o cruzar con
compras, contabilidad e inventario valorizado.

## Instalación

```
/plugin marketplace add lfarr-ironside/ironside-claude-plugins
/plugin install ironside-db-full@ironside-mcp
```

Reinicia Claude. La primera vez te va a pedir iniciar sesión con tu cuenta
Microsoft de Ironside.

## Requisito de acceso

Tu cuenta tiene que estar en el grupo **BBDD Reader Full Claude** de Entra.
Si no lo está, el login falla con un error de permisos. Pide el alta a
lfarr@ironside.cl.

El grupo controla todo: quien no está adentro no pasa el login, y sacar a
alguien del grupo le corta el acceso sin tocar la base de datos.

## Qué se puede consultar

Facturación, presupuesto, catálogo con costos, stock por bodega y por
ubicación, movimientos de inventario, valorización de existencias, órdenes
de compra, proveedores, facturas de proveedores, costos de internación y
apuntes contables.

Ejemplos:

> ¿Cuál fue el margen por categoría en Chile en septiembre?

> ¿Cómo vamos contra presupuesto este trimestre, por canal?

> ¿Qué productos tienen más inventario valorizado sin rotación?

## Lo que NO permite

Es **solo lectura**. La base rechaza cualquier intento de escritura: no es
una validación del servidor, es que el rol no tiene el permiso.

Los esquemas `inv_model` y `pricing` quedan fuera deliberadamente.

## Si algo no funciona

**"No autorizado" o falla el login** — tu cuenta no está en el grupo
`BBDD Reader Full Claude`.

**Claude no encuentra las herramientas** — reinicia Claude después de
instalar el plugin.

**Una consulta devuelve "permission denied"** — estás pidiendo algo fuera
del esquema `public`. Usa `listar_vistas` para ver qué hay.

## Una advertencia

Este acceso incluye información sensible: costos, márgenes, condiciones con
proveedores y contabilidad. Lo que salga de acá merece el mismo cuidado que
un informe financiero.
