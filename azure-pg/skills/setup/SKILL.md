---
description: |
  Guided onboarding for the azure-pg plugin. Use this skill when a user says
  "set up the plugin", "configure azure-pg", "install the database connector",
  "ayúdame a configurar el plugin", "cómo conecto a la base de datos Azure",
  "quiero usar la base de datos", "no puedo conectarme", "cómo empiezo",
  or "setup" after installing azure-pg. Walks through all prerequisites step
  by step so no colleague needs to read the README.
---

# Setup — azure-pg plugin

Walk the user through setup in a friendly, one-step-at-a-time conversation.
Never use technical jargon without explaining it first. Never show all steps
at once. Celebrate each success briefly before moving on.

Assume the user may have zero technical background. Explain every concept the
first time it appears, in one plain sentence.

---

## Step 1 — Greet and set expectations

Open with something like:

> "¡Hola! Voy a ayudarte a conectar Claude con la base de datos de Ironside.
> Son 3 pasos cortos y yo te guío en cada uno. Cuando terminemos, podrás
> hacerme preguntas sobre las facturas y ventas directamente en este chat.
> ¿Empezamos?"

Wait for confirmation before continuing.

---

## Step 2 — Check Node.js (the engine that makes the connection work)

Explain first:
> "Primero necesito verificar que tu computador tiene instalado Node.js,
> que es el programa que permite que Claude se comunique con Azure."

Run:
```
node --version
```

- **Node 18 or higher** → "✓ Todo en orden. Sigamos."
- **Older than 18** → "Tu versión es un poco antigua. Necesitas actualizarla.
  Descarga la versión recomendada desde https://nodejs.org — es el botón verde
  grande que dice 'LTS'. Avísame cuando lo hayas instalado."
- **Command not found** → "Node.js no está instalado en tu computador todavía.
  Ve a https://nodejs.org y descarga el instalador (botón verde grande que dice
  'LTS'). Ejecútalo como cualquier programa de Windows y avísame cuando
  termines."

Pause and wait. Re-run the check once they confirm.

---

## Step 3 — Azure CLI and login (the identity verification)

Explain first:
> "Ahora necesito que tu computador le demuestre a Azure que eres tú.
> Esto se hace con un programa llamado Azure CLI que probablemente ya tienes.
> Lo voy a verificar."

Run:
```
az --version
```

**If not found:**
> "El programa Azure CLI no está instalado. Es el que le dice a Azure quién
> eres. Para instalarlo, sigue estos pasos:
> 1. Ve a https://aka.ms/installazurecliwindows
> 2. Descarga el instalador (.msi) y ejecútalo
> 3. Reinicia tu terminal y dime cuando esté listo"

Pause and wait. Re-run the check.

**If found, check login status:**
```
az account show --output json
```

- **Logged in** → Extract the email from the `user.name` field.
  > "✓ Veo que ya estás conectado como [email]. ¿Es tu cuenta de Ironside?
  > Si no, te ayudo a cambiar de cuenta."
  If confirmed → move to Step 4.
  If wrong account → guide them to run `az login` (see below).

- **Not logged in** → explain and guide:
  > "Necesitas iniciar sesión con tu cuenta de Ironside (@ironside.cl).
  > Voy a pedirte que abras una ventana negra llamada 'Terminal' o
  > 'PowerShell' (búscala en el menú Inicio) y copies este comando:
  >
  > ```
  > az login
  > ```
  >
  > Se va a abrir tu navegador con una pantalla de Microsoft. Ingresa con
  > tu correo de Ironside y tu contraseña habitual. Cuando el navegador
  > diga 'You have logged in', vuelve aquí y dime."

  Once they confirm, re-run `az account show` to verify and extract their email.
  Save this email — it will be used as `--user` in all database queries.

---

## Step 4 — Configure the subscription (automatic for Ironside)

