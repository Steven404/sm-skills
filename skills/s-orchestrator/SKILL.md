---
name: s-orchestrator
description: Rules for choosing models, effort levels and delegation when orchestrating subagents or multi-model work. Use before starting any task that delegates, spawns agents, or picks a model or effort level.
---

# S-Orchestrator v1

**Author:** Stefanos Michelakis · **Last updated:** 2026-11-16

> Read this file in full before starting any task.

---

## 1. Models

| Tier         | Models                                                  |
| ------------ | ------------------------------------------------------- |
| **Frontier** | Opus latest, GPT Sol latest                             |
| **Medium**   | DeepSeek Flash latest, Sonnet latest, GPT Terra latest  |
| **Small**    | Haiku latest, GPT Luna latest                           |

"latest" means the newest generally available version the current tool offers. Skip preview, beta or experimental versions unless the user asks for them.

### Model reference

The models are written in order of preference. Use the first model in each row, and change it when:

- the user specifies otherwise
- the vendor's usage limit has been reached
- the vendor API experiences issues (move down one model then)

If the newest version of a model isn't available in this tool, use the newest one that is, before moving down the list.

---

## 2. Effort levels

Effort levels used in this ruleset, from lowest to highest:
`standard` → `above standard` → `before max` → `max`

| Ruleset level      | Anthropic  | OpenAI     | DeepSeek |
| ------------------ | ---------- | ---------- | -------- |
| **standard**       | medium     | medium     | default  |
| **above standard** | high       | high       | high     |
| **before max**     | extra high | extra high | max      |
| **max**            | max        | max        | max      |

---

## 3. Execution in T3 Code

- Run each stage on the model chosen by these rules using T3's `delegate_task` tool,
  not the provider's built-in subagent tool.
- Always pass the exact provider and model id. If unsure of the id, check
  `orchestrator_capabilities` first.

---

## 4. Step 0: Tag the task

Before anything else, tag the task:

