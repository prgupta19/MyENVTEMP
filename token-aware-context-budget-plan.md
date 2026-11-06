# Token-aware context budget for Advise Workbench

Audience: engineers who will implement this. Repo: `Advise-Workbench/` (branch `dev-master`). All paths below are relative to it.

## 1. Context and goal
`TieredContextEngine.assemble_v2` (`src/memory/retrieval.py`) builds `assembled_context` for runs. Its size is capped by
`memory_compaction_char_cap = 32000` **characters** (`src/core/config.py`), split by fixed character shares across sections.
That is the wrong unit: models limit and bill in tokens, and the app routes to several models with different windows and tokenizers.

Goal: replace the character cap with a **token budget counted per model** and divided across **system prompt, memory, retrieved chunks and history**; then make the budget coherent with the rest of the app and tune quality, cost and latency.

Decisions already made by the product owner:
- Default budget stays **32,000 tokens** (about 3.5x the old ~9K-token footprint; see risk R1).
- Unit tests run **offline**: `memory_token_count_exact=False` in `tests/conftest.py`.
- "History" means the run change trail (`WhatChanged`), not chat. Chat/conversation-digest limits are out of scope.

## 2. Status
**Nothing in this plan is committed.** A throwaway prototype of Workstream A exists as *uncommitted* changes in the working tree
(11 modified + 2 new files: `src/llm/context_budget.py`, `tests/unit/test_context_budget.py`). It was never run against the live API and
has the three defects listed in section 3 (items 3-4). Use it as a reference or discard it (`git checkout -- . && git clean -fd src/llm/context_budget.py tests/unit/test_context_budget.py`); do not merge it as is.

## 3. Verified findings that drive the design
1. **Per-call prompt limits are dead config.** `subagent_max_prompt_tokens=6000`, `coordinator_max_prompt_tokens=12000`, `proposal_max_prompt_tokens=8000`, `token_estimate_ratio=3.5`, `token_budget_safety_margin_tokens=500` (`config.py:301-309`) are read nowhere in `src`. `TokenBudgetChecker` hardcodes `CHARS_PER_TOKEN=3.5` and `SAFETY_MARGIN_TOKENS=100` (`src/llm/token_budgets.py:28,31`). The only enforcement is `src/llm/claude.py:112-145`, which compares against the *per-run* budget; with `ANTHROPIC_MAX_TOKENS_PER_RUN=0` (as in `.env`) nothing is ever compressed.
2. **Real downstream limits are character slices.** Specialists call `_grounded_context_excerpt(ctx, N)` (`src/agents/specialists/_shared.py:896`) with N = 3000-6000 chars (~850-1,700 tokens): xlsx 6000, narrative/docx 4000 (docx also 3000), pptx/pdf 3500. Its docstring admits a head slice misses the ranked chunks. Metrics already exist: `subagent_context_offered_chars_<agent>_total`, `..._consumed_chars_...`, `subagent_context_truncated_total`.
3. **Model resolution must go through the gateway.** Real routing is `gateway.model_for(tier)` (`src/llm/gateway.py:60`; tiers `default`/`planning`/`critique`), and the provider may be Anthropic, Bedrock, OpenAI-compat or Claude CLI. Anthropic's `count_tokens` only applies to the Anthropic provider. Reuse `AnthropicProvider.client` (`src/llm/providers/anthropic_api.py:50`) so `base_url` and timeouts are honoured; do not create a private `anthropic.Anthropic`.
4. **Tiering changes tokenization.** `model_tiering_enabled=True`: planning `claude-opus-4-8`, critique `claude-sonnet-4-6`, drafting `claude-haiku-4-5` (`config.py:341-343`). Per Claude API notes the Opus 4.7+ tokenizer yields roughly 1x-1.35x the tokens of older ones, so a count on the drafting model can under-count for the planning model.
5. **Prompt caching** exists only for the system prompt (`claude.py:93`). Per-agent excerpts differ and are ~1K tokens, so caching them would not pay; a shared 12K prefix reused 6x costs about $0.07 uncached vs ~$0.006 for today's excerpts (Haiku 4.5 at $1/MTok input).
6. **Cost reporting** uses `llm_price_input_per_mtok=3.00`/`output=15.00` (`config.py:296-299`); no `LLM_PRICE*` in any `.env*`, so Haiku 4.5 ($1/$5) is over-reported ~3x.
7. **Old cap rarely bound.** 4 stored snapshots (`workspace/*/runs/*/compaction_snapshot.json`) used 0-22 chars of 3.7K-11.2K-char budgets; near-empty dev runs, not representative.

## 4. Workstreams

### Step 0: baseline (before any code change)
Record the unit-test failure set on clean `dev-master`: `pytest tests/unit --continue-on-collection-errors -rf`, save the FAILED/ERROR names.
Known environmental failures: `test_run_graph.py` (Redis refused at `127.0.0.1:1`), `test_conversation_summary_worker.py` (imports `_generate_summary_text`, which is only a comment at `conversation_summary_worker.py:409`). A full run on the prototype showed 97 failures + 1 collection error; they are **unattributed** until compared with this baseline. Acceptance for every workstream: no new failures vs baseline.

