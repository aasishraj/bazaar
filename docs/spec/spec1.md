# Bazaar — Specification v1

A marketplace platform for sharing, discovering, and installing AI IDE configurations.

---

## 1. Overview

Bazaar is a platform that lets users package, publish, discover, and install global configurations for AI-powered IDE tooling. The initial scope is limited to **OpenAI Codex CLI** configurations, with the architecture designed to support additional IDEs in the future.

The platform consists of two components:

1. **Bazaar Web** — A Next.js web application serving as the public marketplace. Users authenticate via Google OAuth, browse and discover published configurations, manage access, and view bundle details.
2. **Bazaar CLI** — A Python command-line tool that handles authentication, packaging, uploading, downloading, and installing configuration bundles on the user's machine.

The system is **git-based**: every configuration bundle is stored as a versioned git repository on the server, giving users version history, diffing, fork lineage tracking, and rollback for free.

---

## 2. Goals

- Allow users to share their entire Codex CLI global configuration as a single installable bundle.
- Provide a searchable, browsable marketplace for discovering useful configurations.
- Support both public (open to all) and private (access-controlled) bundles.
- Track lineage: when a user publishes a modified version of someone else's bundle, the system records that parentage (fork relationship).
- Ensure security by sanitizing sensitive data (API keys, tokens, secrets) before upload and warning users about potentially dangerous content on download.
- Make installation and rollback trivial via the CLI.

## 3. Non-Goals (v1)

- Paid / monetized bundles. All bundles in v1 are free. Paid access is a future consideration.
- Support for IDEs other than Codex CLI. The architecture accommodates this, but only `codex` is implemented in v1.
- Partial config merging. Installing a bundle replaces the global config entirely (with backup). Smart merging is deferred.
- Rating, reviews, or community features beyond access requests.
- A hosted git interface (no web-based diff viewer, no pull requests between bundles).

---

## 4. Terminology

| Term | Definition |
|---|---|
| **Bundle** | A packaged collection of Codex CLI global configuration files, published to the marketplace. |
| **Bundle Code** | A short, unique, auto-generated alphanumeric identifier for a bundle (e.g., `bz-k7x2m9`). Used in the CLI and displayed on the web. |
| **Author** | The user who created and published a bundle. |
| **Consumer** | A user who downloads and installs a bundle. |
| **Fork** | A new bundle derived from an existing one. The original is recorded as the **parent**. |
| **Manifest** | A metadata file (`manifest.json`) inside a bundle that describes its contents, authorship, parent lineage, and Bazaar-specific settings. |
| **Active Config** | The bundle currently installed and applied to the user's global Codex CLI configuration. |

---

## 5. Architecture Overview

### 5.1 High-Level Components

```
┌──────────────────────────────────┐
│         Bazaar Web (Next.js)     │
│  - Google OAuth                  │
│  - Marketplace UI                │
│  - REST API for CLI              │
│  - Admin / author dashboards     │
└──────────┬───────────────────────┘
           │  HTTPS (REST API)
           │
┌──────────▼───────────────────────┐
│         Bazaar Server            │
│  - Auth service (Google OAuth)   │
│  - Bundle service (CRUD, search) │
│  - Access control service        │
│  - Git storage manager           │
└──────────┬───────────────────────┘
           │
     ┌─────┴──────┐
     │             │
┌────▼───┐  ┌─────▼──────┐
│ Database│  │ Git Storage │
│ (Postgres) │ (bare repos) │
└────────┘  └────────────┘

┌──────────────────────────────────┐
│        Bazaar CLI (Python)       │
│  - Auth (OAuth browser flow)     │
│  - Package / upload              │
│  - Download / install            │
│  - Backup / restore              │
└──────────────────────────────────┘
```

### 5.2 Storage Design

**Database (PostgreSQL):** Stores user accounts, bundle metadata, access permissions, access requests, and tags. Does not store bundle file contents.

