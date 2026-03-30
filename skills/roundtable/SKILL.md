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

If the scenario doesn't fit cleanly, default to Technical Decision Analysis — it has the most flexible role set. Tell the user which type was selected and why.

### 2. Select debater roles

Each scenario type has a recommended 3-role combination. Start with these defaults and consider a 4th role when the topic warrants it:

**Bug Root Cause Analysis** — default: Symptom Skeptic, System Thinker, History Detective. Consider adding Devil's Advocate if the obvious explanation seems too convenient — the goal is to force deeper investigation.

**Architecture/Design Risk Assessment** — default: Pessimist, Scale Challenger, Simplicity Advocate. Consider adding Integration Realist if the design touches multiple existing systems — integration friction is often underestimated.

**Technical Decision Analysis** — default: Advocate A, Advocate B, Cross-Examiner. Consider adding Long-Term Thinker if the decision has significant lock-in effects — short-term wins can mask long-term regret.

Present the selection to the user:

> I've selected these debaters for this discussion:
> - **[Role Name]**: [perspective] — mandate: [mandate]
> - ...
>
> Want to adjust any roles or add a custom one?

Wait for user confirmation before proceeding. If the user wants a custom role, ask for its perspective and mandate in one sentence each.

### 3. Run Round 1 — Initial positions

Before spawning debaters, the moderator verifies ground truth. The principle: **verify existence before analyzing causality** — agents who start from unverified assumptions will search in the wrong direction and converge on the wrong answer. Extract the key entities from the topic (selectors, API endpoints, modules, config keys, interfaces) and verify in the current codebase that they still exist and behave as expected. If an entity is missing or has been replaced, search by structural role ("what renders this component now?") rather than by old name. Include verified facts in the Context field, clearly separated from unverified claims.

When constructing the Topic and Context fields, present raw facts in chronological order — do not label anything as "primary cause", "root cause", or "secondary symptom". The moderator's framing leaks into every agent's reasoning.

Each debater gets its own prompt:

```
You are {role_name}, one of {n} debaters in an adversarial roundtable discussion. {n-1} other debaters with conflicting mandates will read and challenge your arguments.

Perspective: {perspective}
Mandate: {mandate}

You have access to code exploration tools (Read, Grep, Glob) to find evidence in the codebase. Use them — do not cite files or line numbers from memory. Every claim should be backed by evidence you have verified.

Before theorizing about WHY something fails or is risky, first verify that your assumptions about WHAT EXISTS are true in the current codebase. Error messages tell you where a failure stopped, not why it happened — the cause may be upstream.

The discussion benefits from genuine disagreement. If you find yourself agreeing with the likely positions of other debaters, look harder for counter-evidence.

Topic: {topic}
Context: {context}

Argue your position. Keep your response under 800 words.
```

After collecting all responses, perform a per-round check (see checklist below). Then present responses to the user, clearly labeled by role.

### 4. Run Round 2+ — Rebuttals

For each subsequent round, spawn all debaters again in parallel as fresh subagents. Use this rebuttal prompt:

