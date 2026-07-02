---
name: cost-analysis
description: >-
  Use this skill when the user asks for a cost audit, margin analysis,
  unit economics, "how much does this cost to run", "what's our margin",
  "are we losing money on X plan", "compare top-up packs", pricing
  health-check, or any variant of "is this priced right". Inventories
  every paid API call, SDK invocation, scraper, render provider, and
  managed service (Vercel, Resend, Supabase, Stripe, Replicate, Fal,
  Apify, Upstash, Anthropic, OpenAI, Gemini, etc.), pulls per-unit
  rates from canonical sources, then compares against the project's
  pricing model (subscription tiers, top-up packs, usage-based, free).
  Computes margins for redlining / average / power users, surfaces
  loss-making paths, and diffs against prior snapshots stored in
  memory. Optionally writes a COSTS.md to the repo. Also invocable as
  `/cost-analysis`.
metadata:
  author: kevincui1034
  version: "1.3.0"
---

# Cost & Margin Analysis

Inventory every $-generating call in the project, attach a per-unit cost, model the pricing structure, and produce **per-plan / per-user / per-feature** margin math with redlines flagged. Optionally persist the result as a dated memory snapshot and a checked-in `COSTS.md`.

This skill is **stack-agnostic**. It detects the project shape first, then runs only the analyses that apply. If the repo has prior cost snapshots in auto-memory, it diffs against them instead of starting cold.

## Scope & boundaries

**Use this for** recurring, *variable* run-cost and per-unit margins on a software product: what each paid API / SDK / service costs per call, whether each pricing tier is profitable, and where a redlining user breaks the model.

**Not built for:**
- **One-time / fixed costs** — dev time, initial build, design, data migration. Fixed monthly infra (a flat DB tier, a seat you pay regardless of usage) is an *input* to the break-even extra, not a per-unit margin.
- **Formal finance** — this is engineering-grade unit economics, not accounting. It does not do taxes, revenue recognition, amortization schedules, or GAAP. Feed its COGS/margin outputs into real finance tooling; don't treat them as filings.
- **Traffic / capacity forecasting** — it prices the calls you make, not how many you'll make. The growth-curve extra models 10×/100×, but only off an assumed curve you supply.
- **Vendor negotiation / contract discovery** — it reads public + in-repo rates. If you have committed-use or negotiated pricing, supply it; the skill can't discover it.
- **A project with nothing charged** — if there are no paid surfaces and no pricing model, say so in one line and stop; there's no economics to model.

## How to use this skill

Run the **workflow** in order. The discovery phases (1–3) feed the math (4–6); don't skip them or numbers will be fabricated rather than derived. The **catalog of paid surfaces** at the bottom is the reference for what to look for.

---

## Workflow

### Step 1 — Memory recall (do this first, before any file reads)

Look for prior cost / pricing snapshots in auto-memory. Read in this order:

1. The `MEMORY.md` index for entries matching: `pricing-snapshot-*`, `cost-snapshot-*`, `pricing-redlines`, `pricing-sources`, `pricing-roadmap`, or anything tagged `type: project` referencing $ values.
2. Any matched memory files in full.

If snapshots exist, this run becomes a **diff against the most recent one**. State that upfront in the report so the user knows you're not starting from scratch.

**A newer snapshot is not automatically the whole story.** Also read any *dated rate-change / reprice / denomination* memories that post-date the newest full snapshot (naming like `*-reprice-*`, `cost-rate-*`). A small later memory can silently invalidate the snapshot's **units** without invalidating its underlying dollar math — e.g. an in-app currency gets redenominated (say 1 unit = $0.10 → $0.01, with every count ×10 to compensate), leaving the snapshot's *dollar* margins correct but its *unit counts* stale by that factor. Reconcile before quoting any figure, and state which unit you're using.

If no snapshots exist, this is a cold audit — say so, and after the report ask the user whether to persist a baseline snapshot to memory.

### Step 2 — Stack & surface detection (parallel reads)