- **type:** `plan` / `implement` / `routine-implement` / `test` / `automation` / `debug` / `refactor` / `docs` / `other`
- **risk:** `low` or `high` (high = touches any [high-risk case](#11-high-risk-cases))

> If the risk is unclear, treat it as **high** and say so.

> If the orchestrator is not a frontier model and the task is not clearly low-risk, delegate tagging and planning to a frontier model.

### Routine implementation

Use type `routine-implement` only when **all** of the following are true:

- risk is **low**, with no high-risk trigger
- the desired behavior and acceptance criteria are explicit
- an existing implementation pattern can be followed, or the approved plan specifies the approach
- the change is local and requires no independent architectural decisions or investigation of an unexplained bug
- no new dependency or shared configuration change is needed
- correctness has a clear check: an existing test, a specified new test, or a concrete visual check

Route routine implementation to a **small model, above standard effort**. Other low-risk implementation stays with a **medium model, standard effort**. If eligibility is unclear, use `implement` rather than `routine-implement`.

The frontier planner may classify individual subtasks as routine. Those subtasks are covered by the parent plan and need no separate planning. Do not split work into tiny delegations solely to qualify for this category.

Routine implementation is planned by a **medium model, standard effort**. The plan is short: the files to touch, the pattern to follow, the test scope, and the acceptance check. Before planning, the planner confirms every routine criterion above. If any fails, it re-tags the task as `implement` and hands planning to a frontier model.

Routine implementation follows the normal test scope, static checks, and review rules unless it also qualifies for the fast path below. High-risk implementation always uses the high-risk route.

If the implementer discovers that the routine criteria no longer hold, it must stop and report the reason for reclassification. Continue with a medium model for other low-risk implementation, or re-tag and re-plan if a high-risk trigger is found. Existing scope-approval rules still apply.

### Fast path

If **all** of the following are true:

- the change is **≤1 file** and roughly **≤50 changed lines**
- **no new dependency**
- type is `routine-implement` (all routine implementation criteria above apply)

then skip planning and test design, and run:

1. Write a detailed implementation prompt for a **small model, above standard effort**.
2. Implement.
3. Static checks.
4. Review (per rule 7).

> If the implementer finds the change needs another file, more than ~50 lines, or a new dependency, it must **stop** and switch to the normal flow.

### Work projects

If the user specifies that this is a **work project**:

1. **Do not start the orchestration** until you have presented the task tag (type, risk) and the model + effort for each stage to the human, and received approval.
2. Create a separate branch for each task before any changes are made:

- `feature/<task-name>` for new features and other non-fix work
- `fix/<task-name>` for bug fixes

3. Use a short, lowercase, hyphenated task name (e.g. `feature/custom-scrollbar`, `fix/login-redirect`).

---

## 5. Rules

| #   | Stage                                                  | Low risk                                                           | High risk                                       |
| --- | ------------------------------------------------------ | ------------------------------------------------------------------ | ----------------------------------------------- |
| 1   | **Planning**                                           | Medium, standard for routine; otherwise Frontier, above standard   | Frontier, before max                            |
| 2   | **Implementation**                                     | Small, above standard for routine; otherwise Medium, standard      | Frontier, above standard                        |
| 3   | **Test cases**                                         | Decided by the planner during planning, as part of the plan        | Same                                            |
| 4   | **Test code**                                          | Small, above standard                                              | Medium, standard                                |
| 5   | **CI/automation scripts, boilerplate, fixtures, docs** | Small, standard                                                    | Small, standard                                 |
| 6   | **Debugging & refactoring**                            | Medium, above standard                                             | Frontier, above standard                        |
| 7   | **Review**                                             | Sonnet latest, above standard                                      | Frontier from a different vendor, above standard **+ human approval before merge** |
| 8   | **Static checks**                                      | Always                                                             | Always                                          |

### Rule details

- **Review (7):** every implementation is reviewed in a **fresh context**, without the implementer's reasoning. The reviewer must be a different model from the implementer. When possible, prefer a different vendor if a model is available without changing the required review tier.
  - Use a **frontier model, above standard** instead of Sonnet for low-risk diffs that are large or span several modules.
  - If the Sonnet review is unsure about anything, escalate to a frontier review instead of approving.
  - **High risk:** the reviewer must be a frontier model from a **different vendor** than the implementer. If no other vendor's frontier model is available, use the available frontier model at **before max** in a fresh context, even if it is the implementer's model, and tell the human.
  - The integration review after parallel runs is always done by a **frontier model, above standard** (see [Parallel execution](#7-parallel-execution)).
- **Static checks (8):** linter, type checker, secret scan and dependency audit run on **every change**.

---

## 6. Test scope

- Test logic, not styling. Pure visual/CSS changes need no automated tests; list
  what to check manually instead.
- Unit/component tests for any non-trivial logic.
- E2E tests only for critical user flows (auth, payments, core create/save/delete
  flows) and for high-risk tasks.
- The plan states which test levels apply and why.

---

## 7. Parallel execution

- Only run tasks in parallel if they touch different files or modules.
  If unsure, run them sequentially.
- Each parallel agent works in its own git worktree or branch, never the same checkout.
- Shared resources (database, ports, migrations, lockfiles) are never changed in parallel.
- After parallel tasks finish, one frontier model at above standard integrates and reviews the combined result,
  then static checks and tests run on the merged code.

---

## 8. Escalation

1. **Failure or weak output:** retry the same model one effort level higher. If already at `max`, move up one tier.
2. **Still failing:** move up one tier.
3. **Still failing after 3 attempts total:** stop and report to the human.
4. **Reviewer finds a security issue in a low-risk task:** re-tag as high risk and re-plan.

---

## 9. Ask the human before acting when

- the task fits no rule above
- the implementation needs files or modules the plan did not mention
- the escalation ladder is exhausted
- any **irreversible action**: DB migration, deleting data, deploying, pushing to `main`

---

## 10. New dependencies

Before installing a new dependency, verify that:

- [ ] the package exists on the official registry and the name is spelled **exactly** right
- [ ] it is actively maintained and reasonably popular
- [ ] its license is compatible with the project

> Ask the human before adding **any** dependency to a high-risk task.

---

## 11. High-risk cases

A task is **high risk** if it touches any of the following:

- **Auth and authorization** (login, register, roles, etc.)
- **Secrets, sensitive data storage, encryption**
- **Input handling at trust boundaries** (injection, file uploads, deserialization)
- **Payments** and money in general
- **Irreversible DB data changes**
- **Public API contracts** and anything other systems depend on
- **Concurrency and caching logic**
