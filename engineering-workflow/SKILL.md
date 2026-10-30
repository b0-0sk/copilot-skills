---
name: engineering-workflow
description: Interactive engineering workflow for turning vague software requests into repository-grounded analysis, designs, implementations or code reviews. Invoke explicitly to choose mode and priorities before proceeding, or run it for an approved prompt-cycle prompt when the user picks "Lanzar" in the launch form (or sends the whole message LANZAR). Do not activate for unrelated tasks.
---

## GitHub Copilot compatibility

Use this skill in GitHub Copilot desktop, CLI, VS Code or JetBrains agent mode. Use only tools actually available in the current client. To invoke a related skill, discover and read its installed SKILL.md; its name alone is not an executable tool call. Resolve personal skills relative to ~/.copilot/skills and project skills from supported repository skill directories. Prefer the client-provided installed skill location when available. Use a native question tool when available; otherwise ask in chat and wait for the answer. Never infer an answer or approval from silence. On Windows use py and py -m for Python commands unless the project explicitly requires another interpreter.


# Engineering workflow

This skill routes an informal engineering request into the user's chosen approach. Prefer the project's existing architecture and conventions; do not invent repository facts.

## 0. Approved request precondition (integration with prompt-cycle)

This plugin separates requirements approval from implementation. For ordinary development tasks initiated through the harness, **do not inspect the repository or modify code until an approved prompt has been produced by `prompt-cycle` and the user has subsequently explicitly requested execution**. A new request containing the approved prompt, or the handoff word below, is sufficient. If the request is still vague or has no approved specification in conversation, redirect to `prompt-cycle` and stop; don't self-approve. This is behavioral guidance, not an OS-level execution permission gate.

**Execution request.** The user's "Lanzar" choice in the `prompt-cycle` launch form, or the handoff word `LANZAR`, is the explicit execution request for the **last approved prompt** (`Estado: APROBADO` or `APROBADO CON SUPOSICIONES`) in the conversation. For the word: it must be the user's whole message (uppercase, exact; the lowercase verb inside ordinary text does not count). Execute that prompt immediately: take `Modo` and `Prioridad` from it and do not re-ask them (ask only for a value that is missing). If its "Suposiciones y delegaciones" is not "Ninguna", print the assumptions in one line before starting. If no approved prompt exists in the conversation, execute nothing: tell the user and redirect to `prompt-cycle`. These never approve a prompt; they only authorize execution of one already approved.

If the user explicitly requests a standalone engineering analysis outside this harness, follow the user's request without claiming it has passed `prompt-cycle`. When in doubt, ask for approval before modifications.

## 1. Select the approach

When invoked without explicit mode or focus, collect the user's choice **before editing any files**.

Use a native question tool if available, otherwise ask in chat. Collect only missing choices:

1. **Action:** Analyze only / Design and plan / Implement and validate; allow review, refactoring or a custom action.
2. **Priority:** Reuse existing patterns / Test-driven quality / Security and performance; allow a custom priority.

Do not ask for a separate depth choice by default. Infer effort from task size and risk. Do not edit until the action is selected, unless already explicit in the invocation. Respect read-only or planning modes enforced by the current client.

When action and priority are explicitly provided in the prompt, honor them without reasking.

## 2. Inspect evidence, not the entire repository

- Read the applicable `AGENTS.md` and only relevant local docs.
- Identify candidate codepaths, existing implementations, dependency boundaries, contracts, and tests. Include whether the change affects API behavior and whether an e2e suite exists (see section 5).
- Separate observed facts from assumptions and missing business requirements.
- Ask a blocking clarification only when ambiguity could materially change correctness, behavior, or safety.
- Scale exploration and planning with architectural risk and blast radius; avoid exhaustive repository scans for localized changes.

## 3. Apply the selected mode

**Analyze only:** Explain the current design with file references, identify the issue, compare realistic alternatives, and recommend a solution. Never edit project files.

**Design and plan:** Provide requirements, acceptance criteria, affected components, alternatives and tradeoffs, an incremental plan, validation and rollback concerns where relevant. Never modify project files. In a read-only or planning mode, remain non-writing regardless of any requested action.

**Implement and validate:** Determine testable acceptance criteria, reuse project patterns, make the smallest coherent code change, add relevant tests, execute permitted checks, inspect the final diff, and report actual results. In Plan mode, produce the implementation plan and wait for a permitted implementation context rather than claiming to edit.

**Review / refactor (if selected via Other or explicitly):** For review, inspect the relevant diff and report concrete findings prioritized by severity, with evidence and minimal fixes; do not edit. For refactoring, preserve observable behavior, add characterization tests where needed, minimize change scope, then verify. Refactor only when the user explicitly selects that action.

## 4. Apply the selected priority

- **Existing patterns (default):** Prefer established repository conventions; introduce new abstractions only where justified.
- **Test-driven quality:** Use red-green-refactor when practical, emphasize edge cases and observable behavior, but do not force ceremonial TDD when impractical.
- **Security and performance:** Threat-model or profile only the affected paths; validate trust boundaries, concurrency, sensitive data, and performance claims when relevant. Do not assume a particular caching or security technology.
- **Custom:** Follow the user's specific focus with the same evidence-first discipline.

Always apply SOLID, KISS, DRY, and YAGNI pragmatically. Do not force named design patterns or add unnecessary classes, interfaces, tools, or dependencies.

## 5. Acceptance E2E when warranted

**Detection (during inspection, step 2).** Decide whether the change alters observable API behavior (endpoints, contracts, authentication, integrations, multi-service workflows), or whether the project exposes APIs at all, and whether an e2e suite already exists (`<project>/.copilot/tools/e2e` or the project's existing location). If there are no APIs or no observable impact, state "no e2e needed" with the reason and do not ask anything. UI-only or internal-only changes follow the same reasoning.

**Confirmation gates (never skipped, even for an approved prompt).** When e2e applies and the user has not already authorized it in the approved prompt:
1. Ask before **generating or updating** e2e tests (if none exist, say so and offer to create them).
2. Ask before **executing** them.

Use `ask_user` with explicit choices for each gate. A "no" is recorded as unrun e2e in the final report, never as a pass.

**Procedure.** After unit/integration checks, read `acceptance-e2e/SKILL.md` from the installed location (it is a procedure, not an autonomous skill-to-skill call) and follow it, including its `.copilot/tools` bootstrap and local-vs-repo question. Run **only** in a safe disposable test environment; never start production or unrelated containers. If setup or permissions are missing, report BLOCKED rather than treating unrun tests as passing. Use the generated/updated e2e as part of the workflow's validation and report its result (PASS, FAIL or BLOCKED) with evidence.

## 6. Output and limits

Summarize the selected mode and priority, findings, decision, affected files, tests actually run and results, and unresolved risks. Cite repository file paths when available. Never claim a verification happened if it did not.

Do not repeat these instructions or produce a giant rewritten prompt unless specifically requested. Avoid unrelated code edits, commits, pushes, deploys, or changes to external systems without explicit user authorization.
