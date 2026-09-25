# Member 2 — Impact & Validation Engine
### Owns: Change Impact Agent, Contract Validation Agent, the FastAPI orchestrator, the agent-mode pipeline

> Upload this file together with `00-invariantos-project-overview.md`. Read the overview first — this file assumes you know the architecture, folder layout, and data contracts already defined there.

## Your mission

You are the core reasoning engine and the glue that turns a raw PR diff into a verdict. You build the two agents that decide *what a change might break* and *whether it actually does*, and you own the FastAPI app and the "agent mode" orchestration that runs everyone's agents in the right order.

## What you own

```
orchestrator/main.py             # FastAPI app + the 5 endpoints from overview Section 6
orchestrator/orchestrator.py      # the pipeline: runs agents in order, assembles the AnalysisReport
orchestrator/agents/change_impact.py
orchestrator/agents/contract_validator.py
```

## Step-by-step build

1. **Hour 0–2: stand up the API shell.** Build `orchestrator/main.py` with FastAPI, implementing the 5 endpoints from overview Section 6 as stubs that return hardcoded example objects matching the schemas exactly (import `Rule`, `AnalysisReport`, etc. from Member 1's `schemas.py` — coordinate so it lands by hour ~2). This unblocks Member 4's dashboard immediately. Add CORS middleware open to all origins (hackathon speed, not production).

2. **Hour 2–4: diff parsing utility.** Write a small helper (e.g. `orchestrator/diff_utils.py`) that takes a raw unified-diff string and extracts: changed file paths, changed function/symbol names (simple heuristic: nearest preceding `function`/`def`/`class` line above each hunk), and the added/removed lines. This feeds the Change Impact Agent. Test it against the seeded diff from overview Section 3.

3. **Hour 4–12: build the Change Impact Agent.** `orchestrator/agents/change_impact.py`:
   ```python
   def find_impacted_rules(diff: str, rules: list[Rule]) -> list[ImpactedRule]:
       ...
   ```
   - First pass, cheap and deterministic: for each `Rule`, check if any of its `related_functions` or `related_entities` string-match the diff's changed functions/files (from step 2). Collect candidates.
   - Second pass, LLM-assisted (via Member 1's `llm_client.complete()`): send the diff plus the candidate rules' statements and ask the model to confirm relevance, write a one-sentence `reason`, assign `confidence` (`high`/`medium`/`low`), and reconstruct the `affected_call_chain` (it can infer this from function names it sees in the diff and surrounding repo context you pass in).
   - Validate output against `ImpactedRule`. If the LLM call fails, fall back to the deterministic-match results with `confidence: "medium"` and a generic reason — never let the pipeline crash for lack of an LLM response.
   - Confirm against the seeded diff (overview Section 3) that `RULE-001` is found with the correct call chain.

4. **Hour 12–20: build the Contract Validation Agent.** `orchestrator/agents/contract_validator.py`:
   ```python
   def validate(diff: str, impacted_rules: list[ImpactedRule], rules: list[Rule]) -> list[ValidationResult]:
       ...
   ```
   - For each impacted rule, prompt the LLM with: the rule's `statement`, the relevant diff hunk, and any cited `source_refs` text if available. Ask for a strict verdict of `OK`, `VIOLATION`, or `NEEDS_EVIDENCE`, plus a one-paragraph `explanation` and a list of `evidence` strings (line refs / doc refs).
   - Validate against `ValidationResult`; on failure/timeout, default to `NEEDS_EVIDENCE` (never silently mark something `OK` on error — fail closed, not open).
   - Confirm the seeded diff produces `VIOLATION` for `RULE-001`.

5. **Hour 20–26: build `orchestrator/orchestrator.py`, the agent-mode pipeline.** One function:
   ```python
   def run_pipeline(diff: str) -> AnalysisReport:
       ...
   ```
   Sequence:
   1. Load `data/rules.json` (Member 1's output).
   2. Run `find_impacted_rules()`.
   3. Run, in parallel (`asyncio.gather` or a `ThreadPoolExecutor`, to genuinely demonstrate "parallel subagents"): `validate()` (yours), `security_access.check()` (Member 3's), `test_gap.analyze()` (Member 3's).
   4. Pass everything to `evidence_report.generate()` (Member 4's) to get the final `AnalysisReport`.
   5. Compute `final_verdict`: `BLOCK` if any `ValidationResult.verdict == "VIOLATION"` or any `SecurityFinding.verdict != "OK"`; else `NEEDS_EVIDENCE` if any result is `NEEDS_EVIDENCE` or a `TestGap.has_coverage == False` on a critical rule; else `SAFE`.
   6. Save the report to `data/analyses/<analysis_id>.json` and return it.
   - Wire this into `POST /api/analyze` in `main.py`, replacing the stub.

6. **Hour ~16 and ~30: Checkpoints.** At hour 16, make sure the real `/api/rules/extract` and a first pass of `/api/analyze` work against Member 1's mined rules, even if Members 3/4's pieces are still stubbed (stub their function signatures yourself if they're not ready, matching the schemas, so your pipeline never blocks). At hour 30, everything should be real: run the full before/after demo scenario from overview Section 3 twice (buggy diff → `BLOCK`, fixed diff → `SAFE`) and confirm it matches exactly.

7. **Hour 30–40.** Add basic error handling / input validation to the API (reject empty diffs with HTTP 400), add a `docs/architecture.md` sequence diagram of the pipeline (text/ASCII is fine), help debug integration issues, take your Bob screenshot into `docs/bob-screenshots/member-2/`.

## Definition of done

- All 5 endpoints from overview Section 6 are implemented for real and validate request/response bodies against the shared Pydantic schemas.
- `run_pipeline()` runs the demo diff and returns `final_verdict: "BLOCK"` citing `RULE-001`, the correct call chain, and a non-empty explanation.
- Running `run_pipeline()` on the fixed version of the diff returns `final_verdict: "SAFE"`.
- No unhandled exception can crash `/api/analyze` — every agent call is wrapped and degrades to a safe default (`NEEDS_EVIDENCE`, never a silent `OK`).

## Handoffs

- Publish the FastAPI stub with mock responses by hour ~2 so Member 4 can build the dashboard against a live (if fake) API immediately.
- Agree on function signatures with Member 3 for `security_access.check()` and `test_gap.analyze()` by hour ~4, even before either is implemented for real, so your orchestrator can call stubs and swap them later without changing `orchestrator.py`.