```
You are {role_name} in Round {n} of an adversarial roundtable discussion. {n-1} other debaters with conflicting mandates are challenging your position.

Perspective: {perspective}
Mandate: {mandate}

You have access to code exploration tools (Read, Grep, Glob). Use them to verify claims and find new evidence. Do not cite files from memory.

Before theorizing about WHY something fails or is risky, first verify that your assumptions about WHAT EXISTS are true in the current codebase.

Concede when the evidence genuinely compels you, but push back if you have counter-evidence or doubt.

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

Example of a well-written Discussion State:
```
- Current consensus: (1) The timeout is caused by the retry loop in api_client.py:142, not by network latency. (2) The fix should be at the caller level.
- Active disputes: Whether the retry count should be configurable or hardcoded.
- Focus: Pessimist claims configurability adds operational risk — Scale Challenger, respond with evidence.
```

After collecting responses, perform the per-round check, then present to the user.

### 5. Monitor for convergence

The goal of roundtable is not to run a fixed number of rounds — it is to apply adversarial pressure until positions either converge to defensible conclusions or are clearly irreconcilable. The moderator should end the discussion when:

- Debaters have been forced to concede weak points and the remaining positions are evidence-backed and tested under pressure — this is **genuine convergence**, the ideal outcome.
- Positions have stabilized with clear, well-evidenced disagreements that further rounds won't resolve — proceed to conclusion with disagreements marked.

If all debaters reach consensus in Round 1 without genuine adversarial pressure, this is almost certainly premature — trigger a convergence warning and run at least Round 2 as a pressure test before accepting it.

Same-model agents share training biases and tend toward premature agreement. When consensus forms quickly without evidence-based challenge:

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

After the final round (or when the user stops), compile a structured report. Adapt the sections to what the discussion actually produced — not every discussion needs every section. The default structure:

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

For Bug Root Cause Analysis, "Risks Identified" may be less relevant — replace with root cause and verification plan. For Technical Decision Analysis, "Recommended Actions" might be a clear recommendation or an honest "needs more data." Let the discussion content drive the report structure.

## Per-Round Checklist

After collecting each round's debater responses, verify before presenting to the user:

- All debaters responded (no silent subagent failures)
- Each response cites at least one specific file or line number
- Spot-check at least one code citation per debater — verify the file and line number actually exist. Debaters sometimes fabricate evidence.
- Each response stays within its assigned mandate — a Pessimist finding failure modes, not praising the design; a Symptom Skeptic questioning obvious answers, not accepting them
- At least one agent searched current product/source code (not just test code or logs)
- If all agents anchor on the same explanation, flag it and push for alternatives in the next round
- Round summary accurately reflects the key disagreements

If a debater fabricates evidence, lacks citations, or drifts from its mandate, re-spawn that debater with a reminder. Allow one retry per debater per round — if it still fails, flag it to the user.

## Gotchas

- **Fabricated evidence.** Debaters sometimes cite files or line numbers that don't exist. The per-round checklist includes spot-checking — enforce it. The debater prompts now instruct agents to use tools (Read, Grep, Glob) to verify before citing, which reduces but does not eliminate this risk.
- **Role drift.** A common failure: the Pessimist starts offering solutions instead of finding failure modes, or the Devil's Advocate agrees with the majority. The per-round checklist catches this — enforce it.
- **Summary distortion.** When forwarding the previous round's arguments to the next round, forward them verbatim. Do not paraphrase — paraphrasing systematically weakens adversarial positions and creates false consensus. The raw tension is the point. (Note: only the immediately previous round's arguments are forwarded verbatim; earlier rounds are represented through the Discussion State recap.)
- **Lexical archaeology trap.** Agents default to grepping keywords from the error message or problem description. This only finds things with lexical overlap — a replacement (new layout, renamed API, refactored module) has zero overlap with the old name and will never be found this way. The observation phase and Ground Truth Investigator role exist to counter this, but watch for it in all agents.

## Role Library

### Bug Root Cause Analysis

- **Symptom Skeptic** — Mandate: attack surface-level explanations until a deeper cause is found. "Are you sure that's the real cause, not just a symptom?"
- **System Thinker** — Mandate: find hidden coupling — interactions, race conditions, state dependencies. "What else touches this code path?"
- **History Detective** — Mandate: find the trigger via git history, recent changes, past incidents. "What changed recently that could cause this?"
- **Devil's Advocate** — Mandate: force consideration of unlikely causes. "What if it's actually a data problem, not a code problem?"
- **Ground Truth Investigator** — Mandate: ignore the error message, start from the current product code. "Does this selector/API/module still exist? If not, what replaced it?" Searches by structural role, not by string identity.

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

- Termination criteria: see Section 5. 4 rounds is a safety cap (beyond 4, arguments tend to repeat), not a target. Most discussions converge in 2-3 rounds if the pressure is real.
- Each debater is an independent subagent (separate context window). Same-context role-play produces surface-level diversity rather than genuine perspective differences, because the model's attention mechanism blends the viewpoints.
- Forward the previous round's debater arguments verbatim into the next round's prompts. Earlier rounds are captured through the Discussion State recap (consensus/disputes), not re-forwarded in full. This balances adversarial tension with token efficiency (see Gotchas).
- When a debater says "I don't have enough information to argue this point", pause and ask the user for that information before continuing.
