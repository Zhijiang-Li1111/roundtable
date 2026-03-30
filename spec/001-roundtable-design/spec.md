# Roundtable — Adversarial Multi-Agent Discussion Plugin

## Overview

A Claude Code plugin that spawns multiple agents with distinct perspectives to debate a topic through structured adversarial discussion. The goal is to expose hidden risks, challenge assumptions, and find root causes that single-agent analysis would miss.

## Problem

When Claude analyzes a bug, evaluates an architecture, or makes a technical decision alone, it tends toward confirmation bias — it picks a direction early and rationalizes it. Complex problems need multiple conflicting perspectives to surface risks and blind spots. A single agent saying "I think X" is less reliable than three agents attacking each other's reasoning until only the defensible conclusions survive.

## Core Concept

A **moderator agent** receives the topic, selects appropriate debater roles based on the scenario, runs structured rounds of adversarial discussion between debater agents, and produces a conclusion report when consensus is reached or the user decides to stop.

## Use Cases

1. **Complex bug root cause analysis** — bug already happened, need to find the real root cause, not the obvious one
2. **Architecture/design risk assessment** — solution not yet built, need to expose risks before committing
3. **Technical decision analysis** — choose between approach A vs B vs C with full consideration of trade-offs

## Design

### Components

#### 1. Moderator Agent

The orchestrator. Responsibilities:
- Receive the topic and context from the user
- Classify the scenario type (bug analysis / risk assessment / decision analysis)
- Select 3-4 debater roles appropriate for the scenario (from a role library)
- Run discussion rounds: present each debater's argument to the others for rebuttal
- Track which points have consensus vs active disagreement
- Pause and escalate to the user when major disagreements need more context
- Decide when discussion is sufficient (all key points have consensus or clearly marked disagreement)
- Compile the conclusion report

The moderator does NOT have an opinion. It facilitates, tracks, and summarizes. It does not take sides.

#### 2. Debater Agents

Each debater has a **perspective** (how they see the problem) and a **mandate** (what they must defend or attack). They:
- Argue from their assigned perspective
- Challenge other debaters' claims with evidence (code, logs, docs — not just opinions)
- Concede when proven wrong (no stubbornness for its own sake)
- Escalate uncertainty: "I don't have enough information to argue this point" → triggers user input

Debaters must read actual code/files to support their arguments. Assertions without evidence are flagged by the moderator.

#### 3. Role Library

Default roles selected by scenario type:

**Bug Root Cause Analysis:**
- **Symptom Skeptic** — challenges the obvious explanation. "Are you sure that's the real cause, not just a symptom?" Mandate: attack surface-level explanations.
- **System Thinker** — looks at interactions, race conditions, state dependencies. "What else touches this code path?" Mandate: find hidden coupling.
- **History Detective** — traces git history, recent changes, past incidents. "What changed recently that could cause this?" Mandate: find the trigger.
- **Devil's Advocate** — argues for the least popular theory. "What if it's actually a data problem, not a code problem?" Mandate: force consideration of unlikely causes.

**Architecture/Design Risk Assessment:**
- **Pessimist** — assumes everything will fail. "What happens when this service is down?" Mandate: find failure modes.
- **Scale Challenger** — questions whether the design holds under load. "This works for 100 users, what about 100K?" Mandate: find scalability limits.
- **Simplicity Advocate** — argues the design is over-engineered. "Do we actually need this abstraction?" Mandate: attack unnecessary complexity.
- **Integration Realist** — focuses on how this fits with existing systems. "The current auth system doesn't support this pattern." Mandate: find integration friction.

**Technical Decision Analysis:**
- **Advocate A** — argues for option A with full conviction, finds every advantage
- **Advocate B** — argues for option B with full conviction, finds every advantage
- **Cross-Examiner** — challenges both advocates. "You said X is fast, show me the benchmark." Mandate: destroy weak arguments from both sides.
- **Long-Term Thinker** — evaluates 6-12 month implications. "Option A is faster now, but what happens when requirements change?" Mandate: find future regret.

