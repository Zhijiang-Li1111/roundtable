---
name: roundtable
description: "Spawn multiple agents with adversarial perspectives to debate a topic through structured discussion. Use when the user wants to analyze a complex bug with multiple perspectives, assess architecture/design risks before committing, or evaluate trade-offs between technical approaches. TRIGGER when: user says 'roundtable', 'debate this', 'multiple perspectives', 'devil's advocate', 'what am I missing', 'what could go wrong', 'pros and cons', 'stress test this design', 'challenge my assumption', or describes a problem where single-perspective analysis is insufficient. DO NOT TRIGGER when: user wants a straightforward implementation, a quick bug fix with obvious cause, or explicitly asks for a single opinion."
---

# Roundtable — Adversarial Multi-Agent Discussion

Why: A single agent tends toward confirmation bias — it picks a direction early and rationalizes it. Multiple independent agents attacking each other's reasoning until only defensible conclusions survive produces better analysis.

## Process

### 1. Receive topic and classify scenario

Read the user's topic. Classify into one of three scenario types:

- **Bug Root Cause Analysis** — a bug already happened, need the real root cause, not the obvious one
- **Architecture/Design Risk Assessment** — solution not yet built, need to expose risks before committing
- **Technical Decision Analysis** — choose between approaches with full consideration of trade-offs

If the scenario doesn't fit cleanly, pick the closest match. Tell the user which type was selected and why.

### 2. Select debater roles

Each scenario type has a default 3-role combination. Start with the default and add the 4th role when the topic has enough complexity to warrant it:

**Bug Root Cause Analysis** — default: Symptom Skeptic, System Thinker, History Detective. Add Devil's Advocate when the obvious explanation is too convenient.

**Architecture/Design Risk Assessment** — default: Pessimist, Scale Challenger, Simplicity Advocate. Add Integration Realist when the design touches multiple existing systems.

**Technical Decision Analysis** — default: Advocate A, Advocate B, Cross-Examiner. Add Long-Term Thinker when the decision has significant lock-in effects.

Present the selection to the user:

> I've selected these debaters for this discussion:
> - **[Role Name]**: [perspective] — mandate: [mandate]
> - ...
>
> Want to adjust any roles or add a custom one?

Wait for user confirmation before proceeding. If the user wants a custom role, ask for its perspective and mandate in one sentence each.

### 3. Run Round 1 — Initial positions

Spawn all debaters in parallel using the Agent tool — one independent subagent call per debater. Each debater gets its own prompt:

```
You are {role_name} in an adversarial roundtable discussion.

Perspective: {perspective}

Principles:
- Commit fully to your mandate: {mandate}. The roundtable only works when each participant holds their ground — otherwise it degenerates into groupthink.
- Support every claim with evidence from the codebase — cite specific files, line numbers, code patterns, git history. Unsupported assertions are easy to dismiss and won't advance the discussion.

Topic: {topic}
Context: {context}

Argue your position on this topic from your assigned perspective. Keep your response focused and under 800 words — this forces you to prioritize your strongest arguments.
```

After collecting all responses, perform a per-round check (see checklist below). Then present responses to the user, clearly labeled by role.

### 4. Run Round 2+ — Rebuttals

For each subsequent round, spawn all debaters again in parallel as fresh subagents. Use this rebuttal prompt:

```
You are {role_name} in Round {n} of an adversarial roundtable discussion.

Perspective: {perspective}

Principles:
- Commit fully to your mandate: {mandate}.
- Concede when the evidence genuinely compels you, but push back if you have counter-evidence or doubt.
- Cite code/files for every claim.

## Discussion State
- Round: {n}
- Current consensus: {list of points all debaters agreed on, if any}
- Active disputes: {list of points with ongoing disagreement}
- Focus for this round: {specific question or dispute to resolve}

## Previous Round Arguments
{verbatim arguments from all other debaters in the previous round — do not summarize or soften}

Rebut the points you disagree with. Introduce new evidence from the codebase if you can find it. Keep your response under 800 words.
```

The Discussion State recap at the top of each prompt helps debaters stay oriented as rounds accumulate — without it, later rounds tend to repeat earlier points or lose track of what's already been resolved.

After collecting responses, perform the per-round check, then present to the user.

### 5. Monitor for convergence

Same-model agents share training biases and tend toward premature agreement. When consensus forms quickly without rigorous evidence-based challenge, it's worth questioning whether it reflects genuine agreement or shared blind spots.

Flag suspicious consensus for the user:

