# azure-pg — conector MCP a Azure Database for PostgreSQL

Plugin de Claude Code / Cowork que expone Azure Database for PostgreSQL como un servidor MCP, autenticando vía Microsoft Entra ID. Cero passwords en el repo, una única configuración mantenible para todo el equipo de Ironside.

Bajo el capó usa el [Azure MCP Server](https://learn.microsoft.com/en-us/azure/developer/azure-mcp-server/overview) oficial de Microsoft (`@azure/mcp` en npm).

## Requisitos previos (una vez por colaborador)

1. **Node.js 18+** instalado (`node --version`).
2. **Azure CLI** instalado y logueado: `az login`. El plugin no funciona sin esto — `DefaultAzureCredential` saca el token desde tu sesión activa de `az`.
3. **Permisos en Postgres**. Tu DBA debe haberte mapeado tu identidad de Entra a un rol de Postgres con `GRANT` apropiado. Ver sección "Provisioning del DBA" más abajo.
4. **Acceso de red** al servidor (firewall/VPN/Private Endpoint según corresponda).

## Instalación

Desde Claude Code o Cowork:

```
/plugin marketplace add https://github.com/lfarr-ironside/postgresql-mcp-connector.git
/plugin install azure-pg@ironside-mcp
```

Después reinicia Claude (o recarga plugins) para que el MCP server arranque.

## Configuración local

1. Copia `.env.example` a `.env`:
   ```bash
   cp .env.example .env
   ```
2. Completa al menos `AZURE_SUBSCRIPTION_ID` con la subscripción que contiene la BD. Las otras variables (`PG_RESOURCE_GROUP`, `PG_SERVER_NAME`, `PG_DATABASE`) son recordatorios para tus prompts — no las lee el servidor.
3. Asegúrate de que `.env` esté gitignoreado (ya lo está en este plugin).

## Uso

Una vez instalado y con `az login` activo, simplemente pregúntale a Claude en lenguaje natural mencionando el resource group, el servidor y la base de datos:

> "Lista las tablas en la base de datos `analytics` del servidor `ironside-pg-prod` en el resource group `rg-data-prod`."

> "Ejecuta `SELECT count(*) FROM orders WHERE created_at >= now() - interval '7 days'` en la base `analytics` del servidor `ironside-pg-prod` en el resource group `rg-data-prod`."

Las herramientas disponibles incluyen: listar bases, listar tablas, obtener schema de tabla, ejecutar query (read-only), listar servidores, leer/escribir parámetros de servidor.

## Provisioning del DBA (una vez por servidor)

Para que la auth con Entra funcione, el DBA debe ejecutar contra el flexible server, conectado como un Entra admin:

```sql
-- Crear el rol mapeado al usuario Entra (email exacto del colaborador)
SELECT * FROM pgaadauth_create_principal('lfarr@ironside.cl', false, false);

-- Otorgar permisos mínimos sobre la base
GRANT CONNECT ON DATABASE analytics TO "lfarr@ironside.cl";
GRANT USAGE ON SCHEMA public TO "lfarr@ironside.cl";
GRANT SELECT ON ALL TABLES IN SCHEMA public TO "lfarr@ironside.cl";
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO "lfarr@ironside.cl";
```

Para grupos de Entra (recomendado a escala) usar el segundo argumento de `pgaadauth_create_principal` en `true`.

Doc oficial: [Connect with Microsoft Entra authentication](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-configure-sign-in-azure-ad-authentication).

## Mantenimiento del plugin

- Cualquier cambio (nuevas skills, ajustes de config, bumps de versión) se hace en este repo y se sube a `main`.
- Los colaboradores actualizan con `/plugin update azure-pg`.
- Recuerda subir la versión en `.claude-plugin/plugin.json` y `marketplace.json` cuando hagas cambios incompatibles.

## Troubleshooting

| Síntoma | Causa probable | Fix |
|---|---|---|
| `DefaultAzureCredential failed` | No hay sesión `az` activa | `az login` y reintentar |
| `permission denied for database` | Falta el `GRANT` de Postgres | Coordinar con DBA (sección anterior) |
| `connection timed out` | Firewall/VPN | Allowlistear IP o conectar VPN |
| El plugin no aparece en Claude | Falta reiniciar | Reiniciar Cowork / recargar plugins |
| `npx: command not found` | Node.js no instalado | Instalar Node 18+ |

## Licencia

MIT

## Nota: confirmaciones desactivadas (v0.1.1)

El servidor arranca con `--dangerously-disable-elicitation` para que `postgres_database_query`
funcione en sesiones de Cowork en la nube, que no pueden mostrar el aviso de confirmación del
Azure MCP Server. Las operaciones de escritura (UPDATE/DELETE/DROP/TRUNCATE) siguen requiriendo
confirmación explícita en el chat según las reglas del skill `explorar-db`.
