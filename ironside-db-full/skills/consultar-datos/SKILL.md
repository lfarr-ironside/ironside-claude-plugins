---
description: |
  Use this skill when the user asks about Ironside's sales, inventory, stock,
  products, categories, budget, margins, costs, purchases, suppliers,
  accounting entries or landed costs. Trigger phrases: "cuánto vendimos",
  "ventas de", "margen", "rentabilidad", "costo", "qué stock hay",
  "inventario", "cuánto facturamos", "contra presupuesto", "cumplimiento",
  "compras", "proveedores", "apuntes contables", "costos de internación",
  "sales data", "margin", "profitability", "how much did we sell", or any
  natural-language question about Ironside business data.
---

# Consultar la base de Ironside — acceso completo

Las herramientas de `ironside-db-full` consultan las tablas completas del
esquema `public`. **Incluye costos y márgenes.**

Antes de responder cualquier pregunta de negocio, llama a
`contexto_negocio`: trae las reglas que hay que aplicar y evita respuestas
que parecen correctas y no lo son.

## Cómo trabajar

1. `listar_vistas` — qué tablas hay disponibles
2. `describir_vista` — las columnas de la que vayas a usar
3. `consultar_sql` — la consulta

El `search_path` ya apunta a `public`, así que las tablas se nombran
directo: `SELECT ... FROM invoices`.

Es solo lectura: la base rechaza cualquier escritura.

## Las tablas principales

| Tabla | Contenido |
|---|---|
| `invoices` | Facturación, una fila por producto facturado |
| `presupuesto_venta` | Meta de venta diaria por compañía y área |
| `products` | Catálogo, con costo estándar y FOB |
| `product_stock` | Stock por bodega |
| `product_stock_locations` | Stock por ubicación, valorizado |
| `accounting_entries` | Apuntes contables |
| `stock_move_lines` | Historial de movimientos de stock |
| `stock_valuation_layers` | Valorización de existencias |
| `purchase_orders` / `purchase_order_lines` | Órdenes de compra |
| `suppliers` / `supplier_invoices` | Proveedores y sus facturas |
| `landed_costs` y sus líneas | Costos de internación |

Hay más: usa `listar_vistas` para el listado completo.

## Cuatro reglas que cambian el resultado

No son preferencias de estilo: ignorarlas da números que parecen correctos y
no lo son.

**Montos sin impuestos.** Usa `subtotal`. `total` incluye IVA.

**Contar facturas con DISTINCT.** Una factura ocupa una fila por producto,
así que `COUNT(*)` cuenta líneas. Siempre `COUNT(DISTINCT movimiento)`.

**Las notas de crédito ya vienen en negativo.** No hay que excluirlas ni
restarlas: `SUM(subtotal)` las netea solo. Si piden ventas *sin*
devoluciones, filtra `tipo_movimiento = 'out_invoice'`.

**Las monedas no se suman entre países.** Cada compañía factura en la suya y
los montos no están convertidos:

| compania | País | Moneda |
|---|---|---|
| IRONSIDE SPA | Chile | CLP |
| IRONSIDE FITNESS SA | Argentina | ARS |
| IRONSIDE FITNESS SA DE CV | México | MXN |

Sumar las tres da un número sin significado. Un consolidado necesita un tipo
de cambio explícito y acordado; no lo inventes.

## Costos y márgenes

`invoices` trae `margen` y `valor_inventario`. **El costo de venta se deriva
como `subtotal - margen`**, no como `valor_inventario`: esa columna es
inconsistente entre países —negativa en Chile y Argentina, por unidad en
México, e invertida en las notas de crédito—.

Al presentar rentabilidad, aclara siempre de qué compañía y período son los
números, y recuerda que esta información es sensible.

## Presupuesto

`presupuesto_venta` tiene la meta diaria, por `compania` y `area` (que es el
mismo valor que `equipo_ventas` en `invoices`).

**Compara períodos equivalentes.** El presupuesto cubre el año entero, así
que sumar la meta anual contra la venta transcurrida da un cumplimiento
falso y siempre bajo. Acota los dos lados al mismo rango.

**No lo mires día a día.** La meta diaria no es plana: sigue una curva con
los eventos de cada país —CyberDay en Chile, Buen Fin y Hot Sale en México,
CyberMonday y Black Friday en Argentina—. Agrega por semana o por mes.

## Trampas conocidas de los datos

**Al unir `invoices` con `products`, usa LEFT JOIN y nunca INNER.** El 12,9%
de las filas de facturación tiene un SKU que no está en el catálogo: son
códigos históricos, bajas y artefactos de la tienda. El más frecuente es
`dispatch-cost-lp`, que es costo de despacho y no un producto. Un `INNER
JOIN` borra una octava parte de la facturación sin avisar.

**Hay SKU repetidos en `products`.** Unir por SKU multiplica esas filas e
infla los totales. Si solo necesitas el nombre o la categoría, ya están
dentro de `invoices`.

**Artefactos de la tienda entre los más vendidos.** `SHPE`,
`shopifydiscountproduct` y `shopifyshippingproduct` son envío y descuentos,
no productos. En un ranking por unidades aparecen arriba; conviene
excluirlos.

**556 filas incompletas.** Entre abril de 2025 y febrero de 2026 hay 556
filas sin `tipo_movimiento`, `estado_pago`, `cantidad` ni `vendedor`. Tienen
monto, así que entran en los totales pero quedan fuera de cualquier corte
por esas columnas.

**Las tablas de stock son una foto, no un histórico.** `product_stock` y
`product_stock_locations` se reemplazan en cada actualización. Para ver
evolución, usa `stock_move_lines`.

**México antes de marzo de 2026 es historia migrada**, no viene de Odoo. Los
campos del CFDI (`l10n_mx_edi_cfdi_uuid`) están vacíos ahí por diseño.

## Al presentar resultados

Di siempre de qué período y de qué compañía son los números. Un total sin
contexto se malinterpreta fácil, sobre todo si quien pregunta asume que
incluye los tres países y en realidad es uno.
