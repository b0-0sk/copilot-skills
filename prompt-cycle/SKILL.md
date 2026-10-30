---
name: prompt-cycle
description: Use when a user invokes prompt-cycle to prepare a software-development prompt, or prompt-builder routes a direct invocation into the full refinement and validation workflow. Not for executing the engineering task; execution starts only when the user picks "Lanzar" in the launch form or sends LANZAR.
---

## GitHub Copilot compatibility

Use this skill in GitHub Copilot desktop, CLI, VS Code or JetBrains agent mode. Use only tools actually available in the current client. To invoke a related skill, discover and read its installed SKILL.md; its name alone is not an executable tool call. Resolve personal skills relative to ~/.copilot/skills and project skills from supported repository skill directories. Prefer the client-provided installed skill location when available. Use a native question tool when available; otherwise ask in chat and wait for the answer. Never infer an answer or approval from silence. On Windows use py and py -m for Python commands unless the project explicitly requires another interpreter.


# Prompt Cycle — drafting + mandatory quality gate

## Scope and invariants

This is the **controller** for prompt preparation, entered explicitly or through
a direct `prompt-builder` invocation. It owns workflow state and user interaction,
not the drafting or validation policy. Keep the workflow in the current chat;
do not require the user to repeat the request under another skill name.

- Do **not** inspect project source, project `AGENTS.md`, diffs, local project documents, or any repo files. Do not execute application tools, edit code, launch external agents, run tests or build, or begin technical solution design. You may consult the two skill instruction files needed for this workflow; those are not project context.
- Use the installed **prompt-builder** skill's instructions for drafting/revisions and the installed **prompt-quality-gate** skill's instructions for validation. Read their full `SKILL.md` from their discovered installed skill paths, not by assuming absolute locations. If either is missing, report which one is missing and STOP; never skip validation.
- Treat these as two **logical stages with separate responsibilities**, coordinated in the current GitHub Copilot task. Skill discovery/selection is model-guided, not an operating-system-level function call; merely mentioning a skill does not guarantee a second autonomous skill invocation.
- Do not start `engineering-workflow` or any implementation after approval. Return the approved prompt and wait for a **new, explicit request** from the user. The user's "Lanzar" choice in the launch form, or the handoff word `LANZAR` (see "Launch decision"), is that explicit request.
- Honor more restrictive user/system/developer instructions and any tool limitations.

## State machine

```
INPUT -> BUILDER -> VALIDATOR
                    |     |
                 BLOCKED  PASS
                    |     |
              repairable? APPROVED -> OUTPUT -> LAUNCH FORM
                |     |                         |      |       |
               yes    no (user-only facts)   Lanzar  No lanzar  free text
                |      |                       |      |       |
             BUILDER  ASK USER -> WAIT   engineering- STOP   BUILDER (revalidate)
                ^                 |       workflow             |
                +--- after answer-+                            +-> VALIDATOR -> ...
```

### 1. INPUT

Collect the user's free-form request and any prior clarifications. Preserve their meaning verbatim as the source of truth. If the current turn is a follow-up answer, resume the pending blocker without restarting or dropping prior constraints.

### 2. BUILDER (always first)

Apply `prompt-builder` to create a detailed, self-contained draft (mode, priority, objective, expected behavior, scope, constraints, security/data, inspection steps, acceptance criteria). Do not treat a well-written draft as approved without passing it through the validator.
The builder may also return a clarification plan with proposed questions and
options. Validate whether these are blockers **or** answerable gaps that keep the score below 10/10 before presenting a form;
do not ask questions just because the builder proposed them.

### 3. VALIDATOR (mandatory after **every** draft or revision)

Apply `prompt-quality-gate` exactly, including its 0–10 rubric, critical blockers, the **10/10 target**, its three states (`APROBADO`, `APROBADO CON SUPOSICIONES`, `BLOQUEADO`) and boundaries. Record internally the actual issues identified, not only the score.

### 4. BLOCKED / below-10 handling