Read in parallel, then write a one-paragraph stack summary:

- `package.json` — SDKs (`@anthropic-ai/sdk`, `openai`, `@ai-sdk/*`, `stripe`, `resend`, `@vercel/blob`, `replicate`, `@fal-ai/*`, `apify-client`, `@supabase/*`, `@upstash/*`, etc.)
- Lockfile presence (for dep pinning sanity, not for cost)
- `.env.example` / `.env.local` keys — every `*_API_KEY` / `*_TOKEN` / `*_SECRET` is a paid surface candidate
- `CLAUDE.md` / `AGENTS.md` / `PRICING.md` / `COSTS.md` — explicit pricing/cost docs
- `next.config.*` / `vercel.json` / `wrangler.toml` — hosting hints (Vercel, Cloudflare, etc.)
- Glob for likely cost-table files: `**/credits.ts`, `**/pricing.ts`, `**/plans.ts`, `**/quota.ts`, `**/billing.ts`, `**/cost*.ts`

Catalog the paid surfaces detected. For each, record: **provider name**, **what it's used for in this app**, **SDK / endpoint touched**, **billing model** (per-token, per-second, per-request, monthly tier, free-quota-then-paid).

**Classify each surface as deterministic vs. variable — it's the single most decision-relevant attribute:**
- **Deterministic** — cost is fixed by the request *before* you make it: per-request, per-token (by input/output length), per-item, per-second-of-*output*, flat per-tier. These you can price exactly.
- **Variable / uncapped** — cost tracks something you don't control at call time: per-second-of-*runtime* or per-GB-second of compute (a slow, retried, or stuck job bills unboundedly), per-CU/actor-compute, egress/bandwidth, per-MAU. These need a **cap or budget alarm**, not just a per-unit rate — flag any uncapped path in the findings.

Two same-named units can sit on opposite sides of this axis — a provider's "per second" may mean output-seconds on one product and machine-runtime-seconds on another, and the latter can cost many times more. Read the pricing page to see which, and never average a deterministic rate with a variable one.

### Step 3 — Per-unit rate lookup

**Before doing the lookup, ask the user once:**

> "Want me to WebFetch current provider pricing pages and diff against what's in the repo? This catches silent upstream price hikes (Resend, Vercel, Anthropic, etc.) but adds latency and the published rate may differ from your negotiated/legacy rate. Default: no — trust in-repo tables."

**Skip the ask if the user already opted in.** If their invoking prompt already said "search the web", "web-verify the rates", "check current provider pricing", etc., treat that as mode "all" (or the named subset they gave) and go straight to the lookup — re-asking is friction.

Three modes the user can pick:
- **No (default)** — repo + memory only. Note "rate not web-confirmed" next to each surface in the report.
- **Yes, all detected providers** — WebFetch every paid surface from Step 2.
- **Yes, only these:** `<list>` — WebFetch a named subset (e.g., providers the user suspects changed).

Then fill in `$/unit` using this priority:

1. **In-repo rate tables** (e.g., `src/lib/ai/pricing.ts`, `src/lib/credits.ts`). These are canonical for *this app* — they're what the code actually charges against. Trust them over web prices, but flag if they look stale or contradict each other.
2. **Memory snapshots** from Step 1.
3. **Provider docs via WebFetch** — per the user's Step-3 mode above. When web rate ≠ in-repo rate, flag the delta as a finding (the repo might be charging old rates against current provider costs, or vice versa). Don't silently overwrite the repo rate in the analysis — show both and let the user judge which is canonical.
4. **Ask the user** for any rate you can't source from any of the above. Don't invent. A wrong rate makes every downstream number wrong.

**Tag every rate with its provenance and date** — `repo` / `memory` / `web-confirmed <date>` / `estimate` — and carry the tag into the report, so unconfirmed or aging rates are visible at a glance rather than blending in with hard numbers.

