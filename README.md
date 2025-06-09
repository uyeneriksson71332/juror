<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/hero-dark.svg">
  <source media="(prefers-color-scheme: light)" srcset="assets/hero-light.svg">
  <img alt="Juror — frontier models review your pull request in parallel" src="assets/hero-light.svg">
</picture>

<br>

[![CI](https://img.shields.io/github/actions/workflow/status/juror-ai/juror/ci.yml?branch=main&style=flat-square&label=ci&labelColor=1b1f24&color=3fb950)](https://github.com/juror-ai/juror/actions/workflows/ci.yml) [![Juror reviews Juror](https://img.shields.io/badge/dogfooded-juror%20reviews%20juror-F2B33D?style=flat-square&labelColor=1b1f24)](.github/workflows/juror.yml) [![Node](https://img.shields.io/badge/node-%E2%89%A520-8593a8?style=flat-square&labelColor=1b1f24)](package.json) [![License](https://img.shields.io/badge/license-MIT-8593a8?style=flat-square&labelColor=1b1f24)](LICENSE)

</div>

**Juror is a GitHub Action that asks several frontier models to review each pull request in
parallel. It combines duplicate findings into one report and posts the review, merge score, and
cost receipt back to GitHub.**

## Juror Cloud

This repository also contains [`cloud/`](cloud/), a deployable hosted companion with GitHub and Google sign-in, GitHub App automation, a lean findings and run inbox, isolated Cloudflare Sandbox execution, post-merge staging QA, and transparent Stripe metered billing. The open-source Action and CLI remain independently usable, while repositories connected to Cloud use hosted execution exclusively.

See the [Juror Cloud deployment guide](cloud/README.md) for the GitHub App permissions, Cloudflare resources, secrets, billing meter, and verification commands.

## ChatGPT, Codex, and MCP

Juror Cloud is also available as a remote, OAuth-protected Streamable HTTP MCP server at `https://app.juror.dev/mcp`. Add that URL in an MCP-compatible client, sign in with your Juror Cloud account, list your workspaces first, and then inspect concise review summaries.

The integration is intentionally narrower than the GitHub Action and local CLI:

- **Juror Cloud MCP / Plugin** — inspects workspace-scoped findings and hosted runs. It can prepare and, after explicit user confirmation, start or rerun an existing hosted Juror Cloud review. It never accesses arbitrary local repositories or returns raw diffs, full reports, artifacts, screenshots, credentials, or prompt text.
- **Juror Cloud** — the hosted service that owns workspace membership, GitHub App access, isolated execution, retention, and billing.
- **GitHub Action and CLI** — run from your repository or local environment using the configuration you control. Do not run the hosted service alongside a Juror GitHub Action for the same repository.

Starter prompts:

- “Show my Juror workspaces and review findings.”
- “Summarize open P0 and P1 Juror findings in this workspace.”
- “Prepare a hosted review for this open PR; show me the SHA and cost warning before starting.”

The canonical registry metadata is [`registry/server.json`](registry/server.json). See the [directory submission record](docs/mcp-directory-submissions.md) for publication status and the required external verification steps.

## Add Juror to your repository

Start from the [Juror AI listing in GitHub Marketplace](https://github.com/marketplace/actions/juror-ai):
click **Use latest version** to add it to a workflow. If you prefer to add it directly, create
`.github/workflows/juror.yml` with this ready-to-run workflow:

```yaml
name: Juror

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write

concurrency:
  group: juror-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  review:
    # Provider secrets are never exposed to code from forks.
    if: github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262 # v4.4.0
        with:
          fetch-depth: 0
          filter: blob:none
      - uses: juror-ai/juror@178a16d6fa3a1b868094bc4236d42fb8e442ea6a # v1.3.3
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
        env:
          JUROR_OPENAI_API_KEY: ${{ secrets.JUROR_OPENAI_API_KEY }}
          JUROR_FIREWORKS_API_KEY: ${{ secrets.JUROR_FIREWORKS_API_KEY }}
```

Then add at least one provider key in **Settings → Secrets and variables → Actions**. One key
starts a review; adding a second provider makes the default `fast` preset a cross-model jury:

```bash
gh secret set JUROR_OPENAI_API_KEY
gh secret set JUROR_FIREWORKS_API_KEY
```

Commit the workflow and open a pull request. Juror immediately posts a sticky *Juror is
reviewing…* comment, then replaces it with the findings, merge score, and bill. Every Action
input besides `github-token` is optional; see [`action.yml`](action.yml) for `preset`, `models`,
`config`, `cost-target-usd`, `post`, and `pr-number`.

The example pins both Actions to immutable commits. The Marketplace's generated snippet is a
convenient starting point; use an immutable SHA in the committed workflow to keep the reviewed
Action version fixed.

<details><summary><b>Prefer a guided CLI setup or a local review?</b></summary><br>

The CLI can generate the same SHA-pinned workflow, inspect the plan first, and optionally upload
provider keys through the GitHub CLI:

```bash
export JUROR_OPENAI_API_KEY=…
npx juror-ai init --set-secrets

# Or inspect the planned workflow and secrets without changing anything.
npx juror-ai init --dry-run
```

To review an existing pull request locally without creating a workflow:

```bash
npx juror-ai review --pr 1234 --repo owner/name          # prints to your terminal
npx juror-ai review --pr 1234 --repo owner/name --post   # also posts the review
```

The optional `starter` preset uses one `JUROR_OPENROUTER_API_KEY` to reach two model families:

```bash
npx juror-ai init --preset starter --set-secrets
```

</details>

---

> ### Benchmark evidence: Juror Fast found 4× more adjudicated P0–P2 defects than Greptile
>
> On the bundled production-PR seed, Juror Fast found **4/6 (66.7%)** P0–P2 defects at
> **100% precision**. Greptile found **1/6 (16.7%)** at **50% precision**.
>
> | Reviewer | P0–P2 recall | Precision |
> |---|---:|---:|
> | **Juror Fast** | **66.7% (4/6)** | **100% (4/4)** |
> | Greptile | 16.7% (1/6) | 50% (1/2) |
>
> This is a manually adjudicated, one-PR seed—not a statistically sufficient replacement
> benchmark. Inspect the [corpus](benchmarks/platform-10359.json) and
> [methodology](docs/benchmarking.md), or reproduce it with
> `juror benchmark --file benchmarks/platform-10359.json`.

### Optional post-merge browser QA

Juror can also watch merged pull requests, plan affected user journeys, and exercise the live
staging or branch deployment through Playwright with structured evidence and, for runs without
authentication or supplied browser state, videos. Authenticated runs still interact with the
product and evaluate plan-bound checkpoints, but each admitted page-dependent browser call returns the
same sealed acknowledgement and completed-attempt outcomes stay hidden from the model:

```bash
npx juror-ai init --qa --set-secrets --target-url https://staging.example.com
```

For a product that exposes a fixed-identity synthetic support-session endpoint, Juror can mint a
fresh, single-use login URL for every attempt through `qa.auth.session_bootstrap`. This v1 mode is
restricted to the configured canonical staging origin: set `qa.target.preview_fallback: false`,
and do not expect the session to follow a branch deployment with a different origin. Optional
`qa.auth.browser_secret_headers` are injected only into requests to their exact configured origins,
which supports a staging Cloudflare Access service token and a separate WAF header without exposing
either credential to other hosts. All logical secrets remain inside `JUROR_QA_SECRETS_B64`;
authenticated visual and trace evidence stays off and browser observations stay sealed.

Use the fixed synthetic-session service, not an arbitrary-user testing-login bypass. The
[post-merge QA quickstart](docs/2026-08-18-post-merge-qa-quickstart.md#staging-support-session-bootstrap)
includes a generic staging example and the required gateway prerequisite.

Repositories can also configure trusted `qa.testability.early_exit_paths` globs for paths that
never justify browser QA. Juror returns a neutral, not-scored `no_testable_surface` result before
deployment lookup, secret loading, model startup, or Playwright only when every path in the
complete changed-file manifest matches. The default is empty because an infrastructure or docs
tree can still affect a product in some repositories.

The target's exact origin is added to the browser allowlist automatically; repeat `--allow-origin`
for any additional API origin the tested product needs. Without either target option, init writes a
safely disabled QA policy and explains how to enable it. The released action verifies its container
provenance before credential handoff. See the
[post-merge QA quickstart](docs/2026-08-18-post-merge-qa-quickstart.md) for the fastest local loop,
staging target setup, synthetic login credentials, and result meanings.

By default, a policy without a trusted reset hook runs navigation-only journeys. A change whose
affected journey requires interaction is reported as blocked, not as having no browser surface.
Reviewed policy can set `qa.sandbox.interaction_policy: read_only` to enable click, fill, press,
select, and check while Juror blocks non-safe HTTP methods and outbound WebSocket messages after the
first action. Persistent mutations still require a dedicated synthetic tenant and
`qa.sandbox.reset`.
Each checkpoint fixes its assertion kind and exact locator or URL matcher before the browser opens;
authenticated or supplied-state scenarios always run a second sealed attempt, and the controller
derives the result from its private ledger. The final report necessarily reveals one bounded
pass/fail result per predeclared checkpoint, so connect QA only to a synthetic account and tenant
containing no production data or other sensitive records.

---

## Why

Single-model PR bots have three problems, in order of how much they cost you:

1. **Blind spots.** Every model misses different bugs.
2. **Duplication.** Multiple reviewers often describe the same defect in different words.
3. **Opacity.** You pay per seat or per PR and never see what the inference actually cost.

Juror runs several models, uses code-aware similarity plus a conservative referee to
deduplicate reports about the same defect, and defaults to high recall: every unique
eligible finding is shown. Teams that
prefer fewer, higher-confidence findings can switch `review.publish_mode` to `consensus`
and use model agreement as a precision filter.

And there is no index and no SaaS. A coding agent doesn't need a prebuilt semantic index:
the supported agent harnesses all ship repository read/search tools and will go inspect the
callers of the function you changed. You get repo-wide context for the price of a few tool
calls, with zero indexing infrastructure, zero staleness, and no code leaving the runner
beyond the model API call itself.

**Non-goals.** Not an autofix bot. Not a linter (yours is better and free). Not a chat
interface. It reviews a diff and posts findings.

---

## How it works

<div align="center">
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/pipeline-dark.svg">
  <source media="(prefers-color-scheme: light)" srcset="assets/pipeline-light.svg">
  <img alt="The diff fans out to a jury of models; their findings are anchored, blocked, merged and refereed into one comment" src="assets/pipeline-light.svg">
</picture>
</div>

Each model gets the diff, its own private scratch directory, and read-only access to a clean
detached checkout. Their findings then go through five lossless merge stages — cheapest first,
with a model call only for possible semantic duplicates:

1. **Anchor** *(free)* — snap every finding to a line the diff actually adds or modifies.
   Findings landing outside the diff are reported separately, never silently dropped.
2. **Block** *(free)* — group by file, then by overlapping line window.
3. **Exact collapse** *(free)* — normalized identical reports, or identical structured
   trigger/mechanism/consequence/fix claims, collapse without inference.
4. **Similarity + referee** *(cheap)* — weighted prose/symbol similarity nominates possible
   duplicates. A small call per block merges them only when the faulty mechanism and fix
   match and their affected behavior substantially overlaps; extra entry points or effects
   in one report do not make the same bug new. A malformed partition is retried once, then
   fails open to separate findings so deduplication can never hide a report.
5. **Coverage audit** *(free)* — prove every raw atomic finding belongs to exactly one final
   published or explicitly suppressed result. Any accounting failure discards the merge
   decisions and falls back to lossless singletons.

In `consensus` mode an additional **verify** stage runs: eligible P0/P1 and eligible
single-model findings get an adversarial refutation pass. The verifier is asked to
*refute*, and defaults to refuted when the evidence isn't clear.

---

## What it posts

One sticky summary comment, and inline comments delivered as **a single batched review** —
one notification, not twelve. Roughly:

> #### Juror Review
>
> Adds SSE `event: error` detection to the reasoning stream so mid-stream provider failures
> retry instead of ending the turn as a silent success.
>
> #### Merge Confidence: 4/5
>
> <sub>Model votes: GPT-5.6 Terra `4` · Grok 4.5 `5` · Kimi K3 `4` → median **4**, capped at **4.5** by 1 confirmed P2.</sub>
>
> | | Severity | Location | Finding | Agreement |
> |---|---|---|---|---|
> | 1 | P1 | `src/stream/parse.ts:212` | Error branch leaves the reader unlocked | `●●●` 3/3 |
> | 2 | P2 | `src/stream/parse.ts:424` | Same swallow pattern not ported to the sibling class | `●○○` 1/3 |
>
> <details><summary>2 findings suppressed — below severity floor</summary><br>
>
> | Location | Finding | Raised by | Why suppressed |
> |---|---|---|---|
> | `src/stream/parse.ts:387` | `chunks_emitted` hardcoded on error events | GPT-5.6 Terra, Kimi K3 | below severity floor |
>
> </details>
>
> <details><summary><b>💸 This review cost $0.91</b> · 3 models · 2m14s</summary><br>
>
> | Model | Harness | Input | Cached | Output | Cost | Source |
> |---|---|---|---|---|---|---|
> | `GPT-5.6 Terra` | Codex | 39.8k | 12.1k | 8.9k | $0.34 | estimated |
> | `Grok 4.5` | Grok Build | 40.1k | 0 | 5.2k | $0.38 | reported |
> | `Kimi K3` | Kimi Code | 42.0k | 10.0k | 4.2k | $0.19 | estimated |
> | `referee (1 call)` | opencode | — | — | — | $0.0011 | reported |
> | **Total** | | **122k** | **22.1k** | **18.3k** | **$0.91** | |
>
> </details>

Plus a file-by-file overview and an optional sequence diagram of the changed flow.

With `--post`, Juror immediately creates one sticky **Juror is reviewing…** comment with an
animated working indicator and a short progress checklist. The finished summary replaces
that same comment in place; failed runs replace it with a terminal error state instead of
leaving a spinner behind forever.

---

## Install

### GitHub Action

Install from the [Juror AI GitHub Marketplace listing](https://github.com/marketplace/actions/juror-ai)
or copy the [workflow above](#add-juror-to-your-repository). `npx juror-ai init` is an optional
guided way to generate that workflow and set secrets; it is not required to use the Action.
Beyond `github-token`, every Action input is optional: `preset`, `models`, `config`,
`cost-target-usd`, `post` (set `false` for a dry run), and `pr-number`. They are documented with
their defaults in [`action.yml`](action.yml).

### Locally

The same binary, the same code path, no CI-only surprises:

```bash
npm i -g juror-ai

juror review --base main                         # review your working branch
juror review --pr 1234 --repo owner/name         # review a PR, print to the terminal
juror review --pr 1234 --repo owner/name --post  # ...and post it
```

Put your keys in a `.env` beside the repo (it is loaded automatically and never committed).
Juror copies only committed/staged/tracked working changes into a detached model checkout,
so this untracked file is not inside any reviewer read root:

```
JUROR_ANTHROPIC_API_KEY=…
JUROR_OPENAI_API_KEY=…
JUROR_FIREWORKS_API_KEY=…
JUROR_XAI_API_KEY=…
JUROR_OPENROUTER_API_KEY=…
JUROR_SCALEWAY_API_KEY=…
```

---

## Supported models & harnesses

Juror normally drives each model through its **native agent harness**, so each one greps your
repo the way its vendor intended. The opt-in one-secret starter uses Juror's confined in-process
tool loop against OpenRouter instead; it preserves the same sealed checkout and exact findings
file boundary without installing another executable.

| Harness | CLI | Models | Reports cost | Sandbox |
|---|---|---|---|---|
| `claude-code` | `claude -p` | any Anthropic model | ✅ `total_cost_usd` | tool removal |
| `codex` | `codex exec` | any OpenAI model | ❌ → estimated | split filesystem profile (kernel) |
| `deepseek` | `codewhale exec` | DeepSeek models | ❌ → estimated from provider usage | read-only sandbox + tool allowlist |
| `opencode` | `opencode run` | anything on [models.dev](https://models.dev) — Fireworks, Groq, OpenRouter, … | ✅ per-step `cost` | tool removal |
| `grok-build` | `grok -p` | Grok models | ✅ `total_cost_usd` | Landlock |
| `kimi-code` | `kimi -p` | Kimi K3 on Fireworks | ❌ → estimated from session usage | tool allowlist + isolated runtime |
| `generic-openai` | *(in-process)* | any OpenAI-compatible endpoint | provider-reported when documented; otherwise estimated | path-confined tools |

DeepSeek models use the DeepSeek-native CodeWhale harness so interleaved reasoning survives
tool calls. To add **DeepSeek V4.1 Flash** on Fireworks to your jury:

```yaml
models:
  - id: deepseek-v4.1-flash
    harness: deepseek
    harness_model: accounts/fireworks/models/deepseek-v4p1-flash
    pricing_key: accounts/fireworks/models/deepseek-v4p1-flash
    secret: JUROR_FIREWORKS_API_KEY
    args: { reasoning_effort: high }
```

Scaleway's European OpenAI-compatible endpoint is also built in as
`scaleway-deepseek-v4-flash-0731`. It uses `JUROR_SCALEWAY_API_KEY`, the
`deepseek-v4-flash-0731` model id, and `reasoning_effort: high`; select `ultra` or
add this route to an explicit jury:

```yaml
models:
  - id: scaleway-deepseek-v4-flash-0731
    harness: generic-openai
    secret: JUROR_SCALEWAY_API_KEY
    base_url: https://api.scaleway.ai/v1
    harness_model: deepseek-v4-flash-0731
    pricing_key: scaleway/deepseek-v4-flash-0731
    args: { reasoning_effort: high }
```

For a project-scoped or dedicated Scaleway deployment, replace `base_url` with its
public endpoint URL; do not put credentials in that URL.

---

## Configuration

Juror ships five jury presets. Models whose provider key is unavailable are skipped, so
`ultra` means every built-in model that can actually authenticate on that runner.

| Preset | Jury | Intended use |
|---|---|---|
| `starter` *(opt-in)* | GPT-6 Luna · DeepSeek V4.1 Flash through OpenRouter's confined generic harness | Two model families from one `JUROR_OPENROUTER_API_KEY`; awaiting benchmark promotion gate |
| `fast` **(default)** | GPT-6 Luna via Codex/OpenAI (`low`) · DeepSeek V4.1 Flash via DeepSeek/Fireworks (`high`) | Lean two-model jury |
| `balanced` | GPT-5.6 Terra via Codex/OpenAI (`max`) · Grok 4.5 via Grok Build/xAI (`high`) · Kimi K3 via Kimi Code/Fireworks (`max`) | Strong provider diversity without the full burn |
| `high` | GPT-5.6 Sol via Codex/OpenAI (`high`) · Opus 5 via Claude Code/Anthropic · Grok 4.5 via Grok Build/xAI (`high`) | Higher