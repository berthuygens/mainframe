# DAEMON Dashboard

A personal developer dashboard that aggregates GitHub issues/PRs, Google Calendar events, OTOBO helpdesk tickets, Reddit posts, and CCB security advisories into a single view. Built as a static single-page application hosted on GitHub Pages with a Cloudflare Worker backend for OAuth and API proxying.

## Features

- **GitHub Issues & PRs**: View assigned issues, created issues, and pull requests with tab navigation and live counts
- **Watch All PRs**: Monitor PRs from specific repos (configurable in settings)
- **Repo Search**: Real-time search across configurable GitHub users/orgs with keyboard navigation
- **Google Calendar**: Next 4 upcoming events with automatic token refresh — authenticate once, never reconnect
- **OTOBO Helpdesk**: View open tickets with full conversation thread in a detail popup (most recent article first)
- **CCB Security Advisories**: Animated typewriter ticker with the latest advisories from the Centre for Cybersecurity Belgium
- **Quick Links**: User-configurable links with drag & drop reordering and an icon picker (25 icons available)
- **Reddit Posts**: Random posts from SFW subreddits with thumbnails, score, and a refresh button
- **Dark/Light Mode**: Theme toggle with preference saved locally
- **Profile Picture**: Configurable avatar with custom link
- **Auto-refresh**: All widgets refresh every 5 minutes

## Live

- https://automate.terminals.work (custom domain)
- https://berthuygens.github.io/daemon/ (GitHub Pages fallback)

## Setup

### GitHub Token

1. Go to [GitHub Settings > Personal Access Tokens](https://github.com/settings/tokens)
2. Click **Generate new token (classic)**
3. Select scopes:
   - `repo` — Access to private repositories (issues, PRs)
   - `read:user` — Read your profile information
4. Copy the generated token (starts with `ghp_`)
5. Open the dashboard, click the settings icon (gear)
6. Paste your token and save

### Google Calendar (with Cloudflare Worker)

The dashboard uses a Cloudflare Worker to handle OAuth token refresh automatically. Authenticate once — the Worker handles token refresh from then on.

#### Step 1: Google Cloud Setup

1. Go to [Google Cloud Console](https://console.cloud.google.com/apis/credentials)
2. Create a new project (or select existing)
3. Enable the **Google Calendar API**
4. Create **OAuth 2.0 Client ID** credentials:
   - Application type: Web application
   - Authorized JavaScript origins: `https://automate.terminals.work`
   - Authorized redirect URIs: `https://YOUR-WORKER.workers.dev/callback`
5. Note down your **Client ID** and **Client Secret**

#### Step 2: Deploy Cloudflare Worker

```bash
# Install wrangler CLI
npm install -g wrangler

# Login to Cloudflare
wrangler login

# Navigate to worker directory
cd cloudflare-worker

# Create KV namespace for token storage
wrangler kv namespace create "OAUTH_TOKENS"
# Copy the ID and update wrangler.toml

# Set secrets
wrangler secret put GOOGLE_CLIENT_ID
wrangler secret put GOOGLE_CLIENT_SECRET
wrangler secret put API_KEY           # Bearer token for frontend → worker auth
wrangler secret put OTRS_USERNAME     # OTOBO API user
wrangler secret put OTRS_PASSWORD     # OTOBO API password

# Deploy
wrangler deploy
```

#### Step 3: Configure Dashboard

1. Open the dashboard settings (gear icon)
2. Enter your Worker URL: `https://YOUR-WORKER.workers.dev`
3. Enter the API Key (same value as the `API_KEY` secret)
4. Click "Connect Google Calendar"
5. Authorize with Google

You should now see `✓ Auto` in the calendar header — tokens refresh automatically!

### OTOBO Helpdesk

The Worker proxies requests to the OTOBO REST API. No additional dashboard configuration is needed beyond the Worker URL and API key.

Worker secrets required: `OTRS_USERNAME` and `OTRS_PASSWORD` (see Step 2 above).

**Note:** `SessionCheckRemoteIP` must be disabled in OTOBO since Cloudflare Workers use different egress IPs per request.

### Dashboard Settings

All settings are accessible via the gear icon:

| Setting | Description |
|---------|-------------|
| GitHub Token | Personal access token for issues/PRs |
| Worker URL | Cloudflare Worker base URL |
| API Key | Bearer token for Worker authentication |
| Watch All PRs | Comma-separated `owner/repo` list to monitor |
| Search Locations | GitHub search scopes (`user:x` or `org:y`) |
| Profile Picture URL | Avatar image URL |
| Profile Link URL | Link target when clicking the avatar |
| Quick Links | Add, remove, reorder, and pick icons for links |

### Calendar Filtering

The following events are automatically filtered out (work locations):
- home, thuis
- office, kantoor, bureau
- HT BXL, VAC GENT

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  GitHub Pages (Static Frontend)                         │
│  https://automate.terminals.work                        │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTPS requests
                       ▼
┌─────────────────────────────────────────────────────────┐
│  Cloudflare Worker (API Gateway)                        │
│  - /auth          → Redirect to Google OAuth            │
│  - /callback      → Exchange code for tokens            │
│  - /token         → Get fresh access token              │
│  - /status        → Check authentication status         │
│  - /logout        → Remove stored tokens                │
│  - /rss           → RSS proxy for CCB advisories        │
│  - /otrs/tickets  → OTOBO ticket proxy                  │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│  External APIs                                          │
│  - GitHub REST API (issues, PRs, search)                │
│  - Google Calendar API v3                               │
│  - Google OAuth 2.0                                     │
│  - OTOBO REST API (tickets, articles)                   │
│  - CCB RSS Feed (security advisories)                   │
│  - Reddit JSON API (SFW subreddits)                     │
└─────────────────────────────────────────────────────────┘
```

## Privacy & Security

- **GitHub token**: Stored in browser localStorage only
- **Google refresh token**: Stored server-side in Cloudflare KV (encrypted at rest)
- **Google access token**: Kept in-memory only — never persisted to localStorage
- **OTOBO credentials**: Stored as Cloudflare Worker secrets (server-side only)
- **Worker endpoints**: Protected by API key (Bearer token authentication)
- **CSP**: Restrictive `Content-Security-Policy` with `default-src 'none'` and whitelisted connect-src
- **XSS prevention**: All user-controlled and API-sourced values escaped via `escapeHtml()`; URL protocol validation on quick links
- **CSRF protection**: OAuth state parameter with `crypto.randomUUID()` and 5-minute TTL
- **CORS**: Worker validates `Origin` header against an allowlist
- **No tracking**: No analytics or tracking scripts

## Local Development

```bash
python3 -m http.server 8080
```

Then open http://localhost:8080

## Tech Stack

- Vanilla HTML/CSS/JavaScript (single-file, no build system)
- HTML5 Drag & Drop API
- Cloudflare Workers + KV
- GitHub REST API
- Google Calendar API v3
- OTOBO REST API
- Reddit JSON API
- Hosted on GitHub Pages with Cloudflare custom domain

## License

MIT