**Git Storage:** Each bundle is a bare git repository on the server's file system (or object storage). Updates to a bundle create new commits. Forks create new repositories with a recorded parent reference in the database. This gives the system:

- Full version history per bundle.
- Ability to diff any two versions.
- Lightweight branching and rollback.
- Familiar semantics for developers.

### 5.3 IDE Namespacing

All paths and operations are namespaced by IDE identifier. For v1, the only supported value is `codex`. This means:

- Local storage: `~/.bazaar/codex/`
- Config targets: `~/.codex/` (config, rules) and `~/.agents/skills/` (user-scoped skills)
- Bundle manifest field: `"ide": "codex"`

Future IDEs would introduce new namespaces (e.g., `cursor`, `windsurf`) with their own target paths.

---

## 6. Config Bundle

A bundle is a snapshot of a user's global Codex CLI configuration, sanitized and packaged for distribution.

### 6.1 Bundle Contents

A bundle repository has the following structure:

```
<bundle-repo>/
├── manifest.json
├── config.toml
├── AGENTS.md                 (optional)
├── AGENTS.override.md        (optional)
├── rules/                    (optional)
│   └── *.rules
├── skills/                   (optional)
│   └── <skill-name>/
│       ├── SKILL.md
│       ├── agents/
│       │   └── openai.yaml
│       ├── scripts/          (optional)
│       ├── references/       (optional)
│       └── assets/           (optional)
└── themes/                   (optional)
    └── *.tmTheme
```

### 6.2 Manifest Schema

The manifest is the bundle's identity and metadata. It is auto-generated by the CLI during `bazaar upload` and must not be hand-edited.

| Field | Type | Description |
|---|---|---|
| `schema_version` | integer | Manifest format version. Currently `1`. |
| `ide` | string | Target IDE identifier. `"codex"` for v1. |
| `name` | string | Human-readable bundle name (provided by the author). |
| `description` | string | Free-text description of what the config provides. |
| `code` | string | Auto-generated bundle code. Assigned by the server on first upload. `null` for drafts. |
| `author` | object | `{ "id": "<user-id>", "name": "<display-name>" }` |
| `parent` | string or null | Bundle code of the parent bundle if this is a fork. `null` for originals. |
| `visibility` | string | `"public"` or `"private"`. |
| `created_at` | ISO 8601 string | Timestamp of initial creation. |
| `updated_at` | ISO 8601 string | Timestamp of last update. |
| `version` | integer | Monotonically increasing version number, incremented on each upload. |
| `includes` | object | Summary of what the bundle contains: `{ "config": true, "agents_md": true, "rules": ["default.rules"], "skills": ["my-skill"], "themes": [] }` |
| `tags` | array of strings | Author-assigned tags (e.g., `["python", "devops", "minimal"]`). |
| `codex_version_hint` | string or null | The Codex CLI version the author was using when packaging. Informational only. |

### 6.3 Source Files and Target Mapping

When installing a bundle, the CLI maps bundle paths to system paths:

| Bundle Path | System Target |
|---|---|
| `config.toml` | `~/.codex/config.toml` |
| `AGENTS.md` | `~/.codex/AGENTS.md` |
| `AGENTS.override.md` | `~/.codex/AGENTS.override.md` |
| `rules/*.rules` | `~/.codex/rules/` |
| `skills/<name>/` | `~/.agents/skills/<name>/` |
| `themes/*.tmTheme` | `~/.codex/themes/` |

### 6.4 Sanitization Rules

Before packaging, the CLI must sanitize the configuration to prevent accidental secret leakage:

**Always excluded (never packaged):**
- `~/.codex/auth.json` — authentication credentials
- `~/.codex/history.jsonl` — session history
- `~/.codex/log/` — log files
- Any file matching common secret patterns: `*.pem`, `*.key`, `credentials.*`, `.env`

