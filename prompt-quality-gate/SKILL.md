---
name: prompt-quality-gate
description: Review, score, clarify, and rewrite a vague or incomplete software-engineering request BEFORE repository exploration, technical design, code review, implementation, or tool execution. Use when the user asks to validate or improve a coding prompt, wants a strict preflight gate, or explicitly invokes prompt-quality-gate. This skill never performs the engineering task itself.
---

## GitHub Copilot compatibility

Use this skill in GitHub Copilot desktop, CLI, VS Code or JetBrains agent mode. Use only tools actually available in the current client. To invoke a related skill, discover and read its installed SKILL.md; its name alone is not an executable tool call. Resolve personal skills relative to ~/.copilot/skills and project skills from supported repository skill directories. Prefer the client-provided installed skill location when available. Use a native question tool when available; otherwise ask in chat and wait for the answer. Never infer an answer or approval from silence. On Windows use py and py -m for Python commands unless the project explicitly requires another interpreter.


# Prompt Quality Gate

Evaluate the **user's request**, not the codebase. Act as a requirements-quality gate before any engineering workflow. Reply in the user's language.

## Non-negotiable boundary

- **Do not inspect, search, read, or modify the repository**. Do not read `AGENTS.md`, browse project files, inspect diffs, run commands, use web search, invoke subagents, create plans based on code, or implement anything. Work only with the user request and context already present in the conversation.
- Use an available native question tool solely to resolve blocking ambiguities; otherwise ask in chat. When controlled by `prompt-cycle`, return blockers and proposed clarifications to the controller, which owns the form; do not open a second form. Never invoke another skill as part of this gate.
- **Never auto-proceed after PASS**, even when the user requested implementation in the original message. Output an approved prompt and stop. Only a subsequent, explicit user request (the "Lanzar" choice in the `prompt-cycle` launch form, or the handoff word `LANZAR`) may start an engineering task.
- If invoked along with an engineering-execution skill, run this gate alone first; leave execution for another turn. Do not claim this skill enforces a platform-level security barrier: it is a workflow instruction, not a sandbox permission system.

## Evaluation procedure

1. **Extract the intent:** identify the requested action (analyze, design, implement, review, refactor), the desired observable outcome, scope, available technical context, constraints, and validation requirements. Do not infer unstated business behavior.
2. **Assess proportionally:** score each dimension 0, 1, or 2 (total /10):
   - **Objective**: what problem or outcome is requested?
   - **Expected behavior**: what should change or remain unchanged?
   - **Scope**: what is in scope and, when important, out of scope?
   - **Verifiability**: what observable result or acceptance check will demonstrate success?
   - **Risk-sensitive requirements**: are essential security, data, API compatibility, migration or operational requirements specified, or explicitly delegated for investigation?
   The target is always **10/10**. A dimension earns 2 points only when the prompt states it **explicitly and in detail** (see the detail checklist below); 1 point when it is present but vague or partial; 0 when absent. A dimension that genuinely does not apply is stated as "N/A: <reason>" in the prompt and earns full points; silently omitting it does not. Do not demand filenames, class names, framework details or tests that GitHub Copilot can discover later; delegate them to explicit inspection steps instead.
   **Detail checklist for a 10/10 prompt:** objective and motivation; expected behavior (what changes and what must stay unchanged); scope (in and out); constraints and compatibility; concrete, observable acceptance criteria; security/data/operational requirements (or explicit N/A); **mode** (analyze / design / implement / review / refactor) and **priority** for `engineering-workflow`; instructions to inspect the real repository architecture and applicable `AGENTS.md`; no speculative design decisions.
3. **Identify material blockers:** vague or contradictory desired behavior; missing critical business rule; unclear destructive action or scope; unresolved security, authorization or data-integrity requirement; missing decision that would substantially alter the implementation. Missing repository facts that can be discovered later are **not** blockers.
4. **Decide** with three possible states. Never inflate the score and never invent requirements to reach 10/10:
   - `APROBADO` (PASS): score is **10/10** and no material blocker remains.
   - `APROBADO CON SUPOSICIONES`: score is **8-9/10**, no material blocker remains, and every gap below 10 is a non-critical, low-risk gap that the user was asked about interactively and skipped, declined or could not answer. Each gap becomes an explicit assumption (see output contract). This state is only reachable after the questions were offered; a first-pass score below 10 means `BLOCKED` so the controller can ask.
   - `BLOQUEADO` (BLOCKED): any material blocker (critical security, destructive, data-integrity or business decision is **never** replaced by an assumption), a score below 8, or a score below 10 whose gaps have not yet been put to the user.
   Treat the number as an operational heuristic, not a scientific quality measure.
