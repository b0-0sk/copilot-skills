# GitHub Copilot Skills

This repository contains five skills that work together to turn an initial
software request into a clarified, validated task and, when authorized, an
implementation with appropriate testing:

- [Prompt Builder](prompt-builder/SKILL.md)
- [Prompt Cycle](prompt-cycle/SKILL.md)
- [Prompt Quality Gate](prompt-quality-gate/SKILL.md)
- [Engineering Workflow](engineering-workflow/SKILL.md)
- [Acceptance E2E](acceptance-e2e/SKILL.md)

## Install

Install the skill folders in a location supported by your GitHub Copilot
environment. For personal skills, the usual location is `~/.copilot/skills`.
Keep each skill's folder structure intact, including the references under
`acceptance-e2e/references/`.

## Quick start

Start with `/prompt-builder` or `/prompt-cycle` and describe the task in your own
words. Both entry points use the same workflow. State the desired action and
constraints when you know them; for example:

> Analyze why the search endpoint is slow. Do not change files. Identify the
> likely cause and recommend options.

The workflow prepares a detailed prompt, validates it with the Prompt Quality
Gate, and asks for missing decisions before execution. It does not start
implementation just because the initial request mentioned a code change.

## The complete workflow

1. **Describe the outcome.** Explain the problem, expected behavior, what must
   stay unchanged, scope, constraints, and how success should be verified.
   Include the desired mode and priority if known.
2. **Refine and validate.** Prompt Builder turns the request into a
   repository-independent, actionable prompt. Prompt Quality Gate evaluates it
   on a 0-10 scale and targets 10/10. It may block on important missing
   requirements, such as a security boundary or destructive behavior.
3. **Answer clarification questions.** If a decision would change the result,
   the workflow asks you to resolve it. Reply with the requirement or choose an
   offered option. The prompt is revised and validated again. A critical
   security, data-integrity, destructive, or business decision must not be
   guessed. If you skip a non-critical question, the unresolved point may be
   recorded as an explicit assumption; it is never presented as confirmed.
4. **Choose whether to execute.** After the prompt passes the gate, choose
   **Lanzar** to run it, **No lanzar** to stop, or use the form's free-text
   option to request a change. A later message consisting of `LANZAR` (uppercase)
   also launches the last approved prompt. The free-text path revises and
   revalidates the prompt before showing the launch choice again.
5. **Review the result.** Engineering Workflow reports the selected mode,
   decision, affected files, checks actually run, their results, and remaining
   risks. A generated but unexecuted test is not reported as passing.

## Choose an action

Set `Mode` in the request so the work stays within the intended boundary:

| Mode | What it does |
| --- | --- |
| `analyze` | Investigates the current behavior, explains findings, and recommends an approach. It does not edit files. |
| `design` | Produces a repository-grounded design and implementation plan. It does not edit files. |
| `implement` | Makes the approved change, adds relevant tests when appropriate, and runs permitted validation. |
| `review` | Reviews the requested changes and reports evidence-based findings without editing. |
| `refactor` | Changes structure while preserving observable behavior, then validates the result. Use only when refactoring is requested. |

If the action or priority is missing, Engineering Workflow asks you to choose
before editing. Priorities include existing patterns, test-driven quality, and
security and performance.

### Analyze without changing code

Use `/prompt-cycle` or `/prompt-builder` and explicitly request `Mode: analyze`.
State that you want findings and a recommendation only. Example:

> Investigate why the report generation job is slow. Do not modify code. Compare
> realistic options and recommend one based on repository evidence.

### Ask questions and clarify requirements

You do not need to know every implementation detail before starting. The
workflow asks when a missing decision materially affects behavior, scope,
security, or acceptance criteria. Answer with the intended rule, not with a
guess about the code. Repository facts such as existing frameworks and file
locations are discovered during implementation rather than asked of you.

Example:

> Help me specify support for multiple authentication providers. Ask me about
> any trust or authorization rules that are not clear before proposing changes.

### Implement a change

Choose `Mode: implement`, state the desired behavior and what must not change,
and include acceptance criteria. Add a priority such as `existing patterns` or
`security and performance` when relevant.

> Mode: implement. Priority: existing patterns. Add CSV export to the reports
> page, preserve the current filters, and verify that the downloaded file has
> the expected headers and rows.

The workflow inspects applicable `AGENTS.md` files, existing architecture, and
relevant tests before editing. In `analyze` and `design` modes it remains
read-only, even if an implementation is mentioned as a possible next step.

## Testing

Testing is part of `implement` when it is needed to validate a change; it is
not a separate Engineering Workflow mode. You can also explicitly request the
Acceptance E2E skill when the task is to generate or run acceptance tests.

Engineering Workflow first checks the change's observable impact and the
project's existing test setup. It runs the smallest relevant existing checks.
End-to-end tests are appropriate for important user-visible journeys or
behavior across public boundaries when unit or component tests are not enough.
If there is no relevant API or observable behavior change, E2E may not be
needed.

Before creating or updating E2E tests, and again before executing them, the
workflow asks for confirmation unless you already authorized those actions in
the approved prompt. Tests must target a disposable local environment or an
authorized test environment; production is never the default.

### E2E stages and production safety

Acceptance E2E supports these stage names:

- `local`: disposable local application or test stack.
- `test` and `integration`: authorized non-production environments.
- `prod`: optional read-only smoke tests only.

When an E2E suite has no saved `Stage scope` decision, it asks which stages to
cover and whether to defer or decline the others. The decision is recorded in
that suite's README so the same question is not repeated unnecessarily.
Credentials belong in environment variables, never in the stage configuration
files or test output. Production runs require explicit confirmation before
every execution; mutation requests are blocked.

For local API tests, a command may look like this once the project-specific test
directory and dependencies are known:

```powershell
py -m pytest <e2e-directory> -q
```

Use the command and configuration discovered for the project rather than
assuming this placeholder applies everywhere. See the
[API testing guide](acceptance-e2e/references/api-testing.md),
[UI testing guide](acceptance-e2e/references/ui-testing.md),
[Compose isolation guide](acceptance-e2e/references/compose-isolation.md), and
[stage configuration guide](acceptance-e2e/references/stages.md).

## Related skill responsibilities

- [Prompt Builder](prompt-builder/SKILL.md) drafts and refines the request.
- [Prompt Cycle](prompt-cycle/SKILL.md) coordinates clarification, validation,
  and the launch decision.
- [Prompt Quality Gate](prompt-quality-gate/SKILL.md) scores the prompt and
  identifies blockers; it does not implement the request.
- [Engineering Workflow](engineering-workflow/SKILL.md) performs the selected
  analysis, design, implementation, review, or refactor.
- [Acceptance E2E](acceptance-e2e/SKILL.md) guides black-box acceptance tests
  when requested or warranted by the approved change.

For security and data safety, use test credentials only, keep secrets out of
files and logs, and do not run production mutations. Never treat silence,
unanswered questions, or an unrun check as approval or success.
