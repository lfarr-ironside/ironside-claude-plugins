# Contexto de negocio — Ironside Reports

## Schemas verificados

### Tabla `invoices`
| Columna | Tipo | Descripción |
|---------|------|-------------|
| id | bigint | Clave primaria |
| movimiento | varchar | Número de factura/documento (puede repetirse — ver "Facturas aperturadas") |
| fecha_factura | date | Fecha de emisión |
| fecha_vencimiento | date | Fecha de vencimiento del pago |
| tipo_movimiento | varchar | `out_invoice` = factura, `out_refund` = nota de crédito |
| compania | varchar | Empresa (ver tabla de países abajo) |
| equipo_ventas | varchar | Canal/equipo de venta (ver tabla abajo) |
| vendedor | text | Nombre del vendedor |
| sku | text | Código de producto |
| producto | varchar | Nombre del producto |
| categoria_producto | varchar | Categoría del producto |
| cantidad | numeric | Cantidad de unidades |
| subtotal | numeric | Subtotal sin impuestos |
| total | numeric | Total con impuestos (negativo en notas de crédito) |
| margen | numeric | Margen de la línea |
| valor_inventario | numeric | Costo de inventario |
| estado_pago | varchar | Estado de pago |
| estado_facturacion | varchar | Estado de facturación |
| contacto_principal | varchar | Cliente |
| diario | varchar | Diario contable |

### Tabla `presupuesto_venta`
| Columna | Tipo | Descripción |
|---------|------|-------------|
| id | bigint | Clave primaria |
| fecha | date | Fecha del presupuesto |
| monto_neto | numeric | Monto proyectado |
| compania | varchar | Empresa (mismos valores que `invoices`) |
| area | varchar | Equipo de venta (mismos valores que `equipo_ventas` en invoices) |

---

## Países y empresas

| Valor en `compania` | País |
|---------------------|------|
| `IRONSIDE SPA` | Chile |
| `IRONSIDE FITNESS SA` | Argentina |
| `IRONSIDE FITNESS SA DE CV` | México |

---

## Equipos de venta

| Valor en `equipo_ventas` / `area` | Tipo | País |
|-----------------------------------|------|------|
| `ECOM (CL)` | Ecommerce | Chile |
| `Ventas Ironside Chile` | Comercial dedicado | Chile |
| `ECOM (ARG)` | Ecommerce | Argentina |
| `Ironside Argentina` | Comercial dedicado | Argentina |
| `ECOM` | Ecommerce | México |
| `Ventas` | Comercial dedicado | México |

---

## Facturas aperturadas (estructura por línea de producto)

La tabla `invoices` está **aperturada por producto**: cada fila representa una línea de un documento, no un documento completo. Un mismo número de `movimiento` puede aparecer varias veces, una por cada producto incluido en esa factura.

**Consecuencias prácticas:**
- Para contar facturas únicas: `COUNT(DISTINCT movimiento)`
- Para sumar ventas totales: `SUM(total)` agrupando como corresponda
- Para ver el detalle de una factura: `WHERE movimiento = 'X'`
- Nunca usar `COUNT(*)` para contar facturas — siempre usar `COUNT(DISTINCT movimiento)`

---

## Notas de crédito mezcladas con facturas

Las notas de crédito conviven en la tabla `invoices`. Se distinguen por:
- `tipo_movimiento = 'out_refund'` → nota de crédito
- `tipo_movimiento = 'out_invoice'` → factura normal
- `tipo_movimiento` puede ser NULL (`NaN`) en algunos registros

El campo `total` puede ser negativo tanto en `out_refund` como ocasionalmente en `out_invoice` (devoluciones o ajustes).

**Para análisis de ventas netas**, incluir ambos tipos y dejar que los negativos descuenten:
```sql
SELECT SUM(total) FROM invoices
WHERE fecha_factura BETWEEN '2024-01-01' AND '2024-12-31'
```

**Para excluir notas de crédito** y ver solo facturación bruta:
```sql
WHERE tipo_movimiento = 'out_invoice'
```

---

## Tabla `presupuesto_venta` — solo proyección financiera

`presupuesto_venta` es un indicador financiero puro. Representa la proyección de venta para una fecha, independientemente de:
- Si esa fecha ya pasó
- Si se alcanzó o no la meta
- Si hay o no venta real registrada

**No debe usarse como histórico de ventas.** Para comparar real vs presupuesto, cruzar con `invoices`:
```sql
SELECT
  p.fecha,
  p.compania,
  p.area,
  p.monto_neto AS presupuesto,
  COALESCE(SUM(i.total), 0) AS venta_real
FROM presupuesto_venta p
LEFT JOIN invoices i
  ON i.fecha_factura = p.fecha
  AND i.compania = p.compania
  AND i.equipo_ventas = p.area
GROUP BY p.fecha, p.compania, p.area, p.monto_neto
ORDER BY p.fecha, p.compania
```

La columna de join es: `presupuesto_venta.area` ↔ `invoices.equipo_ventas`

---

## Montos: siempre ventas netas (sin IVA)

**Regla por defecto:** usar siempre `subtotal` para reportar ventas, a menos que el usuario pida explícitamente el total con impuestos.

| Columna | Descripción | Cuándo usar |
|---------|-------------|-------------|
| `subtotal` | Monto neto sin IVA | **Siempre por defecto** |
| `total` | Monto con IVA incluido | Solo si el usuario lo pide explícitamente ("con IVA", "total con impuestos", "monto bruto") |

Si una respuesta usa `subtotal`, no es necesario aclararlo cada vez. Solo mencionar si el usuario pregunta por qué o si cambia al modo con IVA.

Esta regla aplica también a `presupuesto_venta`, que ya usa `monto_neto` (sin IVA) por diseño.

---

## Tasas de IVA por país

| País | Empresa | Tasa IVA |
|------|---------|----------|
| Chile | `IRONSIDE SPA` | 19% |
| México | `IRONSIDE FITNESS SA DE CV` | 16% |
| Argentina | `IRONSIDE FITNESS SA` | 21% |

Si el usuario pide el total con IVA y la consulta ya devuelve `subtotal`,
calcular el total como `subtotal * (1 + tasa)` según la compañía.
Si la consulta mezcla países, aplicar cada tasa individualmente por fila.