> **Convergence warning:** All debaters agree on [point] — this may reflect shared model bias rather than genuine consensus. Consider whether this deserves more scrutiny.

Ask the user whether to accept the consensus or push for deeper examination.

### 6. Pause for user input between rounds

After presenting each round's results, provide a brief status update:

> **Round {n} summary:** [Role A] and [Role B] disagree on [X] — [one sentence on each position]. [Role C] raised [new concern]. [Consensus reached on Y].
>
> Continue to Round {n+1}, or want to add context / redirect the discussion?

Wait for user input. If the user provides new information, inject it into the next round's debater prompts as additional context.

If the user says "enough", "stop", "wrap up" — immediately proceed to the conclusion report.

### 7. Generate conclusion report

After the final round (or when the user stops), compile a structured report:

```markdown
## Roundtable Conclusion: [Topic]

### Participants
- [Role]: [one-line perspective description]

### Consensus
- [Point]: [what all debaters agreed on, and why]

### Unresolved Disagreements
- [Point]: [Role A] argues [X] because [evidence]. [Role B] argues [Y] because [evidence].
  The disagreement matters because [impact].

### Risks Identified
- [Risk]: identified by [Role], severity [high/medium/low], [mitigation suggestion]

### Recommended Actions
- [Action]: [what to do, based on the consensus and risk analysis]

### Evidence References
- [file:line] — cited by [Role] to support [point]
```

## Per-Round Checklist

After collecting each round's debater responses, verify before presenting to the user:

- All debaters responded (no silent subagent failures)
- Each response cites at least one specific file or line number
- Each response stays within its assigned mandate — a Pessimist finding failure modes, not praising the design; a Symptom Skeptic questioning obvious answers, not accepting them
- Round summary accurately reflects the key disagreements

If a debater's response lacks code citations or drifts from its mandate, re-spawn that debater with an additional reminder of the requirements. Allow one retry per debater per round — if it still drifts, flag it to the user as a signal that the perspective may not be strongly defensible for this topic.

## Gotchas

- **Fabricated evidence.** Debaters sometimes cite files or line numbers that don't exist. After Round 1, spot-check at least one code citation per debater. If a debater fabricated evidence, note it in the round summary and instruct it to provide real evidence in the next round.
- **Role drift.** A common failure: the Pessimist starts offering solutions instead of finding failure modes, or the Devil's Advocate agrees with the majority. The per-round checklist catches this — enforce it.
- **Summary distortion.** When forwarding arguments to the next round, forward them verbatim. Paraphrasing systematically weakens adversarial positions and creates false consensus. The raw tension is the point.

## Role Library

### Bug Root Cause Analysis

- **Symptom Skeptic** — Mandate: attack surface-level explanations until a deeper cause is found. "Are you sure that's the real cause, not just a symptom?"
- **System Thinker** — Mandate: find hidden coupling — interactions, race conditions, state dependencies. "What else touches this code path?"
- **History Detective** — Mandate: find the trigger via git history, recent changes, past incidents. "What changed recently that could cause this?"
- **Devil's Advocate** — Mandate: force consideration of unlikely causes. "What if it's actually a data problem, not a code problem?"

### Architecture/Design Risk Assessment

- **Pessimist** — Mandate: find failure modes. "What happens when this service is down?"
- **Scale Challenger** — Mandate: find scalability limits. "This works for 100 users, what about 100K?"
- **Simplicity Advocate** — Mandate: attack unnecessary complexity. "Do we actually need this abstraction?"
- **Integration Realist** — Mandate: find integration friction with existing systems. "The current auth system doesn't support this pattern."

### Technical Decision Analysis

- **Advocate A** — Mandate: argue for option A with full conviction, find every advantage.
- **Advocate B** — Mandate: argue for option B with full conviction, find every advantage.
- **Cross-Examiner** — Mandate: destroy weak arguments from both sides. "You said X is fast, show me the benchmark."
- **Long-Term Thinker** — Mandate: find future regret. "Option A is faster now, but what happens when requirements change?"

## Constraints

- Maximum 4 rounds. Beyond 4, arguments tend to repeat rather than deepen — diminishing returns. If no convergence, proceed to conclusion with disagreements clearly marked.
- Each debater is an independent subagent (separate context window). Same-context role-play produces surface-level diversity rather than genuine perspective differences, because the model's attention mechanism blends the viewpoints.
- Forward debater arguments verbatim between rounds. Paraphrasing weakens adversarial tension and leads to premature consensus (see Gotchas).
- When a debater says "I don't have enough information to argue this point", pause and ask the user for that information before continuing.