### Workstream A: replace the character cap (core)
**A1. New module `src/llm/context_budget.py`**
- `count_tokens(text, model) -> (tokens, exact)`: if `text` empty -> `(0, True)`. Use exact counting only when `memory_token_count_exact` is true, the active provider is Anthropic, `model` is non-empty and the offline backoff is not active; call `provider.client.messages.count_tokens(model=..., messages=[{"role":"user","content":text}]).input_tokens`; cache by `(model, sha256(text))` (bounded, e.g. 2048 entries). On any exception: log at info, start a 5-minute offline backoff, return `(TokenBudgetChecker.estimate_tokens(text), False)`. Counting must never fail a run.
- `context_window(model) -> int`: `client.models.retrieve(model).max_input_tokens`, cached per model; on failure or non-Anthropic provider return `DEFAULT_CONTEXT_WINDOW = 200_000` (smallest window among configured models, so the fallback never overflows).
- `MemoryBudgetPlan` (frozen dataclass): `model, window, total, system, memory, retrieved, history, system_counted`; property `context_tokens = memory + retrieved + history`; `as_dict()`.
- `plan_memory_budget(model, token_budget, output_reserve_tokens, system_reserve_tokens=0, system_text="", shares=(0.40,0.40,0.20))`:
  `total = max(0, min(token_budget, window - output_reserve - int(0.05*window)))`; `system = min(total, max(system_reserve, count(system_text)))`; remainder split by normalised shares into memory / retrieved / history (history takes the rounding remainder).
- `plan_from_settings(settings, system_text="", tier=None)`: model from `gateway.model_for(tier)` where `tier` is `"planning"` when `model_tiering_enabled` else `"default"` (verify against the most token-hungry tier, finding 4); reads the `memory_context_*` settings below.
- `estimate_plan(token_budget)`: estimate-only plan with `model=""`; used by tests and offline tooling, never calls the API.

**A2. Config (`src/core/config.py`)**: remove `memory_compaction_char_cap`; add `memory_context_token_budget=32000`, `memory_context_output_reserve_tokens=4096`, `memory_context_system_reserve_tokens=4000`, `memory_context_share_memory=0.40`, `memory_context_share_retrieved=0.40`, `memory_context_share_history=0.20`, `memory_token_count_exact=True`. Add warnings in the existing validator block (~`config.py:755`): budget > `anthropic_max_tokens_per_run` when that is > 0; shares not summing to 1.

**A3. `src/memory/retrieval.py`**
- `ContextBundle.char_budget` -> `token_budget`.
- `_mmr_select(candidates, chunks, *, budget, measure=len, ...)`: stop when summed `measure` of selected >= `budget` (v1 callers at the three `max_chars=` sites switch to `budget=` and keep the default `len`).
- `_compact_items(items, budget)`: measure with `TokenBudgetChecker.estimate_tokens` (+1 per item); only v2 uses it.
- `assemble_v2(..., plan: MemoryBudgetPlan | None = None)` replaces `char_cap`; `plan or estimate_plan(32000)`. Section budgets (tokens): split `plan.memory` across `ObjectiveNow/NonNegotiables/KnownFailures` by 0.16/0.22/0.10 normalised; `WhatChanged = plan.history`; `Evidence = plan.retrieved`. Wiki sub-cap `min(2300, Evidence//2)` (~8000 chars). Pass `measure=estimate_tokens` to both `_mmr_select` calls.
- Tail: `char_limit = int(plan.context_tokens * CHARS_PER_TOKEN)` only to bound string slicing; tier-4 collapse triggers when `estimate_tokens(assembled) >= plan.context_tokens` (keep `_context_collapse(sections, char_cap=char_limit)` signature: it slices characters). Then one `count_tokens(assembled, plan.model)`; if above `context_tokens`, trim proportionally (`assembled[: int(len*context_tokens/final_tokens)]`) and recount once.
- Metadata: drop `char_cap`, `token_budget_estimate`, `token_count_estimate`; add `token_budget`, `token_plan`, `token_count`, `token_count_exact`, `final_trim_applied`; `section_sizes`/`section_budgets` now tokens; `evidence_budget_chars/used_chars` -> `evidence_budget_tokens/used_tokens`.

**A4. Callers**: `src/agents/supervisor_execution.py` (both `assemble_v2` calls, ~lines 730 and 1269) pass `plan=plan_from_settings(settings)`; `src/workflows/orchestrator.py` (~1154) `char_cap` -> `token_budget=settings.memory_context_token_budget`; `src/workflows/retry_policy.py` `overflow_recovery_event(token_budget=...)` (key `char_cap` -> `token_budget`; no frontend consumer found); orchestrator gauges `context_sections_chars*` -> `context_sections_tokens*` (~2398-2400; no registered help text).

**A5. Docs/env**: `README.md` row ~637, `.env.example` ~207, local `.env` ~108: `MEMORY_COMPACTION_CHAR_CAP` -> `MEMORY_CONTEXT_TOKEN_BUDGET`, add `MEMORY_TOKEN_COUNT_EXACT`.

