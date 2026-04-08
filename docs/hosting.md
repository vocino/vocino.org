# Hosting (Cloudflare) — single-user setup

Organizing Vocino is **single-tenant**: one operator, low traffic, no multi-customer isolation layer. This doc picks the **simplest** Cloudflare layout that still follows **Workers best practices** (bindings, secrets, compatibility date, no secrets in git).

For product behavior and data model, see the main [README](../README.md). For UI tokens, see [design.md](design.md).

---

## Recommended shape: one Worker + static assets + bindings

Deploy **one Cloudflare Worker** that:

1. Serves the **built dashboard** (Vite/React `dist/`) as **static assets** attached to the same Worker.
2. Handles **API routes** in the same script (e.g. under `/api/*` or by method + path).

**Why this is enough for you**

- **Same origin** — Browser talks to one hostname; session cookies and OAuth redirects stay straightforward (no CORS between “site” and “API”).
- **One `wrangler deploy`** — No separate Pages project to wire up unless you prefer that workflow later.
- **Scale** — A single user and a handful of scheduled publishes per day is far below Worker limits; no need for Queues or multiple services on day one.

**Reasonable alternative:** **Cloudflare Pages** with **Functions** for the API, plus the same D1 / DO bindings Pages supports. Slightly more split mental model; use it if you already standardize on Pages for frontends.

---

## Bindings (what talks to what)

| Binding | Purpose for this app |
|---------|----------------------|
| **D1** | Queue rows, schedule config, publish history, OAuth token references (store tokens encrypted or as opaque handles—see README security). **One database** is enough. |
| **Durable Object** | **One logical queue / scheduler coordinator** for your account. Use a **fixed id** (e.g. `idFromName("default")` or `"vocino"`) so every request and cron tick hits the same DO instance. |
| **Cron Triggers** | Fire on a schedule: e.g. **every minute or every 5 minutes** to evaluate “is it time for the next slot?” and optional **gap-since-last-post** rules. Keep handlers **idempotent** (safe if cron runs twice). |
| **R2** *(optional)* | Media for Threads posts. Add when you need uploads; skip until then. |
| **Cloudflare Queues** | **Not required for V1.** Introduce only if you outgrow “cron + DO” (retries, fan-out, heavy background work). |

External calls (Meta Threads API) go from the Worker over `fetch`; use normal HTTPS, timeouts, and structured error logging.

---

## Scheduling (simple mental model)

1. **Cron** invokes your Worker (or a dedicated route) on an interval.
2. Worker forwards work to the **Durable Object** (HTTP RPC or `stub.fetch`).
3. DO reads/writes **D1**, decides whether to publish, calls Threads API, updates state.

Keeping **serialization of “who publishes next”** inside the DO avoids race conditions without adding Queues.

---

## Configuration & hygiene (best practices, minimal)

- **`wrangler.jsonc`** (or `wrangler.toml`) — Version in git; **no secrets** in file.
- **`compatibility_date`** — Set to a recent date when you create the project; bump periodically when you adopt new runtime behavior ([Workers compatibility dates](https://developers.cloudflare.com/workers/configuration/compatibility-dates/)).
- **`nodejs_compat`** — Enable if you use npm packages that expect Node built-ins.
- **Secrets** — `wrangler secret put …` for Meta client secret, session signing secret, encryption key material, etc. Never commit values.
- **`wrangler types`** — Generate `Env` from bindings; avoid hand-maintained binding types drifting from config.
- **Observability** — Turn on Workers **Logs** (and tracing if you use it); log **structured JSON** with request ids, never raw OAuth tokens.

---

## Domain & TLS

- Attach a **custom hostname** on **vocino.org** (e.g. apex or `app.`) to this Worker route.
- **OAuth redirect URLs** in the Meta app must match the **exact** public URL you deploy (including `https` and path).
- TLS is handled by Cloudflare; no extra certs to manage.

**vocino.com** can stay your personal/marketing site on its current stack; **vocino.org** (or a subdomain) is a clean split for the dashboard.

---

## Environments

For a single user, **production + local dev** is enough:

- **Production** — Real bindings, real secrets, custom domain.
- **Local** — `wrangler dev` with **local D1** / **remote** D1 depending on what you prefer; `.dev.vars` for local-only secrets (gitignored).

A separate **staging** Worker + D1 is optional; many solo projects skip it until OAuth or migrations get painful.

---

## What we are intentionally not doing (yet)

- **Multiple Workers** wired with service bindings — unnecessary until the app splits into separate deployable units.
- **Per-tenant DO namespaces** — you are the only tenant.
- **Kubernetes, VMs, or a separate database host** — D1 + DO keep everything on Cloudflare for this use case.
- **Aggressive rate limiting / WAF tuning** — optional; turn on basics in the dashboard if the Worker is ever public-facing and you see abuse.

---

## Checklist before first deploy

- [ ] Worker serves SPA **and** API from the chosen URL shape.
- [ ] D1 schema migrated (`wrangler d1 migrations apply`).
- [ ] DO class bound and **stable name** chosen for your singleton coordinator.
- [ ] Cron schedule registered and handler tested (including “nothing to publish”).
- [ ] Meta app redirect URIs match production URL.
- [ ] All secrets in **Secrets** store, not repo.
- [ ] README pre-push / secret hygiene still satisfied.

---

## References

- [Cloudflare Workers — Best practices](https://developers.cloudflare.com/workers/best-practices/workers-best-practices/)
- [Wrangler configuration](https://developers.cloudflare.com/workers/wrangler/configuration/)
- [Durable Objects](https://developers.cloudflare.com/durable-objects/)
- [D1](https://developers.cloudflare.com/d1/)
- [Cron Triggers](https://developers.cloudflare.com/workers/configuration/cron-triggers/)

When implementation lands in this repo, replace generic paths (`/api/*`, `dist/`) with the actual routes and build output your tooling uses.
