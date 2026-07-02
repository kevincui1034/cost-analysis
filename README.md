# Cost & Margin Analysis

> ### 💸 Are you making money on every plan — or quietly subsidizing your heaviest users?
> One command reads your codebase, inventories every paid API, and
> managed service, pulls **live** provider rates off the web, and hands you your
> true margin **per credit, per plan** — with the loss-making paths
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

- **Inventories paid surfaces** — LLM/AI APIs, media & asset generation,
  scrapers & data APIs, hosting & compute, databases, storage/CDN, email &
  messaging, payments, observability, and auth — every SDK, endpoint, or managed
  service that costs money.
- **Sources per-unit rates** from in-repo rate tables → memory snapshots →
  (optionally) provider pricing pages via WebFetch, asking before it fetches.
- **Models the pricing structure** — subscriptions, seat-based, top-up packs,
  usage-based/metered, free tiers, trial/promo credits.
- **Computes three margin scenarios** per plan — average user, redlining user
  (max COGS at min payment), and heaviest legitimate power user — as
  `revenue − COGS = gross margin %`, including payment-processor fees.
- **Compares across plans and packs** and flags non-monotonic or
  cannibalizing pricing.
- **Diffs against prior snapshots** and surfaces margin drift as a finding.
- Optionally persists a dated memory snapshot and/or a checked-in `COSTS.md`
  (both opt-in, never automatic).

> **Scope note:** this skill models *recurring, variable* run-cost and margins.
> It does not compute one-time project setup/build costs (dev time, initial
> provisioning). Fixed monthly infra can be fed into the optional break-even
> calculator as an input.

## What's new

### 1.3.0 — worked example + scope

- **Worked example** — a filled, end-to-end sample report (fictional app,
  invented numbers) so the expected output shape is concrete, not just a
  skeleton.
- **Scope & boundaries** — an explicit "not built for" section (one-time/build
  costs, formal finance/tax, traffic forecasting, contract-rate discovery) to
  keep the skill from over-reaching.

### 1.2.0 — generalized

The 1.1.0 lessons were battle-tested on a media-render audit, so their wording
leaned on image/video specifics. 1.2.0 lifts them into **provider-agnostic
principles** — they apply just as well to LLM tokens, infra compute, scrapers,
and metered APIs — and adds a few general improvements:

- **Billing-model taxonomy** — a general *deterministic vs. variable/uncapped*
  classification you run on every surface, not just GPU models. Runtime-,
  bandwidth-, and compute-metered paths get flagged for a cap or budget alarm.
- **Rate-provenance tagging** — every rate carries `repo` / `memory` /
  `web-confirmed <date>` / `estimate` into the report, so unconfirmed or aging
  numbers don't blend in with hard ones.
- **Seat-based (B2B) pricing** added to the pricing-model detection alongside
  subscriptions, top-ups, usage-based, and free tiers.
- Reworded the sub-agent, live-page, version-verification, denomination, and
  unit-stack guidance to drop render-specific examples for neutral ones.

### 1.1.0 — hardened

Folded in from a real multi-provider audit:

- **Parallel rate lookups, done right** — one flat, leaf-only sub-agent per
  provider-family, each returning a structured price table; sidesteps the
  stall-and-return-status failure mode.
- **Live pages over aggregators** — prefers the provider's own pricing page;
  treats reseller/search-snapshot numbers as estimates to be flagged.
- **Version-and-provider verification** — the same product name can be several
  times different in price across providers, tiers, and versions.
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
pins `version: 1.3.0`, you'll receive changes when that version is bumped.

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
