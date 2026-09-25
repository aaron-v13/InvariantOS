# Member 3 — Test Gap, Security & Demo Repo
### Owns: Test Gap Agent, Security & Access Agent, the seeded demo e-commerce repository, optional GitHub integration

> Upload this file together with `00-invariantos-project-overview.md`. Read the overview first — this file assumes you know the architecture, folder layout, and data contracts already defined there.

## Your mission

You build the "proof" side of InvariantOS — regression tests and cross-tenant/permission checks — and, critically, you build the **demo repository itself**: the sample e-commerce app with the seeded rule and the seeded bug that the entire team's demo depends on. Ship the demo repo skeleton first; everyone else needs real material to work against.

## What you own

```
demo-repo/                      # a small Node/Express e-commerce app
  src/orders.js, payments.js, shipments.js, warehouseApi.js
  tests/orders.test.js, shipments.test.js
  tests/generated/               # Member 2's Test Gap Agent writes generated tests here
  docs/order-lifecycle.md         # describes "cancelled orders must never be shipped" in prose
  postmortems/2025-11-refund-bug.md
  tickets/TICKET-142-cancel-then-ship.md
  .git branches: main (fixed) and feature/cancel-status-bug (the seeded bug from overview Section 3)
orchestrator/agents/test_gap.py
orchestrator/agents/security_access.py
.github/workflows/invariantos-pr-check.yml   # optional, only if time allows after everything else is done
```

## Step-by-step build

1. **Hour 0–4: build the demo repo skeleton FIRST — this is the team's critical path.**
   - `src/orders.js`: an `Order` model/functions with statuses `PENDING`, `PAID`, `CANCELLED`, `REFUNDED`, `SHIPPED`, including `cancelOrder(order)` and `updateOrderStatus(order, newStatus)`.
   - `src/shipments.js`: a `shipmentJob(order)` that calls `warehouseApi.createShipment(order)`, guarded today by the **correct, fixed** condition `if (order.status === "PAID") { createShipment(order); }`.
   - `src/warehouseApi.js`: a stub `createShipment(order)` that just logs/returns.
   - `tests/orders.test.js`, `tests/shipments.test.js`: a handful of realistic Jest tests that all pass against the fixed version and would **still pass** even with the seeded bug (this is the whole point — normal CI doesn't catch it).
   - `docs/order-lifecycle.md`: 1–2 paragraphs of prose stating plainly "A cancelled order must never be shipped, even if it was previously paid," plus 2–3 other order-lifecycle rules (e.g. refund limits, one-time discount use).
   - `postmortems/2025-11-refund-bug.md` and `tickets/TICKET-142-cancel-then-ship.md`: short, realistic writeups referencing the same rule from a past incident angle — this is what makes the rule-mining demo compelling (the same rule is derivable from three independent sources).
   - Create branch `feature/cancel-status-bug` where `shipments.js`'s guard is changed to the buggy version from overview Section 3: `if (order.status !== "CANCELLED")`. Export the raw diff between `main` and this branch to a file, e.g. `demo-repo/seeded-diff.patch` — Member 2 needs this exact text for testing.
   - Push all of this by hour ~4. This is the single most time-sensitive deliverable in the whole project — Members 1, 2, and 4 all depend on it.

2. **Hour 4–6: add 2–3 more small rule scenarios** to the demo repo the same way (e.g. "a discount code must not apply twice," "a support agent must not modify payment details, only view them" — this last one is your Security & Access hook). Keep each to one function + one doc paragraph; you don't need a full app, just enough surface area for ~8 total rules across the repo.

3. **Hour 6–16: build the Security & Access Agent.** `orchestrator/agents/security_access.py`:
   ```python
   def check(diff: str, impacted_rules: list[ImpactedRule]) -> list[SecurityFinding]:
       ...
   ```
   - Filter `impacted_rules` to ones tagged with something like `access-control`/`security` in the `Rule.tags` (coordinate the tag name with Member 1).
   - For each, run a cheap heuristic first: regex-scan the diff for patterns like tenant/org id checks, role checks, or permission decorators being added/removed. Then confirm with an LLM call (via `llm_client.complete()`) asking specifically: "Does this diff create a cross-tenant data exposure or permission bypass relative to this rule?" Return `OK` or a specific `risk_type` (`cross-tenant-exposure`, `permission-bypass`, `sensitive-data-leak`) with an explanation.
   - Validate against `SecurityFinding`. Default to `OK` only when the heuristic AND the LLM both find nothing — otherwise flag it (fail closed).

4. **Hour 16–28: build the Test Gap Agent.** `orchestrator/agents/test_gap.py`:
   ```python
   def analyze(diff: str, impacted_rules: list[ImpactedRule], repo_path: str) -> list[TestGap]:
       ...
   ```
   - For each impacted rule, check whether `repo_path/tests/**` already has a test that exercises the specific scenario (cheap heuristic: does any test file mention the rule's `related_functions` AND the specific status/condition from the diff? e.g. does a test call `updateOrderStatus` with `CANCELLED` and then assert `createShipment` was NOT called?). If yes, `has_coverage: true`.
   - If no, call the LLM with the rule statement, the diff, and an example existing test file as a style reference, asking it to generate a new Jest test that reproduces the violation (should fail against the buggy diff, pass against the fix). Write the returned code to `demo-repo/tests/generated/rule_<id>_regression.test.js` and record that path in `TestGap.generated_test_path`.
   - Validate against `TestGap`. Confirm end-to-end: the generated test for `RULE-001` actually fails when run against `feature/cancel-status-bug` and passes against `main` (actually run `npx jest` on both branches to prove it — this is your strongest demo evidence).

5. **Hour ~16 and ~30: Checkpoints.** Coordinate signatures with Member 2 early (hour ~4) so your two functions can be stubbed and swapped in without changing `orchestrator.py`. At hour 30, confirm your two agents run correctly inside the full pipeline against both the buggy and fixed diffs.

6. **Hour 30–36 (optional, only if ahead of schedule): GitHub integration.** `.github/workflows/invariantos-pr-check.yml` — a GitHub Action that, on `pull_request`, computes the diff, POSTs it to a locally-tunneled or deployed `/api/analyze`, and posts the `summary_markdown` from the `AnalysisReport` as a PR comment via the GitHub API using `GITHUB_TOKEN`. Treat this as a stretch goal — a live dashboard demo (Member 4) is sufficient and safer for the video than depending on live GitHub Actions during recording.

7. **Hour 36–40.** Polish demo-repo docs for readability (the video may briefly show them), double check both git branches are clean and pushable, take your Bob screenshot into `docs/bob-screenshots/member-3/`.

## Definition of done

- `demo-repo/` on `main` has the correct guard, passing tests, and the seed docs/tickets/postmortems describing ≥3 rules.
- `feature/cancel-status-bug` has exactly the buggy diff from overview Section 3, and `seeded-diff.patch` matches it exactly.
- `security_access.check()` correctly flags the "support agent can view but not modify payment details" scenario as a violation when tested with a deliberately bad diff, and returns `OK` on an unrelated diff.
- `test_gap.analyze()` generates a real Jest test for `RULE-001` that demonstrably fails on the buggy branch and passes on `main`.

## Handoffs

- Ship the demo repo skeleton (step 1) by hour ~4 — this blocks Member 1 (needs docs to mine) and Member 2 (needs the exact diff text to test against).
- Agree on `security_access.check()` and `test_gap.analyze()` function signatures with Member 2 by hour ~4.
- Give Member 1 the `Rule.tags` naming convention for security-related rules so their miner and your filter agree (e.g. always tag access-control rules with `"access-control"`).
