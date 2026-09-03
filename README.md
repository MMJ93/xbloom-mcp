# MMJ93 xBloom MCP

Personal xBloom MCP maintained by [MMJ93](https://github.com/MMJ93), based on
[`denull0/xbloom-agent`](https://github.com/denull0/xbloom-agent).

Version 2.1.1 adds explicit Omni/Other dripper selection, complete
account-shared recipe discovery, and portable Supabase deployment support.

Let Codex create custom coffee and tea recipes for your XBloom Studio machine. Just tell Codex about your coffee or tea — or share a photo of the bag — and it designs a recipe that syncs straight to your xBloom app.

Works with Codex and other MCP clients that support Streamable HTTP and OAuth.

---

## Get Started

### Step 1: Connect to Codex

This deployment is live at:

```
https://mbwgxlzkscuxtayrlcjh.supabase.co/functions/v1/xbloom-mcp
```

Add it with `codex mcp add` using the command in the developer guide below, then approve the OAuth connection when prompted.

### Step 2: Sign in with your XBloom account

The first time you use it, Codex will ask for your XBloom email and password. This links your XBloom account so recipes go directly to **your** app. Your password is used once and **never saved**.

### Step 3: Start chatting

Ask Claude to make you a recipe. Here are some ideas:

**Coffee:**

> *"Here's a photo of my coffee bag. Make me a recipe for it."*

> *"I have a medium roast Colombian, 18g dose. I like it bright and clean."*

> *"That last brew was a little bitter — can you adjust?"*

**Tea:**

> *"Create a tea recipe for my hojicha, 5g, two steeps."*

> *"Make a green tea recipe — 3g sencha, 70°C, 60 second steep."*

> *"I want an oolong recipe with three steeps, getting hotter each time."*

**Manage:**

> *"Show me all my recipes."*

> *"Show me every recipe shared with me."*

> *"Delete the old test recipe."*

Recipes sync instantly to the **xBloom iOS app** and are ready to brew.

### What can it do?

- **Coffee recipes** — Pour-over recipes for the Omni dripper using brewing science (Kasuya 4:6, Hoffmann, Rao, etc.)
- **Tea recipes** — Steep recipes for the Omni Tea Brewer with proper temperatures and steep times
- **Shared recipes** — List every recipe shared to your account, with share links
- **Photo-to-recipe** — Share a photo of your coffee or tea bag, Codex reads the label and creates a recipe
- **Link-to-recipe** — Paste a product link, Claude pulls the details and designs a recipe
- **Taste adjustments** — Tell Claude it was too bitter/sour/weak and it tweaks the recipe
- **Manage recipes** — List, edit, and delete recipes right from the chat
- **Import recipes** — Grab any shared XBloom recipe by URL

### Privacy

- Your password is **never stored** — it's used once to log in, then thrown away
- Each user has their own account — nobody else can see or touch your recipes
- Session tokens are encrypted at rest

---

## Developer Guide

Everything below is for developers who want to self-host or modify the server.

### Tech Stack

- **Runtime**: Deno 2.x on Supabase Edge Functions
- **Protocol**: MCP 2.0 (Streamable HTTP + SSE)
- **Auth**: OAuth 2.0 + per-user XBloom login
- **Encryption**: AES-256-CBC (sessions) + RSA (API payloads, XBloom's key)

### MCP Tools

| Tool | Description |
|------|-------------|
| `xbloom_login` | Authenticate with your XBloom account |
| `xbloom_list_recipes` | List all your recipes with IDs |
| `xbloom_list_shared_recipes` | List all recipes shared to your account, including share links |
| `xbloom_create_recipe` | Create a coffee recipe (Omni dripper) |
| `xbloom_create_tea_recipe` | Create a tea recipe (Omni Tea Brewer) |
| `xbloom_edit_recipe` | Update an existing recipe by ID |
| `xbloom_delete_recipe` | Permanently remove a recipe |
| `xbloom_fetch_recipe` | Import a recipe from a share URL |

Coffee create/edit operations accept `dripper_type: "omni" | "other"`.
Use `"other"` when a dose above 18g must remain editable in the xBloom app.

### Self-Hosting

#### Prerequisites

- [Supabase CLI](https://supabase.com/docs/guides/cli/getting-started)
- [Deno 2.x](https://deno.com)

#### 1. Clone

```bash
git clone https://github.com/MMJ93/xbloom-mcp.git
cd xbloom-mcp/xbloom-mcp-remote
```

#### 2. Apply the database migration FIRST

The function writes `refresh_token` / `expires_at` columns, so the schema must exist
before you deploy — otherwise every write fails and login can't persist a session.

```bash
supabase db push   # applies supabase/migrations/ (creates/updates user_sessions)
```

The migration is idempotent and also relaxes/upgrades a table created by an older
version. If you can't run `supabase db push`, run the SQL in
`supabase/migrations/20260707000000_user_sessions.sql` against your database.

#### 3. Deploy the function

```bash
supabase functions deploy xbloom-mcp --no-verify-jwt
```

No environment variables needed — the server uses `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` which are automatically available in edge functions.

#### 4. Connect your MCP client

Add your server URL to Codex or another MCP client:

```
https://mbwgxlzkscuxtayrlcjh.supabase.co/functions/v1/xbloom-mcp
```

For Codex:

```bash
codex mcp add xbloom --url https://mbwgxlzkscuxtayrlcjh.supabase.co/functions/v1/xbloom-mcp \
  --oauth-resource https://mbwgxlzkscuxtayrlcjh.supabase.co/functions/v1/xbloom-mcp
codex mcp login xbloom
```

### Recipe Parameters

**Coffee** (Omni dripper):

| Parameter | Range | Notes |
|-----------|-------|-------|
| `dose_g` | 1–31 | Coffee dose in grams |
| `grind_size` | 40–120 | Lower = finer |
| `grind_rpm` | 60–120 | Grinder speed |
| `temperature_c` | 40–95 | Water temperature |
| `flow_rate` | 3.0–3.5 | mL/s |
| `pattern` | centered, circular, spiral | Pour pattern |
| `pause_seconds` | 0–255 | Pause between pours |

**Tea** (Omni Tea Brewer):

| Parameter | Range | Notes |
|-----------|-------|-------|
| `dose_g` | 1–10 | Tea dose in grams |
| `volume_ml` | 1–90 | Water per steep (machine adds ~30ml for siphon) |
| `temperature_c` | 65–100 | Green: 70-80, White: 75-85, Oolong: 85-95, Black: 90-100 |
| `steep_seconds` | 0–360 | Up to 6 minutes per steep |
| `steeps` | 1–3 | Number of steeps |

### Project Structure

```
xbloom-agent/
├── xbloom-mcp-remote/
│   └── supabase/
│       ├── config.toml                     # Supabase project config
│       └── functions/
│           └── xbloom-mcp/index.ts         # MCP server (OAuth + tools + SSE)
└── xbloom-recipes/
    └── claude-project/
        ├── custom-instructions.md          # Claude project instructions
        └── xbloom-brewing-reference.md     # Coffee brewing science reference
```

### Security

- Passwords are **never stored** — used once for XBloom API login, then discarded
- Session tokens are **AES-256 encrypted** at rest using HMAC-SHA256 derived keys
- Database table has **Row Level Security** — only the server can access it
- Error messages are sanitized — no internal API details leaked

## License

MIT
