---
description: |
  Use this skill when the user asks about Ironside's sales, inventory, stock,
  products or categories. Trigger phrases: "cuánto vendimos", "ventas de",
  "qué stock hay", "inventario", "cuántas unidades", "qué productos",
  "muéstrame las ventas", "consulta la base", "cuánto facturamos",
  "stock en bodega", "sales data", "how much did we sell", "inventory",
  or any natural-language question about Ironside business data.
---

# Consultar datos de Ironside

Las herramientas del servidor `ironside-db` consultan la base de reportes.
Antes de responder cualquier pregunta de negocio, llamá a `contexto_negocio`:
trae las reglas que hay que aplicar y evita respuestas que parecen correctas
y no lo son.

## Cómo trabajar

1. `listar_vistas` — qué hay disponible
2. `describir_vista` — las columnas de la que vayas a usar
3. `consultar_sql` — la consulta

Las vistas están en el esquema `cowork`, que ya es el search_path, así que
se nombran directo: `SELECT ... FROM ventas`.

## Reglas que hay que aplicar siempre

Estas tres cambian el resultado y nadie las va a pedir explícitamente:

**Montos sin impuestos.** Usá `venta_neta`. Solo usá `venta_con_impuestos`
si preguntan por el monto con IVA.

**Contar facturas con DISTINCT.** Una factura ocupa una fila por producto,
así que `COUNT(*)` cuenta líneas, no facturas. Siempre
`COUNT(DISTINCT movimiento)`.

**Las notas de crédito ya vienen en negativo.** No hay que excluirlas ni
restarlas aparte: `SUM(venta_neta)` las netea solo. Si alguien pide "ventas
sin devoluciones", ahí sí filtrá `tipo_movimiento = 'out_invoice'`.

## Monedas: no se suman entre países

Cada compañía factura en su moneda y los montos **no están convertidos**:

| compania | País | Moneda |
|---|---|---|
| IRONSIDE SPA | Chile | CLP |
| IRONSIDE FITNESS SA | Argentina | ARS |
| IRONSIDE FITNESS SA DE CV | México | MXN |

Sumar las tres da un número sin sentido. Si piden un consolidado, decí que
hace falta un tipo de cambio y pedí cuál usar; no inventes uno.

## Lo que no está disponible

Estas vistas **no tienen costos ni márgenes**, por diseño del acceso. Si la
pregunta los necesita —rentabilidad, margen por producto, valorización de
inventario— explicá que no están en este acceso y que hay que pedirlos al
equipo de datos. No intentes derivarlos ni estimarlos.

## Al presentar resultados

Decí siempre de qué período y de qué compañía son los números. Un total sin
contexto se malinterpreta fácil, sobre todo si el usuario asume que incluye
los tres países y en realidad es uno.