**Users can add custom roles** by describing the perspective and mandate. The moderator incorporates them into the discussion.

### Implementation: Team Mode vs Subagent Fallback

Two runtime modes, same discussion logic:

**Team Mode (preferred, when Claude Code Team is available):**

Debaters are spawned as team members. They can see each other's messages directly and respond in real-time. The moderator is the team coordinator. The user can SendMessage to any debater or the moderator at any time. This produces natural, flowing debate.

```
Moderator (coordinator)
├── Debater A (team member) ←→ sees all messages, responds directly
├── Debater B (team member) ←→
├── Debater C (team member) ←→
└── User (SendMessage to anyone)
```

**Subagent Fallback (when Team is not available):**

Debaters are spawned as subagents. The main agent acts as the moderator and **message relay**. The moderator does not alter, summarize, or interpret any debater's message — it forwards verbatim. The moderator's only jobs are:
- Forward each debater's full response to the other debaters
- Track consensus/disagreement state
- Manage turn order and round progression
- Pause for user input when needed

This is critical: the moderator must not filter or soften arguments. The raw adversarial tension is the point.

### Discussion Flow

```
User provides topic + context
    ↓
Moderator classifies scenario, selects roles
    ↓
Moderator presents role selection to user (user can add/remove/modify roles)
    ↓
Round 1: Each debater states initial position (with evidence from codebase)
    ↓
Round 2+: Each debater rebuts others' positions (with evidence)
    ↓
[User can interject at any time: add context, ask questions, redirect]
    ↓
[Moderator pauses at major disagreements: asks user for more context or judgment]
    ↓
Consensus check: moderator identifies which points have agreement vs active dispute
    ↓
If all key points resolved OR user says "enough" → Conclusion report
If major disputes remain → Another round (or user decides)
```

### Conclusion Report Format

```markdown
## Roundtable Conclusion: [Topic]

### Participants
- [Role]: [one-line perspective description]

### Consensus
- [Point 1]: [what all debaters agreed on, and why]
- [Point 2]: ...

### Unresolved Disagreements
- [Point]: [Role A] argues [X] because [evidence]. [Role B] argues [Y] because [evidence].
  The disagreement matters because [impact].

### Risks Identified
- [Risk 1]: identified by [Role], severity [high/medium/low], [mitigation suggestion]
- [Risk 2]: ...

### Recommended Actions
- [Action 1]: [what to do, based on the consensus and risk analysis]
- [Action 2]: ...

### Evidence References
- [file:line] — cited by [Role] to support [point]
```

### Integration with ironflow (when both installed)

The roundtable can be triggered naturally from ironflow's workflow:

- **During spec-first**: when the user or Claude is uncertain about a design approach → "Let's roundtable this design decision"
- **During systematic-debugging**: when root cause is unclear after Phase 2 → "Let's roundtable the possible causes"
- **During receiving-review**: when review feedback contains fundamental disagreements → "Let's roundtable this before deciding"

ironflow does not depend on roundtable, and roundtable does not depend on ironflow. When both are installed, ironflow skills can suggest invoking the roundtable skill, and the roundtable conclusion report can feed back into ironflow's spec or debugging flow.

The integration is through **skill invocation** — ironflow skills say "consider invoking roundtable if..." and roundtable's conclusion report is structured to be consumable by ironflow's next step (spec document, debugging hypothesis, or review response).

## Acceptance Criteria

1. User can trigger a roundtable discussion by describing a topic
2. Moderator automatically selects appropriate roles based on scenario type
3. User can add, remove, or modify roles before discussion starts
4. Debaters argue with evidence from the codebase, not just opinions
5. User can interject at any time during the discussion
6. Moderator pauses at major disagreements to ask user for input
7. Discussion ends when consensus is reached on key points or user says stop
8. Conclusion report follows the defined format with consensus, disagreements, risks, and actions
9. Works standalone without ironflow
10. When ironflow is installed, can be invoked from spec-first, systematic-debugging, or receiving-review flows
