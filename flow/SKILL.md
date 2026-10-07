---
name: flow
description: Run the multi-agent Workflow orchestration (fan-out, pipeline, parallel, adversarial verify) powered by Codex (GPT-6.1 Sol as the workhorse and default crux verifier, GPT-6 Luna for bulk small calls, GPT-6 Astra for long-context sweeps), Claude, or a mix of both. Use when the user invokes /flow, or asks to "run a workflow"/"fan out"/"orchestrate" and wants to choose fully-codex, fully-claude, or mixed execution. Codex is reached via `codex exec` (the CLI), not the MCP tool. Astra-specific facts live in the `astra` skill.
---

# Flow — Workflow orchestration across Codex + Claude

This skill lets you run the **`Workflow` tool** (the deterministic multi-agent orchestrator: `agent()`, `parallel()`, `pipeline()`, schema validation, progress tree, resume) but choose, per run, whether the agents are **Claude**, **Codex** (GPT-6.1 Sol / GPT-6 Luna / GPT-6 Astra), or a **mix**.

**Invoking this skill is explicit opt-in to the `Workflow` tool.** You are authorized to call `Workflow` for the task at hand — that is the entire point of the skill.

The bridge: the Workflow engine only spawns Claude subagents. A "Codex agent" is therefore a **thin Claude relay** — a subagent whose only job is to write the task to a file, run `codex exec -m <model>` against it (`gpt-6.1-sol`, `gpt-6-luna` or `gpt-6-astra`), read the output file, and return it. Codex does the thinking; Claude drives the CLI. That driver runs on the cheapest capable Claude (**sonnet-5.5 at `effort: 'low'`**) since it does no reasoning of its own. This drops straight into `parallel()`/`pipeline()`, so you keep the full Workflow machinery and just swap the executor per agent.

---

## Step 1 — Determine the mode and the per-stage models

Read the invocation argument. The **mode** sets the executor mix; within any mode you still choose a **model per stage** from the palette below.

| Arg | Mode | Behavior |
|---|---|---|
| `claude` | Fully Claude | Every agent is a plain `agent()` — but set its `model` per stage (sonnet-5.5 / opus-5.5 / fable-5.1). Don't silently default everything to the session model. |
| `codex` | Fully Codex | Every agent uses `codexAgent()` — relays to Codex; pick the **tier** per stage (sol/luna, or astra for the crux/long-context stage — table below). |
| `mixed` | Mixed (default recommendation) | Route each stage to the best model by its nature (routing table below). |

If **no mode** was given, ask the user with one `AskUserQuestion`: *"Run this workflow fully on Codex (GPT-6.1 Sol / GPT-6 Luna / GPT-6 Astra), fully on Claude, or mixed?"* — then proceed.

### The model palette (higher = better on every axis)

Cost is scored *higher = cheaper to me*, not list price — GPT-6.1 Sol and GPT-6 Luna ride the flat Codex subscription (effectively free); gpt-6-astra rides the same subscription but burns a much tighter per-window quota (`7`); fable-5.1's `2` means priciest. Intelligence = how hard a problem you can hand it unsupervised. Taste = UI/UX, code quality, API design, copy.

| model | cost | intelligence | taste | reach it via |
|---|---|---|---|---|
| **gpt-6-luna** | 9 | 7 | 4 | `codexAgent(…, { tier: 'luna' })` — high-volume bounded calls, small inputs only |
| **gpt-6.1-sol** | 9 | 9 | 6 | `codexAgent()` (default tier) — the workhorse, and the default Codex crux verifier, migration and browser tier |
| **sonnet-5.5** | 5 | 8 | 8 | `agent(…, { model: 'sonnet' })` — mechanical and terminal-heavy Claude stages, the relay driver; needs Claude Code ≥ 2.1.293 (see below) |
| **opus-5.5** | 4 | 10 | 9 | `agent(…, { model: 'opus' })` — the Claude default for taste and hard reasoning |
| **fable-5.1** | 2 | 9 | 9 | `agent(…, { model: 'fable' })` — escalation or second Claude vote only |
| **gpt-6-astra** | 7 | 9 | 6 | `codexAgent(…, { tier: 'astra' })` — long-context sweeps (the only measured 1M window), escalation when a 6.1 Sol crux vote or browser run is inconclusive. Never in a wide fan-out. See the `astra` skill. |

