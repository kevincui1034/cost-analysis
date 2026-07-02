# Cost & Margin Analysis

> ### 💸 Are you making money on every plan — or quietly subsidizing your heaviest users?
> One command reads your codebase, inventories every paid API, render, and
> managed service, pulls **live** provider rates off the web, and hands you your
> true margin **per credit, per render, per plan** — with the loss-making paths
> flagged in red. No spreadsheet. No guessing. No "we'll figure out unit
> economics later."

A [Claude Code](https://code.claude.com/docs) skill that audits a project's
**unit economics**. It inventories every paid surface (API calls, SDKs,
scrapers, render providers, managed services), attaches per-unit rates from
canonical sources, models the pricing structure, and produces
**per-plan / per-user / per-feature** margin math with redlines flagged.

It's **stack-agnostic** — it detects the project shape first, then runs only
the analyses that apply. If the repo has prior cost snapshots in auto-memory,
it diffs against them instead of starting cold.

## What it does

- **Inventories paid surfaces** — Anthropic, OpenAI, Gemini, Replicate, Fal,
  Apify, Stripe, Resend, Vercel, Supabase, Upstash, Sentry, PostHog, and more.
- **Sources per-unit rates** from in-repo rate tables → memory snapshots →
  (optionally) provider pricing pages via WebFetch, asking before it fetches.
- **Models the pricing structure** — subscriptions, top-up packs, usage-based,
  free tiers, trial/promo credits.
- **Computes three margin scenarios** per plan — average user, redlining user
  (max COGS at min payment), and heaviest legitimate power user — as
  `revenue − COGS = gross margin %`, including Stripe fees.
- **Compares across plans and packs** and flags non-monotonic or
  cannibalizing pricing.
- **Diffs against prior snapshots** and surfaces margin drift as a finding.
- Optionally persists a dated memory snapshot and/or a checked-in `COSTS.md`
  (both opt-in, never automatic).

> **Scope note:** this skill models *recurring, variable* run-cost and margins.
> It does not compute one-time project setup/build costs (dev time, initial
> provisioning). Fixed monthly infra can be fed into the optional break-even
> calculator as an input.

## What's new in 1.1.0

Hardened from a real multi-provider audit (image + video render pricing across
Replicate and Fal, plus Gemini / Anthropic / DeepSeek / Deepgram):

- **Parallel rate lookups, done right** — fan web-verification out to one
  sub-agent per provider-family, kept flat and leaf-only (no nested spawning),
  each returning a structured price table. Sidesteps the stall-and-return-status
  failure mode.
- **Billing-model classification** — distinguishes deterministic *per-output*
  pricing from *per-GPU-second* run-time metering, and flags the latter as an
  uncapped-cost risk (the two can differ 15–30× within one model family).
- **Live pages over aggregators** — prefers the provider's own model/pricing
  page; treats reseller and search-snapshot numbers as estimates to be flagged.
- **Version-and-provider verification** — the same product name (e.g.
  "Seedance") can be 4× different in price across providers and versions, so the
  skill checks what the code actually routes to before attaching a rate.
- **Reprice / denomination reconciliation** — reads dated rate-change memories
  that post-date the newest snapshot so redenominated units aren't quoted stale.
- **Three-layer unit stack** — separates face value, COGS-per-unit, and the
  realized price a user actually pays per unit, and reports each margin gap.

## Install

This repo is both a Claude Code **plugin** and a single-plugin **marketplace**.

### Option A — Plugin (recommended)

In Claude Code, run:

```
/plugin marketplace add kevincui1034/cost-analysis
/plugin install cost-analysis@cost-analysis
```

Update later with `/plugin marketplace update cost-analysis`. Because the plugin
pins `version: 1.1.0`, you'll receive changes when that version is bumped.

### Option B — Drop the skill in manually

If you'd rather not use the plugin system, copy just the skill folder into your
skills directory:

**Personal (all projects):**

```bash
# macOS / Linux
git clone https://github.com/kevincui1034/cost-analysis /tmp/cost-analysis
cp -r /tmp/cost-analysis/skills/cost-analysis ~/.claude/skills/cost-analysis
```

```powershell
# Windows (PowerShell)
git clone https://github.com/kevincui1034/cost-analysis $env:TEMP\cost-analysis
Copy-Item -Recurse "$env:TEMP\cost-analysis\skills\cost-analysis" "$env:USERPROFILE\.claude\skills\cost-analysis"
```

**Project-scoped** (checked into a repo, shared with collaborators): copy the
same `skills/cost-analysis` folder into your project's `.claude/skills/`.

## Usage

Once installed, invoke it explicitly:

```
/cost-analysis
```

…or just ask in natural language — Claude triggers it on prompts like
"how much does this cost to run?", "what's our margin?", "are we losing money
on the free plan?", "compare the top-up packs", or "is this priced right?".

## Repo layout

```
cost-analysis/
├── .claude-plugin/
│   ├── plugin.json         # plugin manifest
│   └── marketplace.json    # marketplace catalog (one plugin, source "./")
├── skills/
│   └── cost-analysis/
│       └── SKILL.md         # the skill itself
├── LICENSE
└── README.md
```

## License

[MIT](LICENSE) © Kevin Cui