For tiered providers (Vercel, Resend, Supabase, Upstash, Sentry, PostHog, etc.), record the **plan the project is on** (ask if unclear) plus the **next-tier breakpoint** — many "cheap" services step-function on row count, bandwidth, or seats. If the user opted into WebFetch, also fetch the current public tier breakpoints (these change more often than per-unit rates).

**Prefer the provider's own live pricing/product page over search-result summaries.** Aggregators, resellers, and search snapshots are frequently stale or plain wrong, and often quote a different SKU than the one you're on. WebFetch the canonical source — the provider's `/pricing` page or the specific model/product page — and trust it over any third-party mirror. If you can only find a third-party number, mark it *estimate* and flag it for confirmation.

**Verify which provider, SKU, *and* version a code path actually routes to before attaching a rate.** The same product name can mean very different money across providers, tiers, and versions — a "mini"/"lite"/"fast" variant, a newer model generation, or the same model on a cheaper host can differ by multiples. Read the call-site for the real model ID / endpoint / SKU rather than matching on the friendly name.

#### Parallelizing the rate lookup (sub-agents)

When there are many providers to web-verify, fan the lookups out to sub-agents — but keep the fan-out **flat and shallow**:

- **One agent per provider-family** (e.g. one for all LLM/token rates, one for all render/media rates, one for infra/hosting). Grouping related pages lets each agent cross-check them.
- **Leaf agents only — forbid nested spawning.** Tell each research agent explicitly: *do the WebFetches yourself and return the table; do NOT spawn your own sub-agents.* A research agent that spawns its own children tends to stall — waiting on descendants it can't observe — and return a status line instead of data, forcing the work to be redone inline. Deep nesting also hides real results behind an intermediary that may summarize or drop them.
- **The agent's final message MUST be the structured findings table** — `Model | Tier/Resolution | Provider | Published price | Unit | Source URL | Date-confirmed` — not a status update. Say this in the prompt.
- **If an agent returns a non-answer, don't ping it in a loop.** Stop it and either re-spawn a fresh leaf agent or just run the handful of WebFetches yourself from the main loop — often faster than babysitting a stuck agent.
- For heavy, repeatable audits a deterministic `Workflow` (fan-out → verify → synthesize) is more reliable than ad-hoc `Agent` calls — but only when the user has opted into orchestration.

### Step 4 — Call-site inventory

Grep every paid surface to its call-sites. For each surface:
- Count distinct call-sites (with `file:line`).
- Identify whether each is **per-user-action** (charges fire when a user does something) or **background / batch** (cron, queue, webhook fan-out).
- Trace gating: is there a quota / credit / rate-limit check upstream? Ungated paid calls are flagged as **unbounded cost risk** even before margin math.

Output a compact inventory table:

| Surface | Provider | $/unit | Call-sites | Gated? | Per-user trigger? |
|---|---|---|---|---|---|

### Step 5 — Pricing model detection

Find how revenue comes in. Look for:
- **Subscriptions**: Stripe products + prices, plan table (`plans.ts` / `usersTable.plan`), monthly/annual amounts.
- **Top-ups / one-time**: Stripe `checkout.session.mode === 'payment'`, in-app credit packs, "buy more X" flows.
- **Usage-based / metered**: per-call billing, metered prices, pay-as-you-go resale of an upstream API.
- **Seat-based (B2B)**: per-seat/per-MAU pricing — revenue scales with users, and so may an upstream per-MAU cost (auth, analytics), so margin can be seat-count-invariant *or* thin depending on which grows faster.
- **Free tier**: what's actually free, monthly grants, daily safety caps.
- **Trials / promo credits**: free credits on signup, referral grants.

Identify the **revenue unit** for each (per-month, per-seat, per-credit, per-API-call). For any **in-app currency** (credits, tokens, points, included-usage units), pin down **three distinct numbers that are easy to conflate**:
1. **Face value** — the accounting unit the app assigns (e.g. "1 credit = $0.01").
2. **COGS per unit** — the provider cost per unit of that currency's worth of output (e.g. if SKUs are priced to a 50% margin floor, ≈ half the face value).
3. **Realized price per unit** — what a user actually *pays* per unit, which varies by acquisition path: subscription grant (plan price ÷ included units), each top-up / à-la-carte pack, promo/free grants ($0).