**Redacted in `config.toml`:**
- `mcp_servers.<id>.env` values — replaced with placeholder `"<REDACTED: set your value>"` while keeping the key names visible so consumers know which env vars to set.
- `model_providers.<id>.experimental_bearer_token` — removed entirely.
- Any value that matches heuristic patterns for API keys, tokens, or secrets (strings starting with `sk-`, `ghp_`, `gho_`, `xoxb-`, etc.) — removed with a warning to the user.

**Preserved as-is:**
- `mcp_servers.<id>.env_vars` — these are env var *names* to forward, not values.
- `model_providers.<id>.env_key` — this is an env var *name*, not a value.
- `model_providers.<id>.env_http_headers` — maps header names to env var *names*.

**Interactive review:** After sanitization, the CLI presents a summary of what will be included and what was redacted, and asks for confirmation before uploading.

---

## 7. Web Application (Marketplace)

### 7.1 Authentication

- Google OAuth 2.0 is the sole authentication method.
- On successful authentication, the server issues a session (for the web app) and can generate API tokens (for the CLI).
- User profile is created on first login: display name, email, avatar sourced from Google.

### 7.2 Pages and Features

**Landing / Browse Page**
- Search bar with full-text search across bundle names, descriptions, and tags.
- Filter by tags, visibility (public only for non-authors), and sort by (newest, most installed, recently updated).
- Each result card shows: bundle name, author avatar + name, short description, tag chips, install count, bundle code, and creation date.

**Bundle Detail Page**
- Full description.
- Author info with link to their profile.
- Bundle code prominently displayed with a copy button.
- "Included in this bundle" section listing which file categories are present (config, agents instructions, N rules files, N skills, N themes) without revealing file contents.
- If the bundle is a fork: link to the parent bundle.
- If the bundle has forks: list of known forks.
- Version history (list of versions with timestamps).
- Install count.
- For the author: edit description, change visibility, delete bundle.
- For consumers: if public, display the bundle code and CLI install instructions. If private and no access, show a "Request Access" button.

**Author Dashboard**
- List of the user's published bundles.
- Pending access requests with approve / deny actions.
- Per-bundle analytics: install count, access request count.

**User Profile Page (public)**
- Display name, avatar.
- List of public bundles by this author.

### 7.3 Bundle Code Generation

- Format: `bz-` prefix followed by 6 lowercase alphanumeric characters (e.g., `bz-a1b2c3`).
- Generated server-side on first upload.
- Guaranteed unique via collision check against existing codes.
- Immutable once assigned.

---

## 8. CLI Tool

The CLI is a Python package named `bazaar` (installed via pip). It manages the full lifecycle of configuration bundles on the user's machine.

### 8.1 Local Directory Structure

```
~/.bazaar/
├── credentials.json          # OAuth token for API access
├── state.json                # Tracks installed bundle, version, history
└── codex/
    ├── bundles/
    │   └── <bundle-code>/    # Downloaded bundle contents (git working tree)
    └── backups/
        └── <timestamp>/      # Snapshot of config state before last install
```

### 8.2 Commands

#### `bazaar login`

1. Opens the user's default browser to the Bazaar web app's OAuth page.
2. The web app authenticates the user via Google OAuth.
3. On success, the web app redirects to a localhost callback URL where the CLI is listening.
4. The CLI captures the API token from the callback and writes it to `~/.bazaar/credentials.json`.
5. Prints confirmation with the authenticated user's display name.

If the user is already authenticated, prints the current identity and exits.

#### `bazaar logout`

1. Deletes `~/.bazaar/credentials.json`.
2. Prints confirmation.

Does not revoke the token server-side (the token simply becomes unusable on expiry). Does not remove installed configs or local bundle data.

#### `bazaar init`

1. Checks authentication. If not authenticated, runs the login flow first.
2. Creates the directory structure: `~/.bazaar/codex/bundles/` and `~/.bazaar/codex/backups/`.
3. Creates `~/.bazaar/state.json` with default empty state.
4. Prints confirmation.