- If feedback is actionable using **already known facts** (unclear wording, missing noncontroversial acceptance criterion, redundant steps), return the feedback to `prompt-builder` for a targeted revision; rerun `prompt-quality-gate` on the revised draft.
- If feedback requires **unknown user-owned requirements** (security trust list, business rule, destructive scope, mode, priority, acceptance threshold), present the smallest validated clarification from the builder's plan, preferably one question. Define a focused question and meaningful options if no suitable plan exists. Use the interaction rules below; do not assume defaults or inflate the score. End the turn pending user response. On response, feed the answer to the builder, then always revalidate.
- **Keep asking until 10/10:** while the score is below 10 and an answerable gap remains, continue the interactive rounds (one round per turn). Stop asking only when the gate reaches 10/10, or the user skips, declines or cannot answer. Then request a final validation: if no material blocker remains, the gate returns `APROBADO CON SUPOSICIONES` and every unanswered gap is listed as an assumption (assumption, reason, risk, how to verify). A material blocker (critical security, destructive, data-integrity or business decision) is never converted into an assumption.
- Avoid circular rewriting: perform at most **3 automatic revisions per turn**. If still blocked, show the actual unresolved issues and ask for clarification or explain why a valid prompt cannot be formed; STOP. Do not change the threshold to force PASS.

### Interactive clarification

- Present the smallest validated clarification using an available native question tool, otherwise ask in chat.
- Prefer one self-contained question, at most three independent decisions per round. Offer meaningful options only when known.
- End the turn pending the answer. Do not duplicate questions, approve a provisional draft, or continue dependent work.
- Preserve the original request, previous decisions and pending questions across turns. Silence is not an answer.
- After the reply, resolve only answered decisions, revise through the builder and revalidate.

### 5. PASS handling

Once the validator has genuinely returned `APROBADO` (10/10) or `APROBADO CON SUPOSICIONES` (8-9/10), output the validator's real score, never an inflated one:

```
Estado: APROBADO | Calidad: 10/10            (or: APROBADO CON SUPOSICIONES | Calidad: <score>/10)
Iteraciones de refinamiento: <count>

Prompt final:
<approved detailed prompt, starting with "Modo: ... Prioridad: ...">

Suposiciones y delegaciones: <"Ninguna" for 10/10; otherwise one item per assumption: suposición | motivo | riesgo | cómo verificarla; plus facts delegated to repository inspection>
Siguiente paso: elige una opción en el formulario de lanzamiento (o escribe LANZAR).
```

Do not explore the project or execute the approved prompt before the user decides. If the
validator is BLOCKED, report its actual blocker(s) and present the clarification
once through the selected form, or in prose when no form is usable. Do not call
a prompt "aprobado" unless the complete validator criteria are satisfied.

## Launch decision

Only after the gate has genuinely approved the prompt (never with a BLOCKED prompt), and right after printing it, ask with `ask_user` (question in Spanish: "¿Qué quieres hacer con este prompt?") offering exactly these choices; the form's free-text entry covers the third path:

- **Lanzar**: start `engineering-workflow` immediately with the approved prompt. This choice is the user's explicit execution request. Read the installed `engineering-workflow` SKILL.md and run it with that prompt: its `Modo` and `Prioridad` are explicit, so no action/priority question is repeated. Print any assumptions in one line first. Do not revalidate.
- **No lanzar**: end without executing. The prompt stays available and can be launched later with `LANZAR`.
- **Free text** (what to change): feed the text to `prompt-builder` as a clarification or change request, revise the prompt, revalidate through `prompt-quality-gate` (10/10 target, questions or assumptions under the usual rules) and show the new approved prompt followed by the launch form again. If the text is not an actionable change, ask one focused question instead of guessing.

If no form tool is available, offer the same three options in chat and wait; silence is not approval. Never ask for the launch decision after a BLOCKED result.

## Handoff word: LANZAR (alternative)

After an approved prompt, the user's whole message `LANZAR` (uppercase, exact) is also an explicit execution request, equivalent to choosing "Lanzar". On receiving it, read the installed `engineering-workflow` SKILL.md and run it with the **last approved prompt** in the conversation, following the same rules as above. If the conversation has no approved prompt, execute nothing: say so and redirect to `prompt-cycle`. The word only counts as a whole message; the lowercase verb "lanzar" inside ordinary text does not trigger it.

## Example

User: `Quiero soportar varios issuers JWT`.

- Builder: draft asks future GitHub Copilot to inspect current auth architecture, identifies unknown trusted issuers.
- Validator: BLOCKED because trusted issuer set (or authority for determining it) is unspecified.
- Cycle: do **not** invent a list. Ask `¿Qué issuers deben considerarse de confianza o qué fuente autorizada lo define?` and STOP.
- When user responds: Builder revises; Validator reevaluates. Only then produce an approved prompt.

## Efficiency

Do not display every intermediate draft or narrate internal score fluctuations. Report only the result, essential blocking question(s), and approved prompt. Never scan the repository during prompt preparation. Avoid repeatedly revising equivalent wording just to raise a subjective score.