(**GPT-6.1 Sol** released 2026-09-29 as `gpt-6.1-sol` and is still OpenAI's newest model as of 2026-10-07. It has a 1.05M API context window, 128K max output and a 2026-04-30 knowledge cutoff. API prices are $2 / $10 per 1M in/out, the same as GPT-6 Sol. OpenAI lists Codex access for Plus, Pro, Business, Enterprise and Edu, but it rolls out per account. This account got it on 2026-09-29 between 20:38 and 21:12 CEST, and it has run in flow since that day. Before that, requests got `400 "The 'gpt-6.1-sol' model is not supported when using Codex with a ChatGPT account"`, and the relay fell back to `gpt-6-sol` by itself (`CODEX_FALLBACK`, verified live on 0.158.0). The Codex catalog as of 2026-10-07 (`client_version` 0.161.0) lists 6.1 Sol as the Codex default model, at efforts `low`–`max` plus `ultra`, with **default effort `low`** and a **373K** Codex working window. Astra, GPT-6 Sol and Luna default to `medium` and have a 272K working window. **No tier lists `none` any more, Luna included.** Luna ran at `none` on codex-cli 0.157.1 on 2026-09-27, but the current catalog drops it. Check the catalog with `grep -c '"slug":"gpt-6.1-sol"' ~/.codex/models_cache.json`; every codex run refreshes that file. `gpt-6-sol` stays in the catalog as 6.1 Sol's fallback, and GPT-5.6 Sol/Terra/Luna are listed as "Older". **`gpt-5.5` retires 2026-10-14**, so don't use it.)

(Claude aliases resolve inside the Claude Code binary, so they change with the Claude Code version. The Agent tool's `model` field takes only the aliases `sonnet`/`opus`/`haiku`/`fable`, never a full model ID, so pin by Claude Code version. Checked 2026-10-07: **Claude Code 2.1.293** maps `opus` → `claude-opus-5-5`, `sonnet` → `claude-sonnet-5-5`, `fable` → `claude-fable-5-1` and `haiku` → `claude-haiku-5-5`. **2.1.280 and older still map `sonnet` to the legacy `claude-sonnet-5`**, so run flow on ≥ 2.1.293. To check the installed version: `grep -aoE '(sonnet|haiku|opus|fable):"claude-[a-z0-9-]+' ~/.local/share/mise/installs/claude/latest/claude | sort -u`. API prices per 1M tokens in/out: Opus 5.5 $4 / $20 (released 2026-09-22), Sonnet 5.5 $2 / $10 (2026-09-28), Fable 5.1 $10 / $50, and Haiku 5.5 $0.10 / $0.50 up to a 100K-token prompt and $0.50 / $2.50 above (2026-10-07). All four have a 1M context window and 128K max output. Sonnet 5 and Haiku 4.5 are legacy.)

**Sol and Luna share the flat subscription, so choosing between them is about quota headroom and task shape, not price.** Astra is the exception: it shares the subscription but has its own tight cap (Plus 5–45 messages per 5 hours, Pro 25–900 depending on plan; higher effort burns more), so treat Astra calls as a scarce resource. Between Sol and Luna it's about three things:
1. **Intelligence needed** — 6.1 Sol is the workhorse and now close to Astra. OpenAI reports that it nearly matches Astra on DeepSWE v1.1, AutomationBench and Terminal-Bench Science. On OSWorld 2.0 it lands within 2.1 points of Astra at about 1/7 the cost per task. On Terminal-Bench 4.0, OpenAI's launch chart puts it at about 55–67% across efforts for $0.4–1.4 per task, against Astra's 50–57% for $4–9 and Opus 5.5's 56–59% for $5–13. On its own harness Anthropic reports 66.4% for Opus 5.5. These are vendor launch-day numbers; no independent runs exist yet. GPT-6 Sol, the fallback, reached about 90–95% of Astra, with a factual-error rate of 4.5% at `max` against Astra's 4.2%. 6.1 Sol's error rate is unpublished. Luna roughly matches the previous-generation flagship on coding (DeepSWE 66.6% at `max` vs GPT-6 Sol's 68.8%) but falls off on open-ended, tool-heavy work (FrontierCode 42.4% vs GPT-6 Sol's 49.3%).
2. **Quota headroom** — OpenAI estimates 15–160 local 6.1 Sol messages per 5-hour window on Plus (GPT-6 Sol was 15–150), against 350–3,000 Luna messages. These are third-party trackers quoting OpenAI's help center, which blocks scrapers. There is no GPT-6 Terra: wide fan-outs that need real context go to Sol at `medium`, and high-volume small calls go to Luna.
3. **Context robustness** — 6.1 Sol has the same 1.05M window as Astra, but OpenAI has published no long-context retrieval (MRCR) numbers for 6.1 Sol, GPT-6 Sol or Luna. Luna's predecessor, GPT-5.6 Luna, had a confirmed cliff (~41% vs GPT-5.6 Sol's ~91% on 256–512K retrieval). **Until GPT-6 Luna is measured, keep the same rule: never hand Luna a whole diff, a repo-wide sweep, or any large context; keep Luna inputs well under ~100K tokens.** When a single head genuinely needs >150K tokens of context, that stays an **astra** stage (MRCR 8-needle 100% at 256–512K, 96% at 512K–1M) until 6.1 Sol's retrieval is measured. Keep a single Astra or Sol input under ~200K. Codex's working window is 272K for Astra (373K for 6.1 Sol), and above 272K input OpenAI bills the whole request 2× on input and 1.5× on output.

How to apply:

- **Defaults, not limits.** You have standing permission to escalate: if a cheaper model's output doesn't meet the bar, rerun that stage on a smarter model without asking (bump the agent's `model`/`tier` and `resumeFromRunId`). Judge the output, not the price tag — escalating costs less than shipping mediocre work.
- **Cost is a tie-breaker only.** When axes conflict for anything that ships: **intelligence > taste > cost**.
- **Bulk / mechanical** (clear-spec implementation, migrations, systematic test writing, wide review fan-outs over a diff) → **sol, effort `medium`**. **High-volume bounded calls** (per-finding verify votes, extraction, classification on small inputs) → **luna, effort `low`** (the lowest effort any tier lists now that `none` is gone). A tiny fully-specified edit (one or two files, small context) can go to luna at `high`.
- **Hardest well-specified logic, long implementation slices, data analysis** → **sol, effort `high`**.
- **Anything user-facing** (UI, copy, API design) needs **taste 9** → opus-5.5. **Never any GPT tier here, Astra included**: Astra and GPT-6 Sol trail Opus 5.5 on FrontierCode (53.3% and 49.3% vs 54.4%), 6.1 Sol's FrontierCode is unpublished, and the taste bar stays Claude.
- **Code review** → by default two independent reviewers, **opus-5.5 and 6.1 Sol at `high`**, with Claude reconciling. Lui rates 6.1 Sol very good at code review (2026-09-29), so it is a full reviewer here, not an optional extra.
- **Reviews / adversarial verify / critic** → opus-5.5, with a **Codex verifier as an independent perspective** (cross-provider second vote). Use **sol `xhigh`** (6.1 Sol) for the crux finding: it is close to Astra on agentic coding at a fifth of the list price and on the looser quota. Use **astra `high`** when the crux needs more than 150K tokens of context in one head, when the 6.1 Sol vote is inconclusive, or as a third vote on money, auth or SQL. Use luna `low` for routine votes. Claude reconciles: both agree → confirmed; only one raises it → targeted verification before reporting. Give the Codex verifier the deployment context Claude holds (DB backend, prod vs local, known conventions) in the task text; it is precise but blind to what it cannot read from the repo.
- **Long-context sweeps** (whole subsystem, 2,000-line diff, a dozen PR diffs in one call) → **astra `high`/`xhigh`**, the only tier with measured 1M-window retrieval. One head, not a fan-out. Revisit once 6.1 Sol's MRCR is published or measured here.
- **Migration / schema / CLI scripting with a complete spec** → **sol `high`** (6.1 Sol is at or above Astra on OpenAI's terminal benchmarks); astra `medium`–`high` to escalate when a sol run gets the ops wrong.
- **Browser / desktop automation stages** → **sol** with the playwright MCP. 6.1 Sol is within 2.1 points of Astra on OSWorld 2.0; GPT-6 Sol was 12 points behind on OSWorld (60.5% vs 72.6%). Escalate to astra when a run fails or stalls.
- **opus-5.5 vs fable-5.1:** default every Claude stage that needs taste or intelligence to **opus-5.5**, including the crux stage. Opus 5.5 beats Fable 5.1 on every head-to-head Anthropic published (Terminal-Bench 4.0 66.4% vs 55.8%, FrontierCode 54.4% vs 50.3%, HLE 67.7% vs 65.6%, GDPval-AA 1846 vs 1735) at 2.5× lower price, and it writes more plainly. Anthropic warns that margins at this level are a weak guide to real-world differences, so keep **fable-5.1** for escalation when an opus-5.5 stage misses the bar, or as an independent second Claude opinion on a crux.
- **sonnet-5.5 vs opus-5.5:** Sonnet 5.5 (2026-09-28) closes most of the gap at half the price and runs 30%+ faster than Sonnet 5. Anthropic's numbers: Terminal-Bench 4.0 70.6% (Opus 5.5 66.4%), FrontierCode 1.1 52.1% at `xhigh` (54.4%), GDPval-AA 1844 (1846) and OSWorld 2.1 80.1% (81.8%). Anthropic still rates Opus 5.5 clearly stronger on complex, open-ended work that needs sustained judgment. So mechanical and terminal-heavy Claude stages go to **sonnet-5.5**, while crux, design, review and synthesis stay on **opus-5.5**. These are vendor launch numbers; no independent runs exist yet.
- **Set `effort` on Claude stages too.** An omitted `effort` inherits the session's. Use `xhigh` for the crux / verify stage, `high` for design and review, and `low` for mechanical Claude stages.
- **The Codex relay driver** is pure CLI plumbing → pinned to **sonnet-5.5 / `effort: 'low'`** inside `codexAgent`. Don't spend a taste-tier model on it.
- **Never use Haiku.**

### Routing table for `mixed`

| Stage nature | Model (executor) |
|---|---|
| High-volume bounded fan-out: per-finding verify votes, extraction, classification (small inputs only) | **luna** — `codexAgent({ tier:'luna', effort:'low' })` |
| Pattern-following backend, mechanical refactor, migrations, systematic tests, wide review fan-outs | **sol** at medium — `codexAgent({ effort:'medium' })` |
| Complex well-specified logic, long implementation slices, data analysis | **sol** — `codexAgent()` |
| Code review of a diff or PR | **opus-5.5** `high` **and** **sol** `high` (`codexAgent({ effort:'high' })`) as independent reviewers; Claude reconciles; astra when diff plus context exceeds ~150K tokens |
| Technical review / adversarial verify / critic | **opus-5.5** — `agent({ model:'opus', effort:'xhigh' })` (fable-5.1 as a second Claude vote on a crux); add a Codex verifier (**sol `xhigh`** for the crux, astra for a long-context crux or a third vote; luna for routine) as an independent cross-provider vote; Claude reconciles |
| Long-context sweep needing >150K tokens in one head (whole subsystem, huge diff, batch of PR diffs) | **astra** — `codexAgent({ tier:'astra', effort:'high' })`, single stage, sequential if more than one |
| Migration / schema / CLI scripting from a complete spec | **sol** `high`; **astra** `medium`–`high` to escalate |
| Browser / desktop automation | **sol** — `codexAgent()` with the playwright MCP available; **astra** to escalate |
| Brainstorming, design, architecture, API-surface decisions | **opus-5.5** — taste axis; fable-5.1 to escalate |
| Synthesis into a narrative, user-facing copy, final go/no-go | **opus-5.5** — taste axis; fable-5.1 to escalate |

---

## Step 2 — Paste the Codex relay header into your Workflow script

Put this block near the top of every script (after `meta`). It is the only Codex-specific machinery you need. **Claude agents stay as plain `agent()` calls** — only Codex agents go through `codexAgent()`.

```js
// ===== Codex relay (GPT-6.1 Sol / GPT-6 Luna / GPT-6 Astra via `codex exec`, codex-cli >= 0.156.1) =====
// Sol and Luna are flat-rate on the subscription; pick by task shape and quota headroom, not price:
//   sol   — GPT-6.1 Sol, workhorse (default): hardest logic, implementation slices, crux verify at 'xhigh',
//           migrations/CLI, browser; wide review fan-outs at 'medium'
//   luna  — fast: high-volume bounded calls ONLY; long-context behaviour unmeasured, keep inputs small
//   astra — >150K-token sweeps (only measured 1M retrieval); escalation for an inconclusive crux or browser run.
//           Separate tight quota: single stage or sequential for..of, NEVER parallel().
//           Keep one input < ~200K tokens.
// Efforts: low/medium/high/xhigh/max on all tiers. No tier lists 'none' any more; codexAgent maps it to 'low'.
// 'ultra' is Codex's own subagent delegation; never use it here, since flow is the orchestrator.
// No GPT-6 Terra exists. gpt-5.5 retires 2026-10-14.
const CODEX_TIERS = { astra: 'gpt-6-astra', sol: 'gpt-6.1-sol', luna: 'gpt-6-luna' }
// One automatic retry on the next model when Codex answers "model is not supported" (CLI too old)
// or "model is at capacity". Usage-limit errors are NOT retried. The log gets a FLOW_FALLBACK line.
const CODEX_FALLBACK = { 'gpt-6.1-sol': 'gpt-6-sol', 'gpt-6-astra': 'gpt-6.1-sol', 'gpt-6-luna': 'gpt-5.6-luna' }

// OpenAI structured outputs REQUIRE `additionalProperties: false` on every object
// node of the --output-schema; without it codex exec dies with invalid_json_schema
// before doing any work. Strictify recursively so authors never think about it.
function strictifySchema(node) {
  if (Array.isArray(node)) return node.map(strictifySchema)
  if (!node || typeof node !== 'object') return node
  const out = {}
  for (const [k, v] of Object.entries(node)) out[k] = strictifySchema(v)
  if (out.type === 'object') {
    out.additionalProperties = false
    if (out.properties) out.required = Object.keys(out.properties)
  }
  return out
}

// Codex runs DETACHED (setsid) so it survives the Bash tool's 10-minute cap; the relay then
// waits in ~9-minute chunks (tail --pid, no `sleep`), up to maxChunks (default 16 = ~2.4 h).
// Stall watchdog: two consecutive chunks with no growth in the --json event log => kill + report.
// Files per slot: <base>.md task, -out.md answer, -log.jsonl events, .pid, .exit.
function codexRelayPrompt(task, { model, fallback, sandbox = 'read-only', effort = 'high', slot, schemaJson, maxChunks = 16, chunkSeconds = 540, extraFlags = '' } = {}) {
  const base = `.codex-run/wf-${slot}`
  const schemaFlag = schemaJson ? ` --output-schema "${base}-schema.json"` : ''
  const run = (m) => `printf "" | codex exec --json --skip-git-repo-check -m ${m} -c model_reasoning_effort="${effort}" -s ${sandbox}${extraFlags}${schemaFlag} -o "${base}-out.md" "Read the file ${base}.md and follow its instructions exactly. Do not create git commits."`
  // The fallback runs inside the detached shell, so the relay driver needs no judgment and schema calls get it too.
  const retry = fallback
    ? ` if [ $RC -ne 0 ] && grep -qE "model is not supported|model is at capacity" "${base}-log.jsonl"; then echo "FLOW_FALLBACK ${model} -> ${fallback}" >> "${base}-log.jsonl"; ${run(fallback)} >> "${base}-log.jsonl" 2>&1; RC=$?; fi;`
    : ''
  const codexCmd = `${run(model)} > "${base}-log.jsonl" 2>&1; RC=$?;${retry} echo $RC > "${base}.exit"`
  return [
    `You are a RELAY to Codex (OpenAI ${model}). Do NOT do the task yourself — Codex does it.`,
    'Follow these steps EXACTLY, in order:',
    `1. Use the Write tool to create "${base}.md" containing the TASK below verbatim (ASCII only — transliterate any non-ASCII).`,
    schemaJson
      ? `2. Use the Write tool to create "${base}-schema.json" with EXACTLY this content:\n${schemaJson}`
      : '2. (no schema file needed)',
    '3. LAUNCH. Use the Bash tool (timeout 120000) to run EXACTLY this command. It starts Codex detached and returns within seconds:',
    `   mkdir -p .codex-run; rm -f "${base}-out.md" "${base}.pid" "${base}.exit" "${base}-log.jsonl"; setsid bash -c 'echo $$ > "${base}.pid"; ${codexCmd}' < /dev/null > /dev/null 2>&1 & for i in 1 2 3 4 5 6 7 8 9 10; do [ -s "${base}.pid" ] && break; timeout 1 tail -f /dev/null; done; echo "LAUNCHED pid=$(cat "${base}.pid" 2>/dev/null)"`,
    '   If the output is "LAUNCHED pid=" with no number, return that line verbatim prefixed with "CODEX_RELAY_FAILED LAUNCH" and stop.',
    `4. WAIT. Use the Bash tool (timeout 600000) to run EXACTLY this command. Each run waits at most ${chunkSeconds} seconds:`,
    `   P=$(cat "${base}.pid"); S0=$(stat -c %s "${base}-log.jsonl" 2>/dev/null || echo 0); timeout ${chunkSeconds} tail --pid="$P" -f /dev/null; RC=$?; S1=$(stat -c %s "${base}-log.jsonl" 2>/dev/null || echo 0); if [ -s "${base}.exit" ]; then echo "DONE exit=$(cat "${base}.exit")"; elif [ $RC -eq 124 ] && [ "$S1" = "$S0" ]; then echo STALLED; else echo RUNNING; fi`,
    `   - "RUNNING": run step 4 again (new Bash call). Count the runs; after ${maxChunks} runs without DONE, go to step 6 with reason TIMEOUT.`,
    '   - "STALLED" twice in a row: go to step 6 with reason STALLED. A single STALLED counts as RUNNING.',
    '   - "DONE exit=0": go to step 5. "DONE exit=<non-zero>": go to step 6 with reason CODEX_ERROR.',
    `5. Use the Read tool to read "${base}-out.md". If it is missing or empty, go to step 6 with reason NO_OUTPUT.`,
    schemaJson
      ? '   It contains JSON. Call StructuredOutput with the parsed JSON object (extract it if Codex wrapped it in prose). Stop.'
      : '   Return the FULL contents of that file as your answer, unmodified. Stop.',
    `6. FAIL. Use the Bash tool to run: kill -TERM -- -$(cat "${base}.pid") 2>/dev/null; grep -o '"thread_id":"[^"]*"' "${base}-log.jsonl" | head -1; tail -n 40 "${base}-log.jsonl"`,
    '   Then return exactly: "CODEX_RELAY_FAILED <reason>" followed by the thread_id line (for `codex exec resume <id>`) and the log tail verbatim.',
    'Hard rules for YOU the relay: never fabricate output; never run codex in the foreground; never edit repo files yourself; never commit.',
    '',
    '=== TASK ===',
    task,
  ].join('\n')
}

// Drop-in replacement for agent() that runs the task on Codex (GPT-6.1 Sol / GPT-6 Luna / GPT-6 Astra).
// REQUIRED: a unique `slot` (or unique `label`) per call so parallel codex runs
// never clobber each other's .codex-run files. Use the map index, e.g. slot: `review-${i}`.
async function codexAgent(task, opts = {}) {
  const { tier = 'sol', sandbox = 'read-only', effort = 'high', schema, label = 'codex', phase, maxChunks = 16, chunkSeconds = 540, extraFlags = '' } = opts
  const slot = opts.slot || label
  const model = CODEX_TIERS[tier]
  if (!model) throw new Error(`codexAgent: unknown tier '${tier}' (use ${Object.keys(CODEX_TIERS).join('/')}; terra has no GPT-6 successor, use sol at effort 'medium')`)
  return agent(
    codexRelayPrompt(task, {
      model,
      fallback: CODEX_FALLBACK[model],
      sandbox,
      effort: effort === 'none' ? 'low' : effort,
      slot,
      maxChunks,
      chunkSeconds,
      extraFlags,
      schemaJson: schema ? JSON.stringify(strictifySchema(schema), null, 2) : undefined,
    }),
    // The wrapper is pure CLI plumbing (write file -> launch detached codex -> wait in chunks -> read file),
    // so it runs on the cheapest capable Claude at low effort. NOTE: `effort` above is
    // CODEX's reasoning effort (the -c model_reasoning_effort flag); the low effort here
    // is the relay DRIVER's own, and the two are independent.
    { label, phase, schema, model: 'sonnet', effort: 'low' },
  )
}
// ===== end Codex relay =====
```

### `codexAgent(task, opts)` options

- **`tier`** — `'sol'` (default; `gpt-6.1-sol`, the workhorse, falls back to `gpt-6-sol`), `'luna'` (`gpt-6-luna`; fast, bounded small-input calls only, long-context unmeasured), `'astra'` (`gpt-6-astra`; long-context sweeps and escalation; one stage at a time, own quota; falls back to `gpt-6.1-sol` on capacity). An unknown tier throws; `'terra'` is gone. For an explicit older model, add a key to `CODEX_TIERS` in that script (e.g. `sol6: 'gpt-6-sol'`).
- **`slot`** (or unique `label`) — REQUIRED unique id per call. Parallel agents sharing a slot will overwrite each other's files. Use the loop index.
- **`sandbox`** — `read-only` (review/analysis/exploration — the default, always parallel-safe), `workspace-write` (implementation), `danger-full-access` (avoid).
- **`extraFlags`** — raw `codex exec` flags appended after `-s`, with a leading space. The common one is `' -c sandbox_workspace_write.network_access=true'`, for a `workspace-write` stage whose tests reach a local database or service.
- **`effort`** — **Codex's** reasoning effort: `low` (luna lane / yes-no extraction; `codexAgent` also sends `low` for `none`, which no tier lists any more), `medium` (Codex's default for Astra, GPT-6 Sol and Luna, while 6.1 Sol defaults to `low`; wide fan-outs, mechanical implementation), `high` (this helper's default), `xhigh` (crux verification where a wrong answer is expensive), `max` (single crux stage only; slow, never in a wide fan-out; on Astra only when explicitly asked). **Never `ultra`**: it makes Codex spawn its own subagents, which bypasses flow's fan-out control and burns quota. Separate from the relay driver, which is pinned to sonnet-5.5 / `effort: 'low'` since it only drives the CLI.
- **`schema`** — pass a JSON Schema to get a validated object back; Codex produces it natively via `--output-schema`. **Prefer schemas** — they also guard against the brevity-truncation tendency seen on GPT-5.6 (see guardrails).
- **`label`, `phase`** — same as `agent()` (progress grouping).
- **`maxChunks`** / **`chunkSeconds`** — how many waits of `chunkSeconds` (default `540`, must stay under 600) and how many ~9-minute waits the relay allows before giving up (default `16` ≈ 2.4 h). Codex runs detached, so a call is no longer capped by the Bash tool's 10-minute limit; raise this only for genuinely long Astra sweeps. A failed call returns `CODEX_RELAY_FAILED <reason>` (LAUNCH / TIMEOUT / STALLED / CODEX_ERROR / NO_OUTPUT) plus the Codex `thread_id` for `codex exec resume <id>` — check for that prefix before using a result. A call that succeeded on the fallback model looks like any other success; `grep -l FLOW_FALLBACK .codex-run/wf-*-log.jsonl` lists which runs fell back.

---

## Step 3 — Author and run the workflow

Build the script for the task, then call the `Workflow` tool. Use Claude `agent()` and `codexAgent()` according to the mode. Default to `pipeline()` over `parallel()` (see the `Workflow` tool docs) — a barrier is only justified when a stage genuinely needs all prior results at once.

### Example A — Codex review fan-out (mode: `codex`, read-only)

```js
export const meta = {
  name: 'codex-review',
  description: 'Fan out a code review across dimensions on Codex GPT-6.1 Sol, then verify each finding',
  phases: [{ title: 'Review' }, { title: 'Verify' }],
}
// <paste Codex relay header here>

const FINDINGS = { type: 'object', properties: { findings: { type: 'array', items: {
  type: 'object', properties: { title:{type:'string'}, file:{type:'string'}, detail:{type:'string'} },
  required: ['title','file','detail'] } } }, required: ['findings'] }
const VERDICT = { type:'object', properties:{ real:{type:'boolean'}, why:{type:'string'} }, required:['real','why'] }

const DIMENSIONS = [
  { key:'bugs',     prompt:'Review the current git diff for correctness bugs and logic errors.' },
  { key:'security', prompt:'Review the current git diff for auth/injection/secret-handling issues.' },
  { key:'perf',     prompt:'Review the current git diff for performance regressions and N+1 patterns.' },
]

// Reviews read the whole diff -> sol at high (6.1 Sol is a strong reviewer; never luna: long context unmeasured).
// Verify votes are bounded single-finding checks -> luna, effort low.
// For a crux finding (money, SQL, auth) add ONE sol `xhigh` vote after the fan-out, e.g.
//   const crux = await codexAgent(`...try to REFUTE...`, { tier:'sol', effort:'xhigh', slot:'crux-sol', schema:VERDICT })
// Crux spans >150K tokens, or the sol vote is inconclusive -> tier:'astra', effort:'high', outside parallel().
const results = await pipeline(
  DIMENSIONS,
  (d, _orig, i) => codexAgent(d.prompt, { tier:'sol', slot:`review-${d.key}`, phase:'Review', schema:FINDINGS, effort:'high' }),
  (review, d) => parallel((review?.findings||[]).map((f, j) => () =>
    codexAgent(`Adversarially verify this finding — try to REFUTE it. Default real=false if uncertain.\nTitle: ${f.title}\nFile: ${f.file}\nDetail: ${f.detail}`,
      { tier:'luna', effort:'low', slot:`verify-${d.key}-${j}`, phase:'Verify', schema:VERDICT })
      .then(v => ({ ...f, verdict:v })))),
)
const confirmed = results.flat().filter(Boolean).filter(f => f.verdict?.real)
return { confirmed }
```

### Example B — Mixed build pipeline (mode: `mixed`)

```js
export const meta = {
  name: 'mixed-build',
  description: 'Design on Claude, implement backend slices on Codex, synthesize on Claude',
  phases: [{ title: 'Design' }, { title: 'Implement' }, { title: 'Synthesize' }],
}
// <paste Codex relay header here>

// Design / architecture — KEEP ON CLAUDE (taste axis) -> opus-5.5 (fable-5.1 only if opus misses the bar)
const plan = await agent(
  'Read the spec at docs/spec.md and produce a JSON list of independent backend slices to implement, each with {title, files, instructions}.',
  { phase:'Design', model:'opus', schema: { type:'object', properties:{ slices:{ type:'array', items:{ type:'object',
    properties:{ title:{type:'string'}, files:{type:'array',items:{type:'string'}}, instructions:{type:'string'} },
    required:['title','files','instructions'] } } }, required:['slices'] } })

// Implementation — SEND TO CODEX (workspace-write). Slices touch DISJOINT files (see guardrails).
// sol at medium for pattern-following slices; bump the hardest slice to effort:'high'.
const built = await parallel(plan.slices.map((s, i) => () =>
  codexAgent(`Implement this slice. Follow existing patterns. Run the project's lint+typecheck+tests.\n\nTitle: ${s.title}\nFiles: ${s.files.join(', ')}\n${s.instructions}`,
    { tier:'sol', slot:`build-${i}`, sandbox:'workspace-write', effort:'medium', phase:'Implement' })))