Idempotent: running it again on an already-initialized directory is a no-op with a message.

#### `bazaar upload`

Packages the user's current global Codex CLI configuration and uploads it.

1. Checks authentication.
2. Scans the following source locations:
   - `~/.codex/config.toml`
   - `~/.codex/AGENTS.md` and `~/.codex/AGENTS.override.md`
   - `~/.codex/rules/` (all `.rules` files)
   - `~/.agents/skills/` (all skill directories containing `SKILL.md`)
   - `~/.codex/themes/` (all `.tmTheme` files)
3. Applies sanitization rules (Section 6.4).
4. Checks `~/.bazaar/state.json` to determine if the user currently has an installed bundle from another author. If so, the current upload is marked as a fork with that bundle as the parent.
5. Presents a summary to the user:
   - Files to be included.
   - Fields that were redacted.
   - Warnings for any remaining potentially sensitive values.
   - Parent bundle (if fork).
6. Prompts for confirmation.
7. On confirmation, creates a temporary git repository with the bundle structure, generates or updates the manifest, and pushes to the server via the API.
8. The server assigns a bundle code (on first upload) or increments the version (on update).
9. Prints the bundle code and a shareable link.

**Required flags:**
- `--name <name>` — bundle display name (required on first upload, optional on update).
- `--description <description>` — bundle description (required on first upload, optional on update).

**Optional flags:**
- `--visibility <public|private>` — defaults to `private`.
- `--tag <tag>` — repeatable flag to attach tags. Replaces all tags on update.

#### `bazaar setup <bundle-code>`

Downloads and installs a configuration bundle.

1. Checks authentication.
2. Queries the server for the bundle metadata.
3. **Access check:**
   - If the bundle is public, proceed.
   - If the bundle is private and the user has access, proceed.
   - If the bundle is private and the user does not have access, prompt: *"You don't have access to this bundle. Would you like to request access from the author?"* If yes, send an access request via the API and exit.
4. Downloads the bundle contents (git clone or pull if already present locally).
5. Displays a pre-install summary:
   - What files will be placed and where.
   - Any skills containing executable scripts (security warning).
   - Any MCP server configurations pointing to external URLs (security warning).
6. Prompts for confirmation.
7. **Backup:** Snapshots the current state of all target locations into `~/.bazaar/codex/backups/<timestamp>/`.
8. **Install:** Copies/symlinks bundle files to their target system locations (see Section 6.3 mapping).
9. Updates `~/.bazaar/state.json` to record the active bundle code and version.
10. Prints confirmation with any post-install notes (e.g., "Set the following environment variables for your MCP servers: ...").

#### `bazaar list`

Lists the user's published bundles.

- Shows: name, bundle code, visibility, version, install count, last updated.
- Flag `--installed` shows the currently installed bundle instead.

#### `bazaar preview <bundle-code>`

Shows detailed information about a bundle without installing it.

- Displays: name, author, description, tags, included file categories, parent (if fork), version, install count.
- If the user has access, also lists the specific files included (filenames only, not contents).

#### `bazaar backup`

Manually creates a backup of the current global Codex CLI config state into `~/.bazaar/codex/backups/<timestamp>/`.

#### `bazaar restore [timestamp]`

Restores global config from a backup.

- Without a timestamp argument, restores the most recent backup.
- With a timestamp, restores that specific backup.
- Prompts for confirmation before overwriting current config.
- Updates `state.json` to clear the active bundle.

#### `bazaar uninstall`

Removes the currently active bundle and restores the most recent backup.

1. Checks that a bundle is currently active (via `state.json`).
2. Equivalent to `bazaar restore` with the backup created during the last `setup`.
3. Clears the active bundle from `state.json`.

---

## 9. Data Model

