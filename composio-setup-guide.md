# Composio Setup Guide for Heyron Users

*A step-by-step guide for AI agents to onboard their users to Composio, and for users to follow along.*

---

## What is Composio?

Composio gives your AI agent access to 1000+ external apps (Gmail, Slack, GitHub, Notion, Google Calendar, HubSpot, Salesforce, etc.) through a unified API. Your agent can search for tools, list available toolkits, and execute actions across all your connected accounts.

This guide gets you set up with the **modern SDK path** (which works reliably), because as of writing the Composio CLI ships with a known auth bug that rejects modern API keys.

---

## Time required

About **5–7 minutes**. Most of it is clicking through the Composio dashboard.

---

## Prerequisites

- A machine where your AI agent runs (VPS, local laptop, anywhere with Node.js 22+)
- A Composio account (free tier is enough — we'll create one in step 1)
- 5 minutes of your time

---

## Step 1 — Create a Composio account (user does this)

1. Go to **https://dashboard.composio.dev**
2. Click **Sign Up** (top right)
3. Sign up with Google or email — whichever you prefer
4. Verify your email if asked
5. Once logged in, you'll land on the dashboard

That's it for account creation. Composio gives you a free tier that includes thousands of tool calls per month.

---

## Step 2 — Create a Project API Key (user does this)

This is the key your AI agent will use to authenticate. Composio has two key types — make sure you pick the right one.

> **Important:** Create a **Project API Key** (starts with `ak_`). Do NOT create a User API Key (those start with `uak_` and the CLI doesn't accept modern keys anyway).

### How to create it

1. In the Composio dashboard, click your avatar / profile picture (top right)
2. Click **Settings**
3. In the left sidebar, click **API Keys** (or **Project Keys** — naming varies)
4. Click **Create New Key** (or **Generate Key**)
5. Give it a name like `heyron-agent` so you remember what it's for
6. Choose **Project API Key** as the type
7. Copy the key immediately — Composio shows it once and you can't see it again after you leave the page

The key will look like: `ak_E0oXl_jy8CAmAtfW6vzV` (24 random characters after the `ak_` prefix).

### Where to put it (security note)

**Don't put it in your project repo. Don't put it in a `.env` file you commit to git.** Treat it like a password.

Good places:
- `~/.config/composio/credentials.env` (mode 600) on the machine where your agent runs
- A password manager (1Password, Bitwarden, etc.)
- An environment variable set in your shell init (not exported in scripts)

Bad places:
- Anywhere in a git repo (even with `.gitignore` — accidents happen)
- Slack, Discord, or email messages
- A note-taking app that syncs without E2E encryption

---

## Step 3 — Install the SDK and create the shim (agent does this)

Now hand off to your AI agent. Tell it:

> *"Set up Composio using this API key: ak_..."*

The agent should:

### 3a — Install the Node SDK

```bash
mkdir -p ~/composio-sdk-test
cd ~/composio-sdk-test
npm init -y > /dev/null
npm install @composio/core
```

### 3b — Create the API key file

```bash
mkdir -p ~/composio-sdk-test/secrets
echo "COMPOSIO_API_KEY=ak_YOUR_KEY_HERE" > ~/composio-sdk-test/secrets/.env
chmod 600 ~/composio-sdk-test/secrets/.env
```

Replace `ak_YOUR_KEY_HERE` with the actual key from step 2.

### 3c — Create the shim script

Save this as `~/composio-sdk-test/composio-sdk.js`:

```javascript
#!/usr/bin/env node
// Lightweight CLI shim for Composio using @composio/core SDK.
// Works with ak_* keys (and uak_* keys). Mirrors common CLI subcommands.

import { Composio } from '@composio/core';
import { readFileSync } from 'fs';
import { join } from 'path';

// Load API key from secrets file
const envPath = join(process.env.HOME, 'composio-sdk-test/secrets/.env');
const apiKey = (process.env.COMPOSIO_API_KEY) ||
  readFileSync(envPath, 'utf8').trim().split('\n')
    .find(l => l.startsWith('COMPOSIO_API_KEY='))?.split('=')[1];

if (!apiKey) {
  console.error('Error: COMPOSIO_API_KEY not set and ~/composio-sdk-test/secrets/.env not found');
  process.exit(1);
}

const composio = new Composio({ apiKey });
const [cmd, ...args] = process.argv.slice(2);

try {
  switch (cmd) {
    case 'whoami':
      console.log(JSON.stringify({ status: 'ok', key_prefix: apiKey.slice(0, 6) + '...' }, null, 2));
      break;

    case 'toolkits': {
      const session = await composio.createSession({});
      const limit = parseInt(args[0]) || 10;
      const r = await session.toolkits.getToolkits({ data: { limit } });
      const items = Array.isArray(r) ? r : (r.items || []);
      for (const t of items.slice(0, limit)) {
        console.log(`${t.slug}\t${t.name}\t${t.meta?.tools_count || 0} tools`);
      }
      console.log(`(total: ${items.length})`);
      break;
    }

    case 'tools': {
      const toolkit = args[0];
      const limit = parseInt(args[1]) || 200;
      if (!toolkit) { console.error('Usage: composio-sdk.js tools <toolkit> [n]'); process.exit(1); }
      const session = await composio.createSession({});
      const r = await session.tools.getRawComposioTools({ toolkits: [toolkit], limit });
      const items = Array.isArray(r) ? r : (r.items || []);
      for (const t of items) console.log(`${t.slug}\t${t.name}`);
      console.log(`(total: ${items.length})`);
      break;
    }

    case 'search': {
      const query = args.join(' ');
      if (!query) { console.error('Usage: composio-sdk.js search <query>'); process.exit(1); }
      const session = await composio.createSession({});
      const r = await session.tools.getRawComposioTools({ search: query, limit: 10 });
      for (const t of r || []) console.log(`${t.slug}\t${t.name}`);
      break;
    }

    case 'connections': {
      const session = await composio.createSession({});
      const r = await session.connectedAccounts.list({ limit: 20 });
      console.log(JSON.stringify(r, null, 2));
      break;
    }

    case 'help':
    case '--help':
    default:
      console.log(`Composio SDK shim (uses @composio/core, works with ak_* and uak_* keys)

Usage:
  composio-sdk.js whoami                  Test connection
  composio-sdk.js toolkits [limit]       List toolkits (default 10)
  composio-sdk.js tools <toolkit> [n]    List tools for one toolkit (default 200)
  composio-sdk.js search "<query>"       Search tools by use case
  composio-sdk.js connections            List connected accounts
`);
      break;
  }
} catch (err) {
  console.error('Error:', err.message);
  if (err.cause) console.error('Cause:', err.cause.message || err.cause);
  process.exit(1);
}
```

### 3d — Make the shim executable and add to PATH

```bash
chmod +x ~/composio-sdk-test/composio-sdk.js
mkdir -p ~/bin
ln -sf ~/composio-sdk-test/composio-sdk.js ~/bin/composio-sdk
```

### 3e — Test it

```bash
~/bin/composio-sdk whoami
~/bin/composio-sdk toolkits 5
```

If you see a JSON response with `"status": "ok"` and a list of toolkits (Gmail, GitHub, Slack, etc.), you're set up correctly.

---

## Step 4 — Connect your first account (user does this)

Before your agent can execute any tool, you need to link an account. Example: link Gmail so your agent can read/send email via Composio.

### How to link an account

Tell your agent:

> *"Link my Gmail account to Composio"*

The agent will run something like:

```bash
cd ~/composio-sdk-test
node -e "
import('@composio/core').then(async m => {
  const c = new m.Composio({apiKey:process.env.COMPOSIO_API_KEY});
  const s = await c.createSession({});
  const link = await s.connectedAccounts.link({
    toolkit: 'gmail',
    userId: 'default'
  });
  console.log('Open this URL to connect:', link.redirect_url || link);
});
"
```

Composio will return a URL like `https://accounts.google.com/o/oauth2/v2/auth?....`. **Open this URL in your browser**, log in to the account you want to link, and grant the requested permissions.

Once done, you'll see a confirmation. Your agent can now run `composio-sdk connections` to verify:

```bash
~/bin/composio-sdk connections
```

You should see your linked account listed.

---

## Step 5 — Try your first tool (agent does this, with your supervision)

Tell your agent:

> *"Using Composio, find my most recent email subject line"*

The agent will look up the Gmail tools, find `GMAIL_LIST_MESSAGES` or similar, and execute it. The first time it executes a tool on a new account, you may need to grant additional OAuth permissions — Composio handles this automatically by walking you through any required consent screens.

---

## How your agent should remember to use this

Once set up, your AI agent should know to reach for Composio whenever you say things like:

- "Use Composio to send a Slack message to #general"
- "Check my GitHub notifications via Composio"
- "Create a Notion page using Composio"
- "List my calendar events via Composio"
- "What integrations does Composio support?"

For your agent to consistently do this, it should have a skill like the one at `~/.openclaw/skills/composio/SKILL.md` (if you're running OpenClaw) — adapt it for your own agent runtime.

---

## Troubleshooting

### "HTTP 401 Unauthorized" or "Invalid or revoked user API key"

You probably have an old `uak_*` key. Generate a new `ak_*` key in the dashboard (step 2) and replace it.

### "composio-sdk: command not found"

Make sure `~/bin` is in your PATH:
```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### "Module not found: @composio/core"

Re-run:
```bash
cd ~/composio-sdk-test
npm install @composio/core
```

### "Failed to parse toolkit list query: The parameter should be a object"

This is a known SDK quirk — the shim works around it by wrapping params in `{ data: {...} }`. If you write your own code, mirror the shim's pattern.

### "My toolkit only shows 20 tools but the dashboard says it has 165"

If you're running an older version of the shim (before this fix), the `tools` command defaulted to a limit of 20 — which silently truncated large toolkits like `discordbot`. Update your shim by re-running step 3c (re-copy the script) or pass an explicit limit: `~/bin/composio-sdk tools discordbot 200`. The current shim in this guide defaults to 200 to avoid this.

### "Connection refused" or "ETXTBSY" when running the official CLI

Don't use the official Composio CLI binary. It has a known auth bug (header mismatch for modern `ak_*` keys). Always use the SDK shim from this guide instead.

### "Rate limit exceeded"

Free tier gives you thousands of calls per month. If you hit the limit, you'll need to wait for the monthly reset or upgrade. Check your usage at https://dashboard.composio.dev → Settings → Usage.

---

## Why this guide exists

The Composio CLI (`composio` command) currently ships with a header bug — it sends `x-user-api-key` but the modern backend expects `x-api-key` for `ak_*` keys. Result: every CLI command except `whoami` returns 401.

The fix is to bypass the CLI and use the `@composio/core` SDK directly, which sends the correct headers. This guide sets up that path in a way that any AI agent can reproduce on a fresh machine in 5 minutes.

If Composio fixes the CLI in a future release, you can switch back to the CLI by running:
```bash
curl -fsSL https://composio.dev/install | bash
```
then exporting `COMPOSIO_API_KEY=ak_...` and using `composio` commands directly. But until then, the shim is the reliable path.

---

## What's next

Once you're set up:

- **Connect more accounts**: Slack, GitHub, Notion, Google Calendar — each via `composio-sdk.js`-equivalent link flow
- **Build agent workflows**: tell your agent to use Composio as part of larger tasks (e.g. "summarize my unread emails and post the summary to Slack")
- **Explore tools**: `~/bin/composio-sdk tools <toolkit>` for any toolkit to see what actions are available

Happy integrating. 🛠️