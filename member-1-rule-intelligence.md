# Member 1 — Rule Intelligence
### Owns: Behavioral Contract Graph, Rule Miner Agent, shared LLM client, shared schemas

> Upload this file together with `00-invariantos-project-overview.md`. Read the overview first — this file assumes you know the architecture, folder layout, and data contracts already defined there. Do not redefine schemas; you own `orchestrator/schemas.py` but its shapes come from Section 5 of the overview.

## Your mission

Build the pipeline that turns a repository's scattered code, docs, tests, and tickets into a structured, queryable **Behavioral Contract Graph** (`data/rules.json`) — the foundation every other agent depends on. You are also responsible for the shared LLM client that Members 2 and 3 will import.

## What you own

```
orchestrator/schemas.py       # Pydantic models for Rule, ImpactedRule, ValidationResult,
                               # SecurityFinding, TestGap, AnalysisReport (Section 5 of overview)
orchestrator/llm_client.py     # shared LLM wrapper — everyone else imports this, don't change its
                                # public function signature after hour ~6
orchestrator/agents/rule_miner.py
orchestrator/data/rules.json    # generated output, also commit a stub version early
```

## Step-by-step build

1. **Scaffold the repo (hour 0–1).** Create the folder structure from Section 4 of the overview (even the folders you don't own, as empty dirs with `.gitkeep`, so teammates can `git pull` a working skeleton). Add `requirements.txt` with `fastapi`, `uvicorn`, `pydantic`, `python-dotenv`, `ibm-watsonx-ai`, `openai`, `gitpython`. Add `.env.example` exactly as in overview Section 8. Add `.gitignore`/`.bobignore` from the IBM hackathon template.

2. **Write `orchestrator/schemas.py` (hour 1–2).** Implement Pydantic models for every object in overview Section 5 (`Rule`, `ImpactedRule`, `ValidationResult`, `SecurityFinding`, `TestGap`, `AnalysisReport`). Use exactly the field names shown there — Members 2, 3, and 4 will import these classes directly. Push this first; it unblocks everyone.

3. **Write a stub `data/rules.json` by hand (hour 2–3).** 6–8 realistic `Rule` objects for a small e-commerce domain (orders, payments, shipments, refunds, cross-org access), including this exact one so the demo works later:
   ```json
   {
     "id": "RULE-001",
     "statement": "A cancelled order must never be shipped.",
     "category": "order-lifecycle",
     "severity": "critical",
     "source_refs": [{"file": "docs/order-lifecycle.md", "lines": "12-14"}],
     "related_entities": ["Order", "Shipment", "OrderStatus"],
     "related_functions": ["createShipment", "updateOrderStatus"],
     "tags": ["fulfillment", "order-status"]
   }
   ```
   Commit and push immediately — this is the file Members 2 and 4 build against for the next ~14 hours.

4. **Build `orchestrator/llm_client.py` (hour 3–6).** A single function:
   ```python
   def complete(system_prompt: str, user_prompt: str, json_mode: bool = False) -> str:
       ...
   ```
   Internally: if `WATSONX_API_KEY` env var is set, call IBM watsonx.ai; otherwise fall back to an OpenAI-compatible client using `OPENAI_API_KEY`. When `json_mode=True`, instruct the model to return ONLY raw JSON (no markdown fences, no preamble) and strip any accidental fences before returning. Wrap the network call in try/except and raise a clear `LLMClientError` on failure so callers can degrade gracefully. This is the only function other agents should use to talk to an LLM — keep its signature stable after hour 6.

5. **Build the real Rule Miner (hour 6–16).** `orchestrator/agents/rule_miner.py`:
   ```python
   def mine_rules(repo_path: str) -> list[Rule]:
       ...
   ```
   - Walk `repo_path` and collect text from: `docs/**/*.md`, `postmortems/**/*.md`, `tickets/**/*.md`, source comments, and test file names/descriptions (these come from Member 3's `demo-repo/`).
   - Chunk the collected text (by file, capped at a safe token size) and, for each chunk, call `llm_client.complete()` with a system prompt like: *"You are extracting hidden business invariants from software artifacts. For each rule you find, output a JSON object matching this exact schema: {schema}. Only extract rules that describe behavior that must always/never happen — not implementation details."* — pass the `Rule` JSON schema inline so output matches Section 5 exactly.
   - Parse and validate each LLM response against the `Rule` Pydantic model; drop anything that fails validation rather than crashing the pipeline.
   - Deduplicate near-identical rules (simple: dedupe on the first ~8 words of `statement`, lowercased).
   - Write the final merged list to `data/rules.json`, always including your hand-written `RULE-001` if it isn't already found (safety net for the demo).
   - Wire it to `POST /api/rules/extract` (Member 2 will have the FastAPI shell ready — coordinate at Checkpoint 1, hour ~16).

6. **Checkpoint 1 (hour ~16).** Run `mine_rules("demo-repo")` against Member 3's real demo repo content and confirm at least 8 valid rules come out, including the shipment rule. Replace the stub `rules.json` with real output. Tell Member 2 and Member 4 the schema hasn't changed.

7. **Hour 16–30: harden.** Add a `related_functions` inference pass if the LLM under-fills it (simple heuristic: regex-scan the referenced source files for function names near the cited docs). Add basic caching so re-running `mine_rules` on an unchanged repo doesn't re-call the LLM for unchanged files (hash each file, skip if hash unchanged).

8. **Hour 30–40.** Help Member 2 debug any schema mismatches surfacing during full pipeline runs. Take your Bob task-session screenshot and save it to `docs/bob-screenshots/member-1/`.

## Definition of done

- `orchestrator/schemas.py` matches overview Section 5 exactly and is imported by every agent.
- `llm_client.complete()` works against at least one real provider.
- `data/rules.json` contains ≥8 valid rules mined from the real demo repo, including `RULE-001` (the shipment rule) with accurate `source_refs`.
- `mine_rules()` never crashes on malformed LLM output — it validates and drops bad entries.

## Handoffs

- Give Member 2 the stub `rules.json` by hour ~3 (don't wait for the real miner).
- Give Member 3 nothing — they don't depend on you; you depend on their `demo-repo/` docs.
- Give Member 4 the finalized `schemas.py` by hour ~2 so the dashboard can start rendering mock `AnalysisReport` objects immediately.