### 9.1 Users

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key. |
| `google_id` | string | Google OAuth subject identifier. Unique. |
| `email` | string | From Google profile. |
| `display_name` | string | From Google profile, editable. |
| `avatar_url` | string | From Google profile. |
| `created_at` | timestamp | |
| `updated_at` | timestamp | |

### 9.2 Bundles

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key. |
| `code` | string | Unique bundle code (e.g., `bz-a1b2c3`). Immutable after creation. |
| `author_id` | UUID | FK → Users. |
| `name` | string | Display name. |
| `description` | text | Free-text description. |
| `visibility` | enum | `public`, `private`. |
| `parent_id` | UUID or null | FK → Bundles. The bundle this was forked from. |
| `ide` | string | IDE identifier. `codex` for v1. |
| `current_version` | integer | Latest version number. |
| `repo_path` | string | Server-side path to the bare git repository. |
| `install_count` | integer | Total installs. |
| `created_at` | timestamp | |
| `updated_at` | timestamp | |

### 9.3 Bundle Versions

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key. |
| `bundle_id` | UUID | FK → Bundles. |
| `version` | integer | Sequential version number. |
| `commit_hash` | string | Git commit SHA in the bundle's bare repo. |
| `includes_summary` | JSONB | Summary of included file categories. |
| `created_at` | timestamp | |

### 9.4 Tags

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key. |
| `name` | string | Lowercase, unique. |

### 9.5 Bundle Tags (join table)

| Field | Type | Notes |
|---|---|---|
| `bundle_id` | UUID | FK → Bundles. |
| `tag_id` | UUID | FK → Tags. |

### 9.6 Access Grants

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key. |
| `bundle_id` | UUID | FK → Bundles. |
| `user_id` | UUID | FK → Users. |
| `granted_at` | timestamp | |
| `granted_by` | UUID | FK → Users (the author who approved). |

### 9.7 Access Requests

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key. |
| `bundle_id` | UUID | FK → Bundles. |
| `requester_id` | UUID | FK → Users. |
| `status` | enum | `pending`, `approved`, `denied`. |
| `message` | text or null | Optional message from the requester. |
| `resolved_by` | UUID or null | FK → Users. |
| `created_at` | timestamp | |
| `resolved_at` | timestamp or null | |

---

## 10. API Surface

All API endpoints are served by the Next.js application under `/api/`. Authentication is via Bearer token in the `Authorization` header (the token issued during the OAuth flow).