**A6. Tests**: `tests/conftest.py` sets `settings.memory_token_count_exact = False`. Update callers of `char_cap=` / `max_chars=`: `tests/unit/test_memory_retrieval.py` (assemble_v2 -> `plan=estimate_plan(N//4)`, `_mmr_select` -> `budget=`, budget test asserts `estimate_tokens(text) <= budget`; leave `_context_collapse` tests unchanged), `test_memory_upgrade.py`, `test_source_chunks_grounding.py` (line ~101), `test_v4_pipeline_contracts.py` (~168-169). New `tests/unit/test_context_budget.py`: buckets sum to context tokens; system reserve honoured; total capped by window (patch `context_window`); fallback to estimate when counting disabled / no model / non-Anthropic provider; `plan_from_settings` picks the gateway tier.

### Workstream B: budget coherence (after A)
B1. `TokenBudgetChecker` reads `settings.token_estimate_ratio` and `token_budget_safety_margin_tokens` (keep class constants as defaults; keep `estimate_tokens` signature).
B2. Resolve the dead per-call limits (finding 1): enforce them through Workstream C, or delete them. Recommended: enforce.
B3. 32K is the assembly **pool ceiling**; consumers get slices.

### Workstream C: consumer views (quality and cost)
C1. Replace char slices in `_grounded_context_excerpt` / `context_excerpt_block` (`src/prompts/prompt_hygiene.py:44`) with per-agent token caps (new `subagent_context_tokens`-style settings). Behaviour-neutral defaults (chars/3.5): 3000 -> ~860, 3500 -> 1000, 4000 -> 1150, 6000 -> 1700.
C2. Rank-first ordering: query-ranked evidence first, then curated context, so any slice keeps the best chunks.
C3. Tune per agent from offered/consumed metrics on real runs; raise a cap only where truncation correlates with weaker grounding.

### Workstream D: pool quality (after A)
D1. Budget-driven selection: `_mmr_select(max_items=15)` derived from remaining Evidence budget with a hard max (e.g. 40); tie the 120/80 snip counts and the wiki sub-cap to bucket budgets instead of constants.
D2. Redistribute unused budget across buckets instead of fixed 40/40/20 shares.
D3. Evaluate on real stored projects with `src/evaluation/evidence_validator.py` (coverage, groundedness) plus `section_utilization`; keep only changes that do not lower them.

### Workstream E: performance and scale (after A)
E1. Telemetry first: estimate-vs-exact error ratio, exact/fallback/trim counters, token versions of `subagent_context_offered/consumed`.
E2. Fast path: skip the exact count when the estimate is <= ~70% of the budget; set the threshold from E1 data, not by guess.
E3. Count hygiene: timeout ~3s (not 10s); treat 429 (honour `retry-after`) separately from connection errors; process-local cache is enough unless multi-worker hit rate is low.
E4. Prompt caching: measure only, using the `tokens_cache_*` totals in `src/llm/run_budget.py`. Adopt only if one large identical prefix is reused >= 3 times per run and exceeds the model's minimum cacheable size.

### Workstream F: cost accounting (optional)
Per-model price table or env overrides so Haiku 4.5 / Sonnet / Opus are not all priced $3/$15; fix the "$2-3 worth" comment at `config.py:290`.

## 5. Order and parallelism
Step 0 -> A (A1-A6 as one PR; A1 first) -> then B, C, D, E can proceed in parallel (C depends on B2's decision; D and E are independent). F anytime.

## 6. Out of scope
v1 `assemble` (characters; active only when `memory_compaction_v1_enabled=False`), conversation digest limits (`conversation_digest_*`), `supervisor_agent.py` `[:12000]` feedback truncation (lines 503/505), `web_capture` 32_000 char cap.

## 7. Verification
- Targeted: `pytest tests/unit/test_context_budget.py tests/unit/test_memory_retrieval.py tests/unit/test_memory_upgrade.py tests/unit/test_v4_pipeline_contracts.py tests/unit/test_source_chunks_grounding.py`. (The prototype passed 77 tests on these.)
- Full `tests/unit` failure set equals the Step 0 baseline.
- **Live check with a real key:** assemble one context; `metadata["token_plan"]` shows the model's real window, `token_count_exact` true, `final_trim_applied` false, count <= `memory+retrieved+history`. Repeat with a non-Anthropic `LLM_PROVIDER`: must fall back to the estimate with no error or long stall.
- After C/D: compare `evidence_validator` output and offered/consumed metrics on 2-3 real projects before vs after.
- After E1: confirm the data supports the fast-path threshold before enabling E2.

## 8. Risks
- R1: 32K tokens can send ~3.5x the old context to any consumer that does not slice it (extraction, process structuring); cost scales with actual content. Watch the new metrics after rollout.
- R2: `messages.count_tokens` and `models.retrieve().max_input_tokens` usage comes from SDK docs and has not been run live.
- R3: Workstreams C and D change what specialists see; ship behaviour-neutral defaults and tune from data.
- R4: The `.env` file is git-ignored; its edit (A5) must be made by each developer.
