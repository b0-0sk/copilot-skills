---
name: prompt-builder
description: Use when a user asks to turn a vague or incomplete software-development request into a precise prompt, clarify missing requirements interactively, or revise a draft during prompt-cycle. Not for executing the engineering task.
---

## GitHub Copilot compatibility

Use this skill in GitHub Copilot desktop, CLI, VS Code or JetBrains agent mode. Use only tools actually available in the current client. To invoke a related skill, discover and read its installed SKILL.md; its name alone is not an executable tool call. Resolve personal skills relative to ~/.copilot/skills and project skills from supported repository skill directories. Prefer the client-provided installed skill location when available. Use a native question tool when available; otherwise ask in chat and wait for the answer. Never infer an answer or approval from silence. On Windows use py and py -m for Python commands unless the project explicitly requires another interpreter.


# Prompt Builder

Turn the user's intent into a detailed, testable prompt. For material missing
decisions, define focused questions and useful answer options instead of choosing
requirements for the user. Reply in the user's language.

## Entry point

- **Direct invocation:** continue through the installed `prompt-cycle` workflow
  in this same chat, passing the original request and existing clarifications.
  The user does not need to invoke another skill manually. If `prompt-cycle` is
  unavailable, explain the missing dependency and stop without approving a prompt.
- **Called by `prompt-cycle`:** perform only drafting, clarification planning or
  the requested revision, and return control to that workflow. Do not re-enter
  `prompt-cycle` recursively.
- `prompt-quality-gate` still owns approval. A direct invocation is not permission
  to skip validation or start implementation.

## Boundary

Work only with the request and already-available conversation context. Skill
instruction files and clarification tools are permitted within this workflow.
Do not inspect or modify project files, run project commands or tests, search the web,
invoke coding agents, or implement anything. Do not invent repository facts or
business requirements.

## Inputs

- `request`: original user's request, unchanged.
- `clarifications`: any answers the user has already supplied.
- `validator_feedback`: optional concrete blockers from the latest gate review.

## Procedure

1. Preserve the user's intent and desired mode (analyze / design / implement / review / refactor); do not silently turn an analysis into an implementation.
2. Determine intended outcome, observable behavior, boundaries, and acceptance criteria. Separate material user-owned decisions from facts that a future executing agent can discover in the repository.
3. For unknown repository facts, direct the **future executing agent** to inspect applicable `AGENTS.md`, relevant code and patterns. Do not inspect them now.
4. If validator feedback is present, revise **only** what the feedback identifies; do not silently choose the user's unresolved business or security decisions.
5. If a material decision is missing, prepare the clarification plan below. When the request is already sufficient, draft immediately without a questionnaire.
6. Produce a detailed, self-contained prompt, not a minimal one: mode and priority, objective, expected behavior, scope (in/out), constraints and compatibility, security/data/operational requirements (or "N/A: reason"), repository-inspection steps for the future agent, and concrete acceptance criteria. Avoid process instructions already contained in skills and `AGENTS.md`.
7. Aim for 10/10 on the `prompt-quality-gate` rubric. Plan questions not only for blockers but for every answerable gap that would keep the score below 10 (including mode and priority when missing). When a gap stays unanswered, list it under "Assumptions" with reason, risk and how to verify it; never turn a critical security, destructive or business decision into an assumption.
8. Never label the prompt approved; approval is owned by the separate validator.

## Clarification plan

Ask about a missing decision when different answers would materially change
the requested outcome, scope, acceptance criteria, security or data policy, or
when the answer is the only thing keeping the prompt below 10/10 (mode, priority,
acceptance threshold). Use
prior answers and explicit delegations; do not ask the user to repeat them.
Class names, file paths, framework details and existing implementation patterns
belong to future repository inspection, not this questionnaire.

For each blocking decision, define a self-contained question, why the answer
matters, and 2-3 concrete, mutually exclusive options when the choices are known.
Explain each option's consequence briefly. Recommend the first option only when
the recommendation has a defensible basis; leave actual selection to the user.
For unknown identifiers, policy values or requirements that cannot be enumerated,
use a free-text question rather than inventing choices.

Return these questions to `prompt-cycle`, which validates the blocker and presents
the interactive form. Keep each round focused: prefer one question, at most three
independent decisions. Do not display the same question in both a form and prose.

The controller presents the validated questions using a native question tool when available, otherwise in chat. Stop dependent drafting and approval until the user replies. Keep pending questions across turns; silence and preselected choices are not answers. Never request passwords, tokens or private keys.

## Output

```
Draft prompt:
<copy-ready request>

Unresolved decisions:
<only user-owned questions that block approval or keep the score below 10/10; otherwise "None">

Assumptions (only if the user skipped or could not answer):
<assumption | reason | risk | how to verify; otherwise "None">
Clarification plan:
<for each unresolved decision: question, why it matters, and options with consequences;
otherwise "None">
```

Return this result to the controlling `prompt-cycle` workflow for mandatory
validation and, when necessary, an interactive clarification round. After a user
reply, update the draft using that answer and existing decisions, then return it
for validation again. Never start execution.