5. **Refine:** improve wording without changing the user's intent. Put unknown repository details into a *future agent's inspection steps* (e.g., 'Find existing JWT validation conventions'), never invent implementation choices or unapproved business rules. Prefer an actionable, concise prompt over an enormous specification.
6. **Stop at the gate:** never start repository analysis, technical design, testing, coding, or a downstream workflow within this skill, regardless of PASS/BLOCKED.

## Interaction when BLOCKED

- Ask about **material blocking** decisions **and** about any non-critical gap that keeps the score below 10/10 (including a missing mode or priority), preferably one focused question; at most three per round. Avoid questions GitHub Copilot can answer from the repository later.
- For a direct gate invocation, use a permitted interactive tool for 1–3 short independent questions, with 2–3 mutually exclusive options when the choices are known. Put a justified recommendation first and explain concise tradeoffs. Follow the actual tool schema and let the client provide its custom-answer option. For unknown identifiers or values, use free text through a supported form or ordinary text.
- When awaiting a user answer, end the turn pending the reply and preserve the unanswered blocker. Do not duplicate the form in prose or validate a draft that depends on an unanswered question. Do not supply a default for a critical decision or treat a preselected option, silence or elapsed time as acceptance.
- Upon reply, re-evaluate the revised request. Repeat while a material blocker or an answerable gap below 10/10 remains. Stop asking when the user skips, declines or cannot answer, and record the unanswered gaps as assumptions rather than looping.
- If no information is supplied and a material blocker remains, report `BLOCKED`, show the exact blocker(s), ask the most important question and end the turn.

## Output contract

For **BLOCKED**:

```text
Estado: BLOQUEADO | Calidad: N/10
Motivo: <one or two concrete ambiguities preventing correct execution>
Pregunta: <single decision-oriented question; selectable options where useful>
No se ha examinado el repositorio ni ejecutado ninguna implementación.
```

When useful, show a short *provisional* prompt, clearly marked as not approved; never imply it is ready for execution.

For **PASS** (`APROBADO`, 10/10) or **PASS WITH ASSUMPTIONS** (`APROBADO CON SUPOSICIONES`, 8-9/10):

```text
Estado: APROBADO | Calidad: 10/10            (or: APROBADO CON SUPOSICIONES | Calidad: N/10)

Prompt aprobado:
Modo: <analyze | design | implement | review | refactor>. Prioridad: <existing patterns | test-driven | security and performance | custom>.
Objetivo: <objective and motivation>
Comportamiento esperado: <what changes and what must remain unchanged>
Alcance: <in scope / out of scope>
Restricciones y compatibilidad: <constraints, public contracts, compatibility>
Seguridad, datos y operación: <requirements, or "N/A: <reason>">
Inspección previa: <inspect the real repository architecture, applicable AGENTS.md and relevant patterns; list the facts to discover>
Criterios de aceptación: <concrete, observable checks and the validation to run>
Fuera de diseño especulativo: <no invented implementation or business decisions>

Suposiciones y delegaciones: <"Ninguna" for 10/10. Otherwise one item per assumption: suposición | motivo | riesgo | cómo verificarla. Also list repository facts delegated to inspection.>
Siguiente paso: elige una opción en el formulario de lanzamiento (o escribe LANZAR). No se ha analizado ni modificado el proyecto.
```

The launch form (Lanzar / No lanzar / free text) is owned by `prompt-cycle`; the word `LANZAR` (whole message, uppercase) is an equivalent explicit execution request. The gate itself still never invokes `engineering-workflow` or shows the form. If the skill is not installed, say to start the next explicit GitHub Copilot task with the approved prompt instead.

## Quality examples

- `Arregla una errata en el README`: reaches **10/10** once the builder states mode, scope, acceptance (the typo is gone, nothing else changed) and N/A for security/data. Do not demand architecture diagrams or an elaborate test plan.
- `Cambia la autenticación JWT para admitir varios emisores`: **BLOCKED** if it is unclear which issuers are authorized and this would change a security boundary. Ask who is trusted or what authoritative source determines allowed issuers; do not invent this.
- `Investiga por qué una consulta es lenta y propón alternativas, sin modificar código`: reaches **10/10** as an analysis request, with future evidence-gathering delegated to the executor; do not invent performance targets.
