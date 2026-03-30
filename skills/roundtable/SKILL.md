---
name: roundtable
description: "Spawn multiple agents with adversarial perspectives to debate a topic through structured discussion. Use when the user wants to analyze a complex bug with multiple perspectives, assess architecture/design risks before committing, or evaluate trade-offs between technical approaches. TRIGGER when: user says 'roundtable', 'debate this', 'let's get multiple perspectives', 'devil's advocate', or describes a problem where single-perspective analysis is insufficient — root cause is unclear, design has hidden risks, or a decision has non-obvious trade-offs. DO NOT TRIGGER when: user wants a simple code review, straightforward implementation, quick bug fix with obvious cause, or explicitly asks for a single opinion."
---

# Roundtable — Adversarial Multi-Agent Discussion

Surface hidden risks, challenge assumptions, and find root causes that single-agent analysis would miss — by having multiple agents with conflicting mandates debate the topic.

Why: A single agent tends toward confirmation bias — it picks a direction early and rationalizes it. Three agents attacking each other's reasoning until only defensible conclusions survive produces better analysis.

## Process

### 1. Receive topic and classify scenario

Read the user's topic. Classify into one of three scenario types:

- **Bug Root Cause Analysis** — a bug already happened, need the real root cause, not the obvious one
- **Architecture/Design Risk Assessment** — solution not yet built, need to expose risks before committing
- **Technical Decision Analysis** — choose between approaches with full consideration of trade-offs

If the scenario doesn't fit cleanly, pick the closest match. Tell the user which type was selected and why.

### 2. Select debater roles

Based on the scenario type, propose 3-4 roles from the role library below. Present the selection to the user:

> I've selected these debaters for this discussion:
> - **[Role Name]**: [perspective] — mandate: [mandate]
> - ...
>
> Want to adjust any roles or add a custom one?

Wait for user confirmation before proceeding. If the user wants a custom role, ask for its perspective and mandate in one sentence each.

### 3. Run Round 1 — Initial positions

Spawn each debater as an independent subagent using the Agent tool. Each debater gets its own prompt with:
- The topic and relevant context
- Its role, perspective, and mandate
- Instruction to read actual code/files as evidence — no unsupported assertions
- No knowledge of what other debaters will say

Use this debater prompt structure:

```
You are {role_name} in an adversarial roundtable discussion.

Perspective: {perspective}
Mandate: {mandate}

Topic: {topic}
Context: {context}

Argue your position on this topic from your assigned perspective. Your mandate is non-negotiable — even if another position seems reasonable, find the counter-argument from your angle.

Support every claim with evidence from the codebase — cite specific files, line numbers, code patterns, git history. Assertions without evidence are worthless in this discussion.

Keep your response focused and under 800 words.
```

Collect all responses. Present them to the user clearly labeled by role.

### 4. Run Round 2+ — Rebuttals

For each subsequent round, spawn each debater again as a fresh subagent. Include in their prompt:
- Their original role, perspective, and mandate
- All other debaters' arguments from the previous round (verbatim, do not summarize or soften)
- Instruction: "Rebut the points you disagree with. Concede only when the evidence is airtight — if you have any doubt, attack. Cite code/files."

Before each round, provide a structured recap at the top of each debater's prompt:

```
## Discussion State
- Round: {n}
- Current consensus: {list of points all debaters agreed on, if any}
- Active disputes: {list of points with ongoing disagreement}
- Focus for this round: {specific question or dispute to resolve}
```

This recap mitigates context drift across rounds (~15-20% improvement per research evidence).

### 5. Monitor for convergence

After each round, check: are the debaters converging too quickly? Same-model agents share training biases and tend toward premature agreement — this is a well-documented phenomenon.

If all debaters agree on a point within 1-2 rounds without strong evidence-based challenge, flag it:

> **Convergence warning:** All debaters agree on [point] — this may reflect shared model bias rather than genuine consensus. Consider whether this deserves more scrutiny.

Ask the user whether to accept the consensus or push for deeper examination.

### 6. Pause for user input between rounds

After presenting each round's results, provide a brief status update:

> **Round {n} summary:** [Role A] and [Role B] disagree on [X] — [one sentence on each position]. [Role C] raised [new concern]. [Consensus reached on Y].
>
> Continue to Round {n+1}, or want to add context / redirect the discussion?

Wait for user input. If the user provides new information, inject it into the next round's debater prompts as additional context.

If the user says "enough", "stop", "wrap up", or similar — immediately proceed to the conclusion report. Do not run another round.

### 7. Generate conclusion report

After the final round (or when the user stops the discussion), compile a structured report:

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

## Role Library

### Bug Root Cause Analysis

- **Symptom Skeptic** — Perspective: challenges the obvious explanation. Mandate: attack surface-level explanations until a deeper cause is found. "Are you sure that's the real cause, not just a symptom?"
- **System Thinker** — Perspective: looks at interactions, race conditions, state dependencies. Mandate: find hidden coupling. "What else touches this code path?"
- **History Detective** — Perspective: traces git history, recent changes, past incidents. Mandate: find the trigger. "What changed recently that could cause this?"
- **Devil's Advocate** — Perspective: argues for the least popular theory. Mandate: force consideration of unlikely causes. "What if it's actually a data problem, not a code problem?"

### Architecture/Design Risk Assessment

- **Pessimist** — Perspective: assumes everything will fail. Mandate: find failure modes. "What happens when this service is down?"
- **Scale Challenger** — Perspective: questions whether the design holds under load. Mandate: find scalability limits. "This works for 100 users, what about 100K?"
- **Simplicity Advocate** — Perspective: argues the design is over-engineered. Mandate: attack unnecessary complexity. "Do we actually need this abstraction?"
- **Integration Realist** — Perspective: focuses on how this fits with existing systems. Mandate: find integration friction. "The current auth system doesn't support this pattern."

### Technical Decision Analysis

- **Advocate A** — Perspective: argues for option A with full conviction. Mandate: find every advantage of A.
- **Advocate B** — Perspective: argues for option B with full conviction. Mandate: find every advantage of B.
- **Cross-Examiner** — Perspective: challenges both advocates. Mandate: destroy weak arguments from both sides. "You said X is fast, show me the benchmark."
- **Long-Term Thinker** — Perspective: evaluates 6-12 month implications. Mandate: find future regret. "Option A is faster now, but what happens when requirements change?"

## Constraints

- Maximum 4 rounds of discussion. If no convergence after 4 rounds, proceed to conclusion with unresolved disagreements clearly marked.
- Each debater must be spawned as an independent subagent (separate context window) — never simulate multiple roles in a single agent call. Independent contexts produce genuine perspective diversity; same-context role-play produces fake diversity.
- Forward debater arguments verbatim to other debaters. Do not alter, summarize, or soften any argument. The raw adversarial tension is the point.
- When a debater says "I don't have enough information to argue this point", pause and ask the user for that information.