Explain first:
> "Ahora necesito decirle a Azure a qué cuenta de la empresa conectarse.
> Para Ironside esto es siempre lo mismo, así que lo configuro yo
> automáticamente."

The subscription ID is NOT hardcoded here: this repo is public, and an
internal subscription ID doesn't belong in it. Read it from the user's own
Azure session instead — they already ran `az login`:

```
az account show --query id -o tsv
```

If that returns more than one subscription or the wrong one, ask the user
which to use.

Check if already set:
```
echo $env:AZURE_SUBSCRIPTION_ID
```
(On Mac/Linux: `echo $AZURE_SUBSCRIPTION_ID`)

**If already set to the correct value** → "✓ Ya estaba configurado. Sigamos."

**If empty or wrong** → explain and give the exact command to copy-paste:

> "Voy a configurar la conexión a la suscripción de Ironside. Abre tu
> Terminal o PowerShell y copia este comando exactamente:
>
> ```powershell
> [System.Environment]::SetEnvironmentVariable("AZURE_SUBSCRIPTION_ID", "<el id que devolvió az account show>", "User")
> ```
>
> Este comando guarda una 'llave' en tu computador que le indica a qué
> cuenta de Azure conectarse. Solo necesitas hacerlo una vez.
> Una vez que lo ejecutes, **cierra y vuelve a abrir Cowork** para que
> tome efecto. Avísame cuando estés de vuelta."

After they reopen Cowork, verify:
```
echo $env:AZURE_SUBSCRIPTION_ID
```
If it shows the UUID → "✓ Perfecto."
If still empty → offer the alternative via Windows GUI:
> "Si el comando no funcionó, podemos hacerlo visualmente:
> 1. Presiona la tecla Windows + R, escribe `sysdm.cpl` y Enter
> 2. Ve a la pestaña 'Opciones avanzadas'
> 3. Haz clic en 'Variables de entorno'
> 4. En la sección 'Variables de usuario', haz clic en 'Nueva'
> 5. En 'Nombre' escribe: `AZURE_SUBSCRIPTION_ID`
> 6. En 'Valor' pega el id que devolvió `az account show --query id -o tsv`
> 7. Acepta todo y reinicia Cowork"

---

## Step 5 — Final connection test

Explain first:
> "Último paso: voy a hacer una prueba real para confirmar que todo funciona."

Use the email saved in Step 3 and run via MCP tool `postgres_list`:
- subscription: el valor de `AZURE_SUBSCRIPTION_ID` (ver arriba cómo obtenerlo)
- resource-group: `invoices-db-ironside`
- server: `invoices-ironside-postgresql`
- user: [email from Step 3]

**If it returns databases** → celebrate and close:
> "🎉 ¡Todo funciona perfecto! Ya estás conectado a la base de datos de
> Ironside. Ahora puedes hacerme preguntas como:
> - 'Muéstrame las ventas de Chile del mes pasado'
> - '¿Cuántas facturas emitió Argentina en 2024?'
> - 'Compara ventas reales vs presupuesto de este año'
>
> ¿Hay algo que quieras consultar ahora?"

**If auth error** → revisit Step 3 (az login).
**If permission error on database** →
> "Tu usuario está autenticado, pero el administrador de la base de datos
> aún no ha habilitado tu acceso. Contacta a [lfarr@ironside.cl] para que
> te agreguen los permisos necesarios."

---

## General tone rules

- One step at a time. Never mention future steps unless asked.
- Replace all technical terms with plain language on first use:
  - "variable de entorno" → "una llave guardada en tu computador"
  - "terminal / PowerShell" → "una ventana negra donde se escriben comandos"
  - "autenticación" → "decirle a Azure quién eres"
  - "suscripción" → "la cuenta de Azure de la empresa"
- When a step succeeds, say so with a checkmark and move on immediately.
- When a step fails, give one specific, actionable fix — not a list of possibilities.
- Never ask the user to edit files or look up IDs. Do it for them.
