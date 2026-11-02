# AI Agent Workflow & Integration Rules

## 1. Goal

Enable several AI agents to deliver independent pieces while preserving one coherent application. The orchestrator—not individual agents—owns integration order, shared-contract changes, and project memory.

## 2. Before starting a task

1. Read `memory.md` to learn current state and blockers.
2. Find the task in `tasks.md` and confirm its status/dependencies.
3. Read only the task's named context pack.
4. Inspect directly relevant code; do not recursively read the whole repository.
5. Confirm file ownership and branch/worktree.
6. State assumptions only when the docs leave a material gap; do not assume a new API or field.

## 3. Work isolation

- Use one branch/worktree per agent/task.
- Each task owns an explicit file set. Shared files (root config, route registry, API types, theme tokens) have one owner.
- If a task needs a shared file changed, send the orchestrator a small patch proposal rather than editing it concurrently.
- Avoid broad reformatting and unrelated renames.
- Commit in small logical units when Git is available.

## 4. Contract-first integration

- `api-contract.md` is the REST source of truth; `data-model.md` is the schema source of truth; `realtime.md` is the event source of truth.
- Backend route agents implement the documented contract; frontend agents consume it.
- If a contract is insufficient, propose: problem, current contract, proposed contract, sample JSON, security impact, data migration, affected task IDs.
- Orchestrator approves and updates docs before implementation.
- Generate/maintain TypeScript types from OpenAPI or a shared schema when practical. Until then, keep one typed API adapter aligned with the Markdown contract.

## 5. Integration sequence

1. Scaffold + DB + UI foundation merge.
2. Backend public ordering and staff APIs merge.
3. Payment/billing and reporting merge.
4. Customer/staff/owner UI features integrate against real API.
5. QA critical-path suite runs.
6. Deployment hardening and staging sandbox run.
7. Owner acceptance and production-readiness decision.

A frontend may use fixtures to build components before APIs exist, but fixtures must be isolated and removed from production paths when the real API is connected.

## 6. Definition of done

Agent returns exact changed files, commands/tests run, results, contract deviations, known limitations, and next dependency. Orchestrator checks diff, reruns impacted tests, updates `memory.md`, and changes task status. `DONE` means tested and integrated—not merely code generated.

## 7. Anti-patterns to reject

- One agent implementing the entire project in one giant prompt.
- Agents independently choosing endpoint names, database columns, enum values, or monetary units.
- Mock payment success treated as real payment success.
- Frontend-only role checks.
- Hard-coded GST/tax rates or client-calculated final totals.
- Microservices/Kubernetes added before a demonstrated need.
- Huge code dumps without tests, file ownership, or a completion report.
- Marking a task complete when it only compiles but its acceptance criteria are untested.

## 8. Compact agent handoff format

```text
TASK: <ID>
STATUS: DONE | NEEDS REVIEW | BLOCKED
FILES: <exact paths>
IMPLEMENTED: <1–4 bullets>
TESTS: <command + pass/fail; do not guess>
CONTRACT DEVIATIONS: NONE | <proposal>
LIMITATIONS: <specific>
NEXT DEPENDENCY: <task ID or none>
```
