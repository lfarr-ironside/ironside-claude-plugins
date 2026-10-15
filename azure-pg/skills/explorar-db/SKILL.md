---
description: |
  Use this skill when the user wants to query, explore, or inspect their Azure
  PostgreSQL database. Trigger phrases: "consulta la base de datos", "muéstrame
  las tablas", "ejecuta este query", "qué hay en la tabla", "busca registros",
  "dame el schema", "cuántos registros", "lista las bases de datos",
  "list tables", "run a query", "describe table", "explore the database",
  or any natural-language question about data in the PostgreSQL server.
---

# Explorar-db — consultas a Azure PostgreSQL

Use the Azure MCP Server tools to answer the user's question about their
PostgreSQL database. Before answering any question about business data, read
`references/contexto-negocio.md` — it contains verified column names, exact
company and team values, and critical rules about the data model.

## Connection context (fixed for Ironside)

- **resource-group:** `invoices-db-ironside`
- **server:** `invoices-ironside-postgresql`
- **database:** `ironside_reports` (default — confirm if user mentions another)
- **user:** ask for the user's Entra email if not provided (e.g. `name@ironside.cl`)
- **subscription:** read from `AZURE_SUBSCRIPTION_ID` env var

## Tool reference

| Goal | Tool | Required params |
|------|------|----------------|
| List databases | `postgres_list` | subscription, resource-group, server, user |
| List tables | `postgres_list` | + database |
| Get table schema | `postgres_table_schema_get` | + database, table |
| Run SQL query | `postgres_database_query` | + database, query |

## Always apply business rules from references/contexto-negocio.md

Apply these rules automatically on every query — never wait to be asked:

1. **Montos — siempre ventas netas sin IVA.** Use `subtotal` by default for all
   sales figures. Only switch to `total` if the user explicitly asks for amounts
   with taxes ("con IVA", "total con impuestos", "monto bruto"). This rule also
   applies when comparing against `presupuesto_venta.monto_neto`, which is
   already net by design.

2. **Facturas aperturadas** — always use `COUNT(DISTINCT movimiento)` to count
   invoices, never `COUNT(*)`.

3. **Notas de crédito** — `subtotal` can be negative for `out_refund` rows.
   For net sales, include all rows and let negatives offset naturally.
   Mention credit notes only if they materially affect the result.

4. **Company filter** — when the user mentions a country, map it to the exact
   `compania` value from the reference file.

5. **Team filter** — when the user mentions "ecommerce" or "comercial" for a
   country, map to the exact `equipo_ventas` value.

6. **presupuesto_venta** — never use this table as sales history. It is a
   financial projection only. Join with `invoices` on
   `presupuesto_venta.area = invoices.equipo_ventas` for real vs budget.

## Query safety rules

- Only run `SELECT` unless the user explicitly requests a write.
- For destructive queries (DROP, DELETE, TRUNCATE, UPDATE), confirm:
  "Esto modificará datos reales. ¿Confirmas?" and wait for explicit yes.
- Never expose connection strings, passwords, or tokens in responses.

## Response format

1. State what you will do in one sentence before calling any tool.
2. Call the tool.
3. Present results as a readable table or summary — not raw JSON.
4. If results are large (>50 rows), summarize and offer to filter.
5. Suggest a natural follow-up question.

## Troubleshooting

If auth error: remind user to run `az login` and check `AZURE_SUBSCRIPTION_ID`.
If permission error: point to "Provisioning del DBA" in README.
If unsure about schema: run `postgres_table_schema_get` before querying.
