# Bazaar — Hackathon MVP Specification

A CLI + web app that lets you package your Codex CLI global config, publish it, and let anyone install it with one command.

All bundles are public. No private access, no versioning, no fork tracking.

---

## 1. Architecture

```
┌─────────────────────────────┐
│    Next.js App (Vercel)     │
│  - Google OAuth (NextAuth)  │
│  - Bundle listing UI        │
│  - Bundle detail UI         │
│  - REST API (/api/*)        │
│  - Postgres DB (Prisma)     │
│  - Vercel Blob (archives)   │
└──────────┬──────────────────┘
           │ HTTPS
┌──────────▼──────────────────┐
│    Python CLI (pip install) │
│  - login (browser OAuth)    │
│  - upload (pack + POST)     │
│  - setup (GET + install)    │
│  - uninstall (restore)      │
└─────────────────────────────┘
```

Next.js API routes handle everything. No separate server process. Bundle archives (tar.gz) are stored in Vercel Blob. Metadata lives in Postgres via Prisma.

---

## 2. Data Model

### Users

| Field | Type | Notes |
|---|---|---|
| `id` | string (cuid) | Primary key. |
| `google_id` | string | Unique. |
| `email` | string | |
| `name` | string | |
| `avatar_url` | string | |
| `api_token` | string | Random token for CLI auth. Generated on first login. |
| `created_at` | datetime | |

### Bundles

| Field | Type | Notes |
|---|---|---|
| `id` | string (cuid) | Primary key. |
| `code` | string | Unique. Format: `bz-` + 6 lowercase alphanumeric chars. |
| `author_id` | string | FK → Users. |
| `name` | string | |
| `description` | text | |
| `archive_url` | string | URL to the stored tar.gz in Vercel Blob. |
| `includes` | JSON | `{ "config": true, "agents_md": false, "rules": 2, "skills": 1, "themes": 0 }` |
| `install_count` | integer | Default 0. Incremented on each download. |
| `created_at` | datetime | |
| `updated_at` | datetime | |

---

## 3. API Endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/auth/google` | No | Initiates Google OAuth via NextAuth. |
| `GET` | `/api/auth/callback` | No | OAuth callback. Creates user + issues API token. |
| `GET` | `/api/bundles` | No | List all bundles. Returns metadata array (no archive URLs). |
| `GET` | `/api/bundles/[code]` | No | Get single bundle metadata by code. |
| `POST` | `/api/bundles` | Yes | Create bundle. Multipart: JSON metadata + tar.gz archive. Returns assigned code. |
| `GET` | `/api/bundles/[code]/download` | Yes | Returns the archive file. Increments install count. |

Auth is a Bearer token in the `Authorization` header. The token is `api_token` from the Users table.

---

## 4. Config Bundle

### What gets packaged

| Source Path | Bundle Path |
|---|---|
| `~/.codex/config.toml` | `config.toml` |
| `~/.codex/AGENTS.md` | `AGENTS.md` |
| `~/.codex/AGENTS.override.md` | `AGENTS.override.md` |
| `~/.codex/rules/*.rules` | `rules/*.rules` |
| `~/.agents/skills/*/` | `skills/*/` |
| `~/.codex/themes/*.tmTheme` | `themes/*.tmTheme` |

All files are optional except `config.toml`.

### Never packaged

- `~/.codex/auth.json`
- `~/.codex/history.jsonl`
- `~/.codex/log/`
- `*.pem`, `*.key`, `.env`, `credentials.*`

### Sanitization

Two rules:

1. Replace all values in `mcp_servers.<id>.env` with `"<SET_YOUR_VALUE>"`. Keep key names.
2. Remove any `experimental_bearer_token` field from `model_providers`.

The CLI prints a summary of what was redacted before uploading.

### Archive format

tar.gz. Files at the archive root, no wrapper directory.

```
bundle.tar.gz
├── config.toml
├── AGENTS.md
├── rules/
│   └── default.rules
└── skills/
    └── my-skill/
        └── SKILL.md
```

---

## 5. CLI Commands

### `bazaar login`