The floor invariant lives in the gap between (1) and (2); the business margin lives in the gap between (2) and (3). Report **both** gaps — collapsing them into one "margin" number hides which lever is doing the work.

### Step 6 — Unit economics math

For each pricing path, compute three margin scenarios. Be transparent — every number must trace back to a Step 3 rate × Step 4 call count or a Step 5 revenue figure.

**A. Average user** — derive from real usage if available (DB tables like `users`, `*_event`, ledgers); otherwise assume a *named, conservative* profile and state the assumption ("assumed 2 chats / 1 render / week per Pro user").

**B. Redlining user** — the user pattern that maximizes COGS at minimum payment. Find it by inverting: "what's the cheapest plan that exposes the most expensive endpoint?" Examples:
- Free user spamming the only-gated-by-quota endpoint
- Pro user always picking the most expensive model on every turn
- Power user near monthly cap but still under daily safety cap

**C. Power user (paying)** — the heaviest *legitimate* user. Useful to verify caps actually bind before economics break.

Report each scenario as **revenue – COGS = gross margin %**. Include payment processor fees (`gross × 2.9% + $0.30` for Stripe US-card baseline; ask if non-US or non-card).

### Step 7 — Cross-plan & cross-pack comparison

If subscriptions exist with multiple tiers: tabulate $/unit-of-value across tiers (e.g., $/credit, $/render, $/chat) and flag tiers where the higher plan is *worse* value per unit (a common bug).

If top-up packs exist: tabulate $/credit at each pack size and across plans. Flag:
- Non-monotonic pricing (a bigger pack costs more per unit than a smaller one)
- Packs that undercut the subscription's effective $/credit (cannibalization risk)
- Packs that exceed the subscription's $/credit by enough to look exploitative

If usage-based: show $/unit at each pricing tier and where bulk discounts kick in.

### Step 8 — Diff against prior snapshot

If Step 1 surfaced a prior snapshot, produce a **delta section**:
- Rates that changed (provider price hikes, internal repricing)
- New surfaces (new SDKs added since last run)
- Removed surfaces (deprecated)
- Plan changes (new tier, new top-up pack, grant size changes)
- Margin direction (improved / worsened) per scenario

Drift in the wrong direction (worsening margin without a documented reason) is a **finding**, not a footnote.

### Step 9 — Output the report

Output in this structure. Don't pad sections that don't apply — skip them with a one-line "N/A: <reason>".

```
## Cost & Margin Analysis — <project name> (<date>)

**Stack detected**: <one paragraph>
**Pricing model**: <subscription / top-up / usage-based / hybrid>
**Diff basis**: <prior snapshot name, or "cold baseline">

### Paid surfaces inventory
<table from Step 4>

### Unit economics

#### Plan: Free
- Avg user: revenue $X · COGS $Y · margin Z%
- Redlining user: revenue $X · COGS $Y · margin Z%   ← flag if negative

#### Plan: Pro ($29/mo)
- ...

### Cross-plan / cross-pack comparison
<tables from Step 7>

### Findings (severity-ordered)
- **Critical**: <loss-making path that fires today>
- **High**: <margin below stated floor on a real-user pattern>
- **Medium**: <silent cost growth: orphan storage, ungated background job>
- **Low**: <minor inconsistency or hygiene>

### Drift from prior snapshot
<from Step 8, or "N/A — cold baseline">

### Assumptions made
- <each assumption you had to make, with the reason>

### Open questions for the user
- <rates you couldn't confirm, plan tiers unknown, etc.>
```

#### Worked example (illustrative — fictional app, invented numbers)

A compact end-to-end example of the shape the report should take. The app and
every number are made up to show the format — do not treat any rate here as real.