### 10.1 Authentication

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/auth/google` | Initiates Google OAuth flow. Redirects to Google. |
| `GET` | `/api/auth/google/callback` | OAuth callback. Issues session + API token. |
| `POST` | `/api/auth/cli-token` | Exchanges a session for a long-lived CLI API token. |

### 10.2 Bundles

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/bundles` | List/search bundles. Query params: `q` (search), `tag`, `sort`, `page`, `limit`. Returns only public bundles (plus the caller's own private bundles). |
| `GET` | `/api/bundles/:code` | Get bundle metadata by code. Returns 403 if private and caller has no access. |
| `POST` | `/api/bundles` | Create a new bundle. Body: multipart with manifest + bundle archive. Returns the assigned bundle code. |
| `PUT` | `/api/bundles/:code` | Update an existing bundle (new version). Author only. |
| `DELETE` | `/api/bundles/:code` | Delete a bundle and its git repo. Author only. |
| `GET` | `/api/bundles/:code/download` | Download the bundle archive (latest version). Requires access. |
| `GET` | `/api/bundles/:code/download/:version` | Download a specific version. Requires access. |
| `GET` | `/api/bundles/:code/versions` | List version history for a bundle. |
| `GET` | `/api/bundles/:code/forks` | List bundles that are forks of this one. |

### 10.3 Access Control

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/bundles/:code/access-requests` | Request access to a private bundle. |
| `GET` | `/api/bundles/:code/access-requests` | List access requests for a bundle. Author only. |
| `PUT` | `/api/bundles/:code/access-requests/:id` | Approve or deny an access request. Author only. |
| `DELETE` | `/api/bundles/:code/access/:user-id` | Revoke a user's access. Author only. |

### 10.4 User

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/me` | Get the authenticated user's profile. |
| `GET` | `/api/me/bundles` | List the authenticated user's bundles. |
| `GET` | `/api/me/access-requests` | List access requests the user has sent. |
| `GET` | `/api/users/:id` | Get a public user profile. |
| `GET` | `/api/users/:id/bundles` | List a user's public bundles. |

---

## 11. Security Considerations

### 11.1 Secret Leakage Prevention

The CLI's sanitization step (Section 6.4) is the primary defense. It operates on a deny-by-default philosophy for known sensitive patterns. The interactive review step before upload serves as a second line of defense.

The server does **not** inspect or validate bundle contents for secrets. This is intentionally a client-side responsibility to keep the server simple and avoid false positives. A future enhancement could add server-side scanning as an advisory layer.

### 11.2 Malicious Content

Bundles can contain:

- **Skills with executable scripts:** The `scripts/` directory in a skill can contain arbitrary executables. The CLI must warn the user during `bazaar setup` if any installed skills contain scripts, and list the script filenames.
- **MCP server configurations:** `config.toml` can reference external MCP servers (via `command` for stdio or `url` for HTTP). These could point to malicious servers. The CLI must list all MCP server configurations during `bazaar setup` and highlight any external URLs.
- **Rules that weaken security:** A bundle's `config.toml` could set `approval_policy = "never"` or `sandbox_mode = "danger-full-access"`. The CLI must detect and warn about security-weakening settings during `bazaar setup`.
- **Agent instructions:** `AGENTS.md` could contain instructions that direct the AI to perform harmful actions. This is inherent to the Codex platform and outside Bazaar's scope to fully mitigate, but the pre-install summary should make the presence of agent instructions visible.

### 11.3 Authentication and Authorization

- API tokens are opaque, long-lived (configurable expiry), and revocable.
- All state-changing operations require authentication.
- Bundle downloads for private bundles require both authentication and an access grant.
- Authors can only modify or delete their own bundles.
- Access requests are visible only to the bundle author and the requester.

### 11.4 Transport Security

All communication between the CLI and the server uses HTTPS. The server must enforce TLS.

---

## 12. User Flows

### 12.1 First-Time Author Publishing a Config

1. User installs the CLI: `pip install bazaar`
2. User runs `bazaar login` → browser opens, authenticates with Google, CLI receives token.
3. User runs `bazaar init` → local directory structure created.
4. User runs `bazaar upload --name "My Python Setup" --description "Optimized for Python data science with MCP tools" --visibility public --tag python --tag data-science`
5. CLI scans config files, sanitizes, shows summary, user confirms.
6. Bundle is uploaded. CLI prints: *"Published as bz-k7x2m9. View at https://bazaar.example.com/bundles/bz-k7x2m9"*

### 12.2 Consumer Installing a Public Config

1. User discovers `bz-k7x2m9` on the marketplace website.
2. User installs CLI and runs `bazaar login`.
3. User runs `bazaar preview bz-k7x2m9` to see what's included.
4. User runs `bazaar setup bz-k7x2m9`.
5. CLI shows pre-install summary with warnings. User confirms.
6. CLI backs up current config, installs bundle files, updates state.
7. User opens Codex CLI — the new config is active.

### 12.3 Consumer Requesting Access to a Private Config

1. User runs `bazaar setup bz-x9y3z1`.
2. CLI reports: *"This bundle is private. You don't have access."*
3. CLI prompts: *"Would you like to request access?"* User confirms.
4. Access request is sent. CLI prints: *"Access request sent. You'll be notified when the author responds."*
5. Author sees the request in their dashboard and approves it.
6. User runs `bazaar setup bz-x9y3z1` again — this time it proceeds.

### 12.4 Author Updating a Bundle

1. Author modifies their Codex CLI config locally (edits `config.toml`, adds a new skill, etc.).
2. Author runs `bazaar upload --name "My Python Setup"`.
3. CLI detects this is an update (the user is the author of the bundle recorded in `state.json`).
4. A new version is created. Existing consumers can update by running `bazaar setup <code>` again.

### 12.5 Forking a Config

1. User installs bundle `bz-k7x2m9` via `bazaar setup`.
2. User modifies their local Codex config (adds skills, changes model, etc.).
3. User runs `bazaar upload --name "My Fork" --description "Based on My Python Setup with extra DevOps tools"`.
4. CLI detects the active bundle is from another author → marks the new bundle as a fork with `parent: bz-k7x2m9`.
5. The new bundle appears on the marketplace with a "Forked from bz-k7x2m9" attribution.

### 12.6 Rolling Back

1. User runs `bazaar setup bz-abc123` and realizes it doesn't suit them.
2. User runs `bazaar uninstall`.
3. CLI restores the backup taken during setup. The previous config is active again.

---

## 13. State File Schema

The file `~/.bazaar/state.json` tracks the CLI's local state.

| Field | Type | Description |
|---|---|---|
| `active_bundle` | object or null | `{ "code": "bz-...", "version": N, "installed_at": "ISO8601" }` — the currently active bundle, or null if none. |
| `last_backup` | string or null | Timestamp of the most recent backup. |
| `known_bundles` | array | List of `{ "code": "bz-...", "name": "...", "role": "author" | "consumer" }` for bundles the user has interacted with. |
| `ide` | string | `"codex"` for v1. |

---

## 14. Open Questions

1. **Notifications:** How should the author be notified when someone requests access? Email? In-app only? The web app should support in-app notifications at minimum; email notifications are a follow-up.

2. **Bundle size limits:** What is the maximum allowed bundle size? Skills with large reference docs or assets could be heavy. A reasonable default might be 10 MB per bundle.

3. **Search backend:** Full-text search over bundle names, descriptions, and tags. For v1, PostgreSQL full-text search is sufficient. A dedicated search engine (e.g., Meilisearch, Typesense) can be introduced if needed.

4. **CLI token refresh:** Should CLI tokens have a fixed long expiry (e.g., 90 days) with re-login, or should they use a refresh token flow? For simplicity in v1, long-lived tokens with re-login on expiry.

5. **Multiple active bundles:** The current design supports only one active bundle at a time (full replacement). A future version could support composing multiple bundles (e.g., one for MCP servers, one for skills, one for rules) using Codex's profile and layering system.

6. **Bundle verification / trust:** Should the marketplace support verified authors or bundle signing? Deferred to a future version, but the architecture should not preclude it.

---

## 15. Future Considerations

- **Paid bundles:** Introduce Stripe-based payments for premium configs. Authors set a price; consumers pay to gain access. Revenue split between author and platform.
- **IDE expansion:** Add support for Cursor, Windsurf, and other AI IDEs. Each IDE gets its own namespace, file mappings, and packaging logic in the CLI.
- **Composable bundles:** Allow users to install multiple bundles that contribute to different parts of the config (e.g., one bundle for skills only, another for MCP servers only) with intelligent merging.
- **Community features:** Ratings, reviews, comments, and curated collections on the marketplace.
- **Bundle dependencies:** A bundle could declare that it depends on specific MCP servers, npm packages, or system tools, and the CLI could validate or install them.
- **Automatic updates:** Opt-in mechanism for consumers to automatically receive new versions of bundles they've installed.
- **Web-based diff viewer:** Show what changed between bundle versions directly on the marketplace website.
- **Config playground:** A web-based preview of what a Codex session would look like with a given config applied (e.g., which model, which tools, which rules).