1. Start a temporary HTTP server on `localhost:<random-port>`.
2. Open browser to `https://<bazaar-web>/login?callback=http://localhost:<port>`.
3. Web app authenticates via Google, redirects to callback with `?token=<api_token>&email=<email>&name=<name>`.
4. Write `~/.bazaar/credentials.json`:
   ```json
   { "token": "<api_token>", "email": "<email>", "name": "<name>" }
   ```
5. Print: `Logged in as <name> (<email>)`.

### `bazaar logout`

1. Delete `~/.bazaar/credentials.json`.
2. Print: `Logged out.`

### `bazaar upload --name <name> --description <desc>`

1. Check auth. Error if `credentials.json` missing.
2. Scan source paths. Error if `~/.codex/config.toml` doesn't exist.
3. Sanitize `config.toml`.
4. Print summary: files included, fields redacted.
5. Create tar.gz archive in a temp directory.
6. POST to `/api/bundles` with archive + metadata.
7. Receive bundle code from server.
8. Print: `Uploaded: bz-xxxxxx` and the web URL.

Re-running upload creates a new bundle, not an update.

### `bazaar setup <code>`

1. Check auth. Error if `credentials.json` missing.
2. GET `/api/bundles/<code>` for metadata. Error if not found.
3. GET `/api/bundles/<code>/download` to fetch archive.
4. Backup current config to `~/.bazaar/backups/<timestamp>/`:
   - `~/.codex/config.toml`
   - `~/.codex/AGENTS.md` (if exists)
   - `~/.codex/AGENTS.override.md` (if exists)
   - `~/.codex/rules/` (if exists)
   - `~/.agents/skills/` (if exists)
   - `~/.codex/themes/` (if exists)
5. Extract archive and place files at target locations (reverse of the packaging mapping).
6. Write `~/.bazaar/state.json`:
   ```json
   { "active_bundle": "bz-xxxxxx", "backup_path": "~/.bazaar/backups/<timestamp>" }
   ```
7. Print: `Installed bz-xxxxxx. Run 'bazaar uninstall' to revert.`

### `bazaar uninstall`

1. Read `~/.bazaar/state.json`. Error if no active bundle.
2. Copy all files from the backup back to their original locations.
3. Clear `state.json`.
4. Print: `Reverted to previous config.`

---

## 6. Web Pages

### Landing page (`/`)

- Hero: project name, one-liner, "Browse Configs" CTA.
- Grid of bundle cards sorted newest first.
- Each card: name, author name + avatar, description (truncated), bundle code with copy button, install count.
- "Sign in with Google" in the header.

### Bundle detail page (`/bundles/[code]`)

- Bundle name, full description.
- Author name + avatar.
- Bundle code with copy button.
- "How to install" section with CLI command: `bazaar setup <code>`.
- "What's included" section: file categories present (config, N rules, N skills, agent instructions, N themes).
- Install count.
- Created date.

### Login redirect page (`/login`)

- Handles OAuth flow.
- If `?callback` param present: redirects to CLI callback URL with token after auth.
- If no callback: redirects to landing page.

No upload UI. Uploads go through the CLI only.

### Design

Dark theme. Monospace accents for bundle codes and CLI commands. Header with logo + sign in. Card-based grid. No sidebar.

---

## 7. Local Directory Structure

```
~/.bazaar/
├── credentials.json              # { token, email, name }
├── state.json                    # { active_bundle, backup_path }
└── backups/
    └── 2026-02-28T12-00-00/
        ├── config.toml
        ├── AGENTS.md
        ├── rules/
        ├── skills/
        └── themes/
```

---

## 8. Tech Stack

| Component | Technology |
|-----------|-----------|
| Web framework | Next.js 16 (App Router) |
| Auth | NextAuth.js + Google OAuth |
| Database | Postgres (Prisma) |
| File storage | Vercel Blob |
| CLI framework | Python + Typer |
| CLI HTTP | httpx |
| CLI packaging | tarfile (stdlib) |
| CLI TOML parsing | tomllib (3.11+) |
| Deployment | Vercel |