// Synthesis — KEEP ON CLAUDE (taste axis) -> opus-5.5
const summary = await agent(
  `Summarize what was implemented across these slices for the operator, flag anything risky:\n${built.filter(Boolean).join('\n---\n')}`,
  { phase:'Synthesize', model:'opus' })
return { summary }
```

For `claude` mode: identical structure, but every `codexAgent(...)` becomes a plain `agent(...)` — drop the `tier`/`slot`/`sandbox` opts and instead set `model` per stage from the palette (mechanical → sonnet-5.5; review / design / synthesis → opus-5.5, fable-5.1 to escalate). Keep `effort`; on `agent()` it is Claude's own reasoning effort. GPT is off the table in `claude` mode.

---

## Guardrails (always apply)

- **Codex never commits.** The relay command forbids it once, and the sandbox flag is the hard enforcement. After the workflow returns, *you* (the operator) review the changes and handle the commit. The workflow itself must not commit.
- **State each prohibition ONCE in Codex task text.** GPT-5.6 over-triggered permission-checking on repeated "do NOT / ask first" language; assume GPT-6 does too until shown otherwise. In non-interactive `codex exec` that means stalls or under-acting, not safety. Write one compact constraints line per task; let the sandbox flag (`-s read-only` / `workspace-write`) do the actual enforcement.
- **Never ask Codex to "be concise."** GPT-5.6 could silently truncate deliverables (drop findings, skip sections), and GPT-6 defaults to low verbosity. Ask for prioritized completeness instead ("always include X, Y, Z; trim preamble"), or better, force structure with `schema`.
- **Luna context cap.** Never route a whole diff, multi-file review, or repo-wide sweep to luna. GPT-6 Luna's long-context retrieval is unpublished and its 5.6 predecessor had a confirmed cliff. Luna gets bounded, single-item tasks with inputs well under ~100K tokens. When in doubt, sol at `medium`.
- **"Model is not supported when using Codex with a ChatGPT account"** means either the CLI is too old (GPT-6 Luna needs ≥ 0.156.1) or the model isn't in this account's catalog yet (`~/.codex/models_cache.json`; GPT-6.1 Sol as of 2026-09-29). The relay then runs the fallback model; `FLOW_FALLBACK` in a log shows which runs it affected. Check the catalog before upgrading, and don't downgrade the tier.
- **Never `mise upgrade codex` while a relay is running.** It deletes the old install, and a running `codex exec` then fails every command and patch with `CreateProcess … No such file or directory`. On 2026-09-29 that cut short an hour-long slice-3 build. Upgrade side by side instead: `mise install codex@<version> --minimum-release-age 0d`. mise hides releases younger than 24 h by default, and the flag overrides that for an explicit version. `latest` moves to the new version while running jobs keep their binary. Run `mise prune codex` only when no job is live: `for f in .codex-run/wf-*.pid; do [ -f "${f%.pid}.exit" ] || echo "live: $f"; done` in every repo with a running workflow.
- **Check the weekly limit before any long Codex build.** `grep -o '"rate_limits":{.\{0,300\}' "$(ls -t $(find ~/.codex/sessions -name '*.jsonl' -mmin -120) | head -1)" | tail -1` shows `used_percent`, `window_minutes` (10080 = weekly) and `resets_at` (epoch seconds). An hour-long workspace-write build on Sol `high` used about 7% of the weekly window on 2026-09-29. Usage is not a routing constraint for Lui, who has a second Codex account to switch to. Don't ration or pre-warn; when a relay fails on the usage limit, tell him right away so he can switch, then resume.
- **Usage exhausted mid-run.** Stop the fan-out and report; don't loop retries. Selected Plus/Pro accounts get Luna Reserve (the hidden `gpt-reserve` slug, GPT-5.6 Luna on a separate meter) as Codex's automatic fallback. Never put it in `CODEX_TIERS`.
- **After a session restart, look for live Codex jobs before anything else.** A workflow that stopped with the session can leave `codex exec` running, and on 2026-09-30 a wave-2 job even started after the round had been committed. Either way it keeps editing the working tree, uncoordinated. Run `ps -eo pid,etime,args | grep "codex exec" | grep -v grep` and check each job's cwd (`readlink /proc/<pid>/cwd`). A job in this repo that you did not just launch: stop it (`kill -TERM -- -<pgid>`), then review `git status` and its diff before keeping or discarding anything. Never resume a workflow over a slot whose Codex job is still alive: the relay's `rm -f` and relaunch would clobber it. Wait for its `.exit` instead, then continue from its `-out.md`.
- **Detached Codex jobs outlive a cancelled workflow.** The relay launches `codex exec` with `setsid` so it survives the Bash tool's 10-minute cap; if you stop a workflow mid-run, kill the leftovers yourself: `for f in .codex-run/wf-*.pid; do [ -f "${f%.pid}.exit" ] || kill -TERM -- -"$(cat "$f")" 2>/dev/null; done`. Results are in `.codex-run/wf-<slot>-out.md`, the event log in `-log.jsonl`.
- **Astra is a scarce single-head resource.** Never put `tier:'astra'` inside `parallel()` or a `pipeline()` fan-out stage — parallel Astra calls hit the 5-hour burst cap and the whole stage fails. One Astra stage per workflow is the norm; if you truly need several, run them sequentially with `for (const x of items) await codexAgent(..., { tier:'astra', slot:`astra-${i}` })`. Bulk mechanical work stays on sol/luna.
- **Astra runs read-only unless it must patch.** OpenAI's own system card reports a small rate of malicious contributions under scope restrictions and a chain-of-thought that is hard to monitor. `-s workspace-write` only when the stage is meant to produce a patch, and that diff is reviewed by Claude (opus-5.5) before staging. Trust Astra's cited evidence (quoted lines, traced callers), never its summary of its own reasoning.
- **Astra sandbox has no `php`.** Anything that needs PHP lint or tests to run goes to a Claude agent; Astra can read and trace PHP but not execute it.
- **Never forward an Astra finding unverified, never drop one just because Claude missed it.** Its findings are precise and well-cited but blind to deployment context (e.g. flagged MySQL-only syntax on a Postgres production). Put that context in the task text, and let the Claude reconcile stage do a targeted check on any finding only one provider raised.
- **Sandbox by intent.** Review/analysis/exploration → `read-only` (default; always parallel-safe). Implementation → `workspace-write`. Avoid `danger-full-access`.
- **Parallel write-agents must touch disjoint files.** Two `workspace-write` codex agents editing the same file in parallel will conflict. Options: (a) give each slice non-overlapping files (preferred), (b) run write-stages sequentially, or (c) use the `Workflow` tool's `isolation: 'worktree'` if they must overlap.
- **Unique slot per agent.** Always derive `slot` from the loop index (`build-${i}`). Shared slots overwrite `.codex-run/wf-*.md` files mid-run.
- **ASCII-only prompt files.** Non-ASCII in `.codex-run/*.md` causes mojibake. The relay prompt already instructs transliteration; keep your task text ASCII too.
- **Codex defaults to GPT-6.1 Sol. Tier down to Luna by task shape, and up to Astra only for a long-context stage or an escalation.** Tune `tier` + `effort`, not the model family. "Not supported" and "at capacity" errors fall back automatically (`CODEX_FALLBACK`). If a tier misbehaves in any other way, swap its `CODEX_TIERS` value by hand (`gpt-6-sol`, or the `gpt-5.6-*` predecessors). If Astra errors or its quota is exhausted, rerun that stage on `sol` at `xhigh`. `gpt-5.5` retires 2026-10-14 and `gpt-5.4` is banned. On the Claude side, pick per stage from the palette (sonnet-5.5 / opus-5.5 / fable-5.1) by task nature, on Claude Code ≥ 2.1.293. Never Haiku.
- **Escalate freely.** Model choices are defaults, not limits: if a cheap stage's output is weak, bump that agent's `tier`/`model` to a smarter one and `resumeFromRunId` (the unchanged prefix replays from cache; only the re-modeled stage and after re-run). Intelligence > taste > cost.
- **Cost is real.** Workflows fan out — a 3×3 review is 12 sessions. Scale the fan-out to the request: a few finders for "quick check," a larger pool + multi-vote verify for "thoroughly audit this." On the Codex side "cost" = subscription quota: small-input fan-outs go to luna, context-heavy fan-outs to sol at `medium`, and never to astra.

## Why `codex exec`, not the MCP tool

For fan-out the user prefers `codex exec` over `mcp__codex-cli__codex`: separate processes (true parallelism, no MCP serialization concern), no inline-prompt truncation (task is read from a file), and native `--output-schema`. The `mcp__codex-cli__codex` tool remains the right choice for a *single* interactive dispatch outside a workflow (see the `/codex` skill) — but inside `/flow`, always use the relay.

## When NOT to use this skill

- A single Codex task → use the `/codex` skill (one MCP dispatch), not a workflow. A single Astra review or sweep → the `astra` skill's bash relay, not a workflow.
- A trivial edit or quick read → just do it inline.
- The user hasn't asked for orchestration and the task doesn't need it → don't spin up a fleet of agents.