---

## Cost & Margin Analysis — Recapp (2026-07-02)

**Stack detected**: Next.js + Postgres (Supabase) SaaS that summarizes meeting notes via one LLM call per summary. Revenue: Stripe subscriptions (Free / Pro) plus a per-summary "credit" top-up. Transactional email via Resend.
**Pricing model**: subscription + top-up credits (1 credit = 1 summary).
**Diff basis**: cold baseline (no prior snapshot).

### Paid surfaces inventory

| Surface | Provider | $/unit | Unit basis | Rate source | Det/Var | Gated? | Per-user? |
|---|---|---|---|---|---|---|---|
| Summarize | OpenAI gpt-x-mini | $0.0021 | ~3k in + 400 out tok | web-confirmed 2026-07-02 | det | yes (credit) | yes |
| DB/host | Supabase Pro | $25/mo | flat tier | repo | fixed | n/a | no |
| Email | Resend | $0.0004 | per-email | **estimate** | det | no | on signup |
| Payments | Stripe | 2.9% + $0.30 | per-charge | canonical | det | n/a | on purchase |

### Unit economics

Credit stack → face value **$0.02** · COGS/credit **≈ $0.0021** · realized price/credit: Pro grant $20 ÷ 500 = **$0.040**, top-up $5 ÷ 200 = **$0.025**.

#### Plan: Free ($0 · 20 summaries/mo)
- Avg user (6/mo): rev $0 · COGS $0.013 · **−$0.013/mo** (acceptable CAC)
- Redlining user (all 20): COGS $0.042 · **−$0.042/mo** — capped by the grant ✅

#### Plan: Pro ($20/mo · 500 credits)
- Avg user (80/mo): rev $19.12 net of Stripe · COGS $0.168 · **99.1%**
- Redlining (500 + refills): rev $19.12 · COGS $1.05 · **94.5%** ✅

### Cross-plan / cross-pack comparison

| Acquisition path | $/credit |
|---|---|
| Pro subscription grant | $0.040 |
| Top-up pack (200) | $0.025 ← **cheaper than the subscription credit** |

### Findings (severity-ordered)
- **High**: top-up ($0.025/cr) undercuts the subscription credit ($0.040/cr) — heavy Pro users rationally buy top-ups instead of upgrading tiers (cannibalization). Re-price top-up ≥ subscription $/credit.
- **Medium**: Resend rate is an *estimate*, not confirmed against the live plan — verify before trusting the email COGS line.
- **Low**: Supabase $25/mo flat isn't attributed per-user; immaterial below ~10k MAU, revisit at the next tier breakpoint.

### Assumptions made
- Avg Pro user = 80 summaries/mo (no usage data yet — stated, conservative).
- Token mix 3k in / 400 out sampled from one representative prompt.

### Open questions for the user
- Real per-user summary distribution, to replace the 80/mo assumption?
- Is Resend on the free tier or a paid plan?

---

Note how the example carries **provenance tags** (`estimate`, `web-confirmed`) and the **det/var** column straight into the inventory, splits the **three-layer credit stack**, and lands a **cross-pack cannibalization** finding — those are the load-bearing habits, not the specific numbers.

### Step 10 — Persist & offer to write file

This is the two-track persistence the user asked for:

1. **Memory snapshot — always offered, never automatic.** After the report, ask:
   > "Save this as `cost-snapshot-<YYYY-MM>` in auto-memory so the next run can diff against it?"
   If yes: write the snapshot following the [[pricing-snapshot-YYYY-MM]] format (frontmatter with `type: project`, body has the rate tables + margin scenarios + assumptions). Add a `MEMORY.md` index line. Also update or create a `cost-redlines` memory for any Critical/High findings so they survive even if a future snapshot drops them by mistake.

2. **Repo file — offered separately.** Ask:
   > "Write this to `COSTS.md` in the repo so the team can see it?"
   If yes: write to repo root (or `docs/COSTS.md` if a `docs/` convention exists). **Don't overwrite an existing `COSTS.md` without showing the diff first.** Use markdown that renders cleanly on GitHub.

Both prompts default to no — never silently write either.

### Step 11 — Update on mention (the "remember these" requirement)

This skill should re-run partially when the user mentions cost-related changes in *any* future conversation, not just an explicit invocation. Triggers:
- "I'm switching from X to Y" (provider swap)
- "I raised Pro to $X" (subscription change)
- "We added <new SDK>" (new paid surface)
- "Resend bumped their pricing" (external rate change)
- "I'm seeing margin X on Y" (real-world margin data)

When you see one of these and a `cost-snapshot-*` memory exists, **update the snapshot in place** (don't create a new one for every small change — only at explicit audit time). For findings that contradict the snapshot, update `cost-redlines` too.

If the user mentions a change but no snapshot exists, save it as a small standalone memory (e.g., `cost-rate-resend-2026-05.md`) so the *next* full audit can fold it in.

---

## Catalog of paid surfaces

Use this as a checklist during Step 2. For each found, populate the inventory.

### AI / LLM providers
- **Anthropic** (`@anthropic-ai/sdk`) — per million tokens, separate input/output/cache-read/cache-write. Cache-read is ~10% of input. Caching gates: `cache_control` on system or tool blocks.
- **OpenAI** (`openai`) — per million tokens, distinct rates per model. Vision tokens, tool-call tokens, structured-output surcharge.
- **Google Gemini** (`@ai-sdk/google`, raw `fetch` to `generativelanguage.googleapis.com`) — per million tokens, **context cache** (`cachedContents`) has separate read & storage rates; file uploads cost storage too.
- **DeepSeek** (`@ai-sdk/deepseek`) — per million tokens; cache-read discount when prompt prefix is reused.
- **xAI / Groq / Mistral / Cohere** — same shape; flag if SDK present.
- **Voice / TTS / STT**: ElevenLabs, OpenAI Whisper, Deepgram, AssemblyAI — per character or per minute.
- **Embeddings**: per million tokens, usually cheap but easy to over-call (every search bar keystroke can become a $).

### Image / video / audio / asset providers
- **Model hosts** (Replicate, Fal, Modal, RunPod, `replicate` / `@fal-ai/*`) — billing is **per-model, not uniform**: some models are flat **per-output** (per image, or per second of *output*) — deterministic; others are **per second of GPU/machine runtime** — variable and effectively uncapped (see the deterministic-vs-variable note in Step 2). Some are **usage-scaled** (cost = tokens/pixels × rate), where a quoted per-second/per-image figure only holds at the resolution/length it was computed for — a bigger output costs proportionally more, so don't reuse a low-tier rate for a high-tier SKU. Classify each model before pricing it, and never average a deterministic model with a runtime-billed one.
- **Per-generation media APIs** (Runway, Pika, Luma, Sora, Suno, ElevenLabs, etc.) — per-generation or per-second, often **denominated in the provider's own credits**. Convert to USD via the dev/API-portal buy rate, and note that subscription-bundled credits convert at a *different* effective rate than the dev API.
- **Multi-step / "extension" chains** — a pipeline that fans one request into N provider calls multiplies cost; verify the planner caps total spend per request.

### Scraping / data
- **Apify** (`apify-client`) — per-actor compute (CU) + per-result + dataset storage. The per-actor `usageTotalUsd` from a recent run is the best ground truth.
- **ScraperAPI / Bright Data / Zyte** — per-request, often with country/JS-render multipliers.
- **SerpAPI / Brave / Google CSE** — per-query.

### Hosting / infra
- **Vercel** — function compute (`@vercel/functions` invocation × duration × memory), bandwidth, image optimizations, KV/Postgres/Blob row counts. Pro plan starts free but Vercel Functions compute and Blob storage become real $ at scale. Note the team's plan and overage rates.
- **Cloudflare Workers / Pages** — per-request after free, per-CPU-ms on paid plan.
- **AWS / GCP / Fly / Render / Railway** — instance hours + bandwidth + storage.
- **Supabase** (`@supabase/supabase-js`) — DB compute tier, bandwidth, storage, MAUs (auth users). Free tier paused after 7d of inactivity.
- **Neon / PlanetScale / Turso** — compute-time + storage + branch count.
- **Upstash** (Redis / Kafka / QStash, `@upstash/*`) — per-request after free, with regional multipliers.

### Storage / CDN
- **Vercel Blob** (`@vercel/blob`) — per-GB-month storage + per-GB egress + simple ops. **Orphan blobs are a silent cost** — flag if uploads don't have a delete path.
- **S3 / R2 / B2** — same shape; R2 has free egress.
- **Cloudinary / ImageKit / Uploadcare** — per-transformation + storage.

### Email / messaging
- **Resend** (`resend`) — per-email after free tier; bumped pricing recently — confirm against current plan. Inbound forwarding billed separately.
- **Postmark / SendGrid / SES** — per-email; SES is cheapest but has reputation overhead.
- **Twilio / MessageBird** — per-SMS, country-dependent; **the redline that ate startups**.
- **Slack / Discord** — webhooks free, but bot tier may be paid.

### Payments
- **Stripe** — 2.9% + 30¢ on US cards baseline; international cards 3.9% + 30¢; ACH cheaper. Disputes $15. Treat fees as a COGS line, not "outside the system".
- **Lemon Squeezy / Paddle / Polar** — merchant-of-record fees (~5%) bundled.

### Observability / analytics
- **Sentry** — per-event after free; performance & replays priced separately.
- **PostHog** — per-event + per-recording-minute; rate-limit recordings to avoid surprises.
- **Datadog / New Relic / Honeycomb** — per-host or per-event.
- **LogRocket / FullStory** — per-session.

### Auth
- **Auth.js** — free (DB-backed).
- **Clerk / WorkOS / Stytch / Kinde** — per-MAU above free tier. Often the surprise line on B2B.
- **Auth0** — per-MAU with feature gates.

### Misc paid surfaces commonly missed
- **GitHub Actions minutes** (private repos beyond free)
- **Domain renewals + DNS providers**
- **Status page services** (Statuspage, BetterStack)
- **CAPTCHA** (Turnstile is free, hCaptcha free at low volume, reCAPTCHA Enterprise paid)
- **Geo / IP services** (ipinfo, MaxMind)
- **Currency conversion APIs**

---

## Findings rubric

| Severity | Meaning | Examples |
|---|---|---|
| **Critical** | Loss-making path that fires today, or a plan whose redlining-user margin is negative | Free user can trigger an unbounded paid call; a render plan charges less in credits than its provider cost; webhook reprocesses without idempotency so credits are double-granted |
| **High** | Margin below stated floor, or an obvious near-future blow-up | Pro plan margin <40% on average user; orphan blobs accumulate forever; one provider holds >70% of COGS (concentration risk); a plan tier offers worse $/unit-of-value than the tier below it |
| **Medium** | Silent cost growth or unclear pricing | Background job has no concurrency cap; an SDK is imported but never gated by quota; rate not confirmed against canonical source in >90 days |
| **Low** | Hygiene / consistency | Two rate tables disagree on the same model's price; pricing doc out of date with code |

Err one level higher when redlining users can exploit a path — Critical false positives are cheaper than missed losses.

**Billing-model risk is its own axis.** A path whose cost tracks **runtime / wall-clock / bandwidth** (rather than a deterministic per-request or per-output unit) is effectively uncapped — treat it as at least **High** ("uncapped cost path") even if a sample run looks cheap, because a slow, retried, or stuck job bills unboundedly. Confirm real spend against the provider's usage ledger / billing dashboard before concluding it's fine, and check that a cap or budget alarm exists.

---

## Suggested extras (offer these proactively when relevant)

Not part of the default run — surface as **"want me to also …?"** after the main report when the data supports it.

- **Break-even calculator** — given fixed monthly infra ($X), how many of each plan type to break even? Plots conversion required from free → paid.
- **Free-tier cost & conversion required** — $ spent per free user × conversion rate needed to pay them back from a Pro signup.
- **Growth-curve cost trajectory** — model COGS at 10× and 100× current MAU; flag where managed-service tiers step-jump (Supabase compute, Vercel Pro→Enterprise, etc.).
- **Vendor concentration / SPOF** — pie chart of COGS by provider; flag any >50% concentration as both pricing risk and reliability risk.
- **Latent-cost scan** — orphan storage (uploads with no FK), abandoned Stripe customers, dormant Auth users in paid-MAU tiers, expired but unrevoked context caches.
- **Pricing-experiment ideas** — based on the cross-plan comparison, suggest concrete A/B tests (price points, pack sizes, grant amounts) with predicted margin impact.
- **Annual-vs-monthly LTV swing** — given churn rate (ask), show the LTV uplift of annual prepay against the discount cost.
- **Currency / region sensitivity** — if the project has international users, model FX + payment-method-fee variance.
- **Provider-price-change watch list** — record the date each rate was last confirmed; flag any older than 90 days for re-check.
- **Cost-per-feature breakdown** — group COGS by user-facing feature (e.g., "remix chat", "video render", "trend refresh") to identify candidates for feature pricing or sunset.
- **Auto top-up margin model** — if auto top-up exists or is being considered, compute the perverse-incentive risk (does it route around the subscription's bulk discount?).

---

## Don'ts

- **Don't invent rates.** If a rate isn't in the repo, in memory, or confirmed by the user, leave it as `?` and list it as an open question. A wrong rate breaks every downstream number.
- **Don't pull from web for rates without asking.** Step 3 prompts the user once per run for a WebFetch mode (none / all / named subset). Honor that choice — don't WebFetch outside it. When web and repo disagree, surface both and let the user pick canonical; don't silently substitute.
- **Don't recommend price increases without showing the assumed user mix.** "Raise Pro to $39" is meaningless without "assumes 80% avg + 15% power + 5% redlining users."
- **Don't silently write `COSTS.md` or memory snapshots.** Both require explicit user confirmation at Step 10 — this is irreversible team-visible content.
- **Don't conflate revenue with profit.** Stripe fees, refunds, chargebacks, taxes, and amortized fixed infra are real COGS lines.
- **Don't run the full catalog on a side-project with one provider.** Skip categories with a one-line "N/A: no <category> surfaces detected" — pad-by-N/A is noise.
- **Don't quote a snapshot number as current truth** without verifying the underlying file hasn't changed. Snapshots are time-stamped baselines, not live state — see the auto-memory `Before recommending from memory` rule.
- **Don't claim a margin "passes the floor" without naming the floor.** State the invariant ("project's stated floor is ≥40% gross at Max plan") next to the number.
- **Don't bundle the memory write and the file write as one yes/no.** They are two different commitments — one private, one team-visible.
- **Don't spawn deeply-nested research agents.** Rate-lookup sub-agents must be **leaf** agents that WebFetch and return a table — a research agent that spawns its own children tends to stall and return status text instead of data (see Step 3). Fan out flat, one agent per provider-family, and demand the findings table as the final message.
- **Don't trust search-result / aggregator prices over the provider's live page.** Reseller blogs and search snapshots go stale and contradict each other. WebFetch the canonical model/pricing page and treat it as authoritative; mark anything you could only get from an aggregator as an estimate.
- **Don't attach a rate by product name alone.** Verify the provider + model/SKU + version the code actually calls — the same name can be several times different in price across providers, tiers ("mini"/"fast"/"pro"), and versions.
- **Don't reuse a rate quoted at one size/tier for a different one** on usage-scaled providers. Token- or resolution-scaled pricing means a figure quoted at a low tier under-counts a higher tier; re-derive per SKU.
