# Organizing Vocino

**A personal-first social publishing dashboard for professional channels.**

Organizing Vocino is a small app for planning, scheduling, and publishing posts across the social accounts that matter for your work—starting with **Threads** and **LinkedIn**. Think of it as a focused alternative to juggling multiple tabs and native schedulers: one place to compose, queue, and track what went out.

The project lives at [vocino.org](https://vocino.org) and is built for **Vocino’s own workflow first**. The code is open source so others can learn from it, fork it, or self-host with their own API credentials—without routing traffic through anyone else’s keys or quotas.

---

## Why this exists

Posting consistently across professional networks is fragmented: each platform has different limits, media rules, and scheduling UX. This repo is an experiment in **owning that workflow**: a single dashboard that scrolls through accounts and content state, similar in spirit to tools like Buffer—but scoped to what actually gets used for work, with room to grow.

Starting with **Threads** and **LinkedIn** keeps V1 realistic while covering two high-signal professional surfaces.

---

## V1 features (planned)

- **Connections** — Link Threads and LinkedIn accounts via platform OAuth (credentials stored only in your environment / Cloudflare secrets, not in the repo).
- **Composer** — Draft posts; optional per-platform variants (length, mentions, media).
- **Scheduling & queue** — Schedule publishes and process them through a reliable queue + triggers.
- **Status** — Minimal visibility: published, failed, retry—enough to trust the system without building a full analytics product.

The subsections below capture **target product behavior** for each V1 platform (similar in spirit to mature schedulers such as Buffer), always bounded by what each vendor’s **API and app review** actually allow. **Threads** is Meta Graph–centric and tied to Instagram; **LinkedIn** is the more policy- and matrix-heavy surface.

### Threads integration (target behavior)

**Accounts & connection**

- Authenticate via **Meta** (Threads is tied to **Instagram**): user must have a Threads profile linked to Instagram as Meta expects.
- Typically **public** Threads accounts only for third-party publishing flows—do not promise private-profile support unless the API explicitly allows it.
- Request the minimum OAuth scopes your app needs (e.g. account/read vs publish); product docs often cite scopes such as **`threads_basic`** and **`threads_content_publish`**—confirm names and combinations against [Meta’s current Threads API documentation](https://developers.facebook.com/docs/threads).

**Publishing features**

| Spec | Target |
|------|--------|
| Max text length | **500 characters** (enforce in composer). |
| Images / videos per post | Up to **10** items (carousel) where the API allows. |
| Video length | Up to **~5 minutes** (validate server-side). |
| Video aspect ratio | Roughly **0.01:1** through **10:1** (reject out-of-range uploads early when possible). |
| Thread chains | Support scheduling a **thread** (sequence of posts); product goal **up to ~50** posts in one scheduled thread, published in order with correct reply/parent linkage per API. |
| Mentions | **`@username`** in copy where the API supports mentions. |
| Links | URLs in text become **clickable links** when the platform renders them (no fake “preview builder” unless the API exposes one). |
| Direct scheduling | **Yes** — publish at scheduled time via API (no “mobile notification tap to post” workflow as the primary path). |

**Known limitations (communicate in UI)**

- **No in-dashboard analytics** for Threads in V1 if the API does not expose performance metrics you are allowed to use.
- **No engagement inbox** — no replying, liking, or unified “Threads inbox” from this app unless APIs and scope expand later.
- **No image alt text** if the publishing API does not support it.
- **No platform “tags” / location** features that Meta does not expose to third-party publishers.
- **No “first comment”** scheduling as a separate feature (unlike LinkedIn-style flows)—Threads behavior is thread-of-posts instead.
- **Rate limits / throttling** — same discipline as LinkedIn: backoff, spacing, clear errors.

**Workflow**

- **Drafts** and **scheduled** posts (and **multi-post threads**) are in scope; define queue semantics so partial thread failure is visible and recoverable.
- **Analytics** and **engagement** are **out of scope** for V1 unless you later add scopes and storage for data Meta actually returns.

---

### LinkedIn integration (target behavior)

**Accounts & connection**

- Support **LinkedIn Profile** and **LinkedIn Page** destinations where the API permits.
- For **Pages**, assume the connecting user has sufficient admin (e.g. Super Admin / Content Admin—exact names depend on LinkedIn’s product UI).
- Use **OAuth**; expect **periodic re-authentication** when refresh fails or policies expire (often on the order of weeks, not “set and forget forever”).

**Post types (profiles vs pages)**

Exact matrix depends on LinkedIn API version and app approval; the **product goal** is parity where the platform allows:

| Capability | Target |
|------------|--------|
| Text | Yes, enforce **~3,000 characters** (validate in composer). |
| Images | Yes; **up to 9** images per post where supported. |
| Video | Yes where supported. |
| Links | Yes; **link preview** (title/description/thumbnail) when the API exposes it. |
| First comment | Yes as a **second step**: publish main post, then post the “first comment” once the parent id is known (same scheduling story, two API actions). |
| @mentions | **Pages only** where the API allows; **do not promise** @mention of personal profiles (LinkedIn restricts this for third parties). |
| Image alt text | Yes for accessibility when the API supports it. |
| Document / PDF carousel | Yes when the API supports document posts. |

**Media validation (backend / UX guardrails)**

Use these as defaults to fail fast before upload; adjust when LinkedIn’s docs change.

- **Images:** e.g. **≤ 10 MB**; **JPG, PNG, non-animated GIF**; aspect ratio guidance **~1.91:1 to 4:5** where relevant.
- **Video:** e.g. **≤ 200 MB**; **~3 s–10 min** duration; common containers (**MP4**, **WebM**, **MKV**, etc.); resolution roughly **256×144** through **4096×2304**; frame rate **~10–60 fps**.

**Known limitations (communicate in UI)**

- No **Stories** or **Live** via third-party APIs (do not surface as options).
- **Throttling / rate limits** — backoff, queue spacing, and clear user-visible errors when LinkedIn returns quota or abuse signals.
- **No personal-profile @mentions** if the API cannot deliver them.

**Workflow**

- **Drafts** and **scheduled** publishes (specific datetime + queue) are in scope for V1.
- **Multi-user approvals** remain out of scope for now (see Non-goals); solo draft → schedule → publish is the path.
- **Analytics** (likes, comments, impressions, clicks) is a **stretch** after reliable publish + status; depends on API access and storage.

---

## Non-goals (for now)

- Full **SaaS** onboarding, billing, or multi-tenant product polish.
- **Every** social platform on day one (Instagram, Discord, X, etc. come later if they still fit).
- **Team** collaboration (roles, approvals, shared calendars)—solo / personal use first.

---

## Architecture (Cloudflare-first)

The intended stack for the hosted instance is **Cloudflare**:

| Piece | Role |
|--------|------|
| **Workers** | HTTP API, auth callbacks, webhooks if needed |
| **Durable Objects** | Coordination for publish queue / per-user or global sequencing |
| **D1** | Metadata: drafts, schedules, publish attempts, platform account refs |
| **R2** *(optional)* | Media uploads / attachments |
| **Queues + Cron** | Scheduled publishing and retries |

```mermaid
flowchart TD
  user[UserDashboard] --> composer[PostComposer]
  composer --> queue[PublishQueueDO]
  queue --> scheduler[CronOrQueueTrigger]
  scheduler --> adapters[PlatformAdapters]
  adapters --> threads[ThreadsAPI]
  adapters --> linkedin[LinkedInAPI]
  queue --> state[D1State]
  adapters --> media[R2MediaOptional]
```

Forks and self-hosters can swap hosting details; the README stays honest that **your** deployment is Cloudflare-first.

---

## Open source posture

- **Public repo** — Anyone can read the code. Design and copy should assume that.
- **Bring your own API credentials** — If you fork or self-host, you register apps with Meta (Threads), LinkedIn, etc., and supply secrets via environment / Cloudflare Secrets—not shared with the original author’s quotas.
- **Personal hosted path** — The primary story is: one person (Vocino) runs the real instance on Cloudflare; others clone and run their own stack if they want.

---

## Security & public-repo safeguards

This repository must **never** contain:

- API keys, client secrets, or long-lived tokens  
- `.dev.vars`, `.env` files with real values  
- Exported OAuth tokens or database dumps with PII  

**Practices:**

1. Use **Cloudflare Secrets** (and local `.dev.vars` only on disk, gitignored) for all third-party credentials.
2. Keep a **strict `.gitignore`** for env files, wrangler local state, and credential artifacts.
3. **Do not log** raw access tokens or refresh tokens; log opaque IDs and high-level errors only.
4. Prefer **short-lived tokens** and refresh flows where platforms allow; encrypt sensitive fields at rest in D1 when you store them (design goal—implement as the app lands).
5. Run **secret scanning** (e.g. GitHub push protection, `gitleaks`, or similar) before and after meaningful changes.

**Pre-push checklist (quick):**

- [ ] `git status` — no unexpected files (especially env or credential dumps).
- [ ] No secrets in diff (`git diff` / review in GitHub UI).
- [ ] Screenshots or docs redact tokens and real account handles if needed.

If anything sensitive is ever pushed, **rotate credentials immediately** and consider history cleanup per your host’s docs.

---

## Quick start (assumptions)

> The app code is not in this repo yet; this section describes the intended path once Workers + frontend exist.

1. Clone the repo and install dependencies (exact commands will live in the project root once added).
2. Create developer apps for **Threads** (Meta) and **LinkedIn**; note client ID, client secret, and redirect URLs matching your Worker routes.
3. Set secrets via `wrangler secret put …` (or the Cloudflare dashboard)—**never** commit them.
4. Apply D1 migrations and deploy Workers + any Pages frontend.

Details will be expanded when `wrangler` config and source land in the tree.

---

## Roadmap

| Phase | Focus |
|--------|--------|
| **Now** | README + repo hygiene; scaffold Cloudflare app; Threads + LinkedIn OAuth + publish MVP |
| **Next** | Instagram, Discord, or other platforms if APIs and use case still align |
| **Later** | Optional hosted offering for others (BYO keys or small fee)—out of scope until core is stable |

---

## Contributing

Issues and PRs are welcome once there is code to contribute to. Please keep changes aligned with the **public-repo security** section above.

---

## License

See [LICENSE](LICENSE).
