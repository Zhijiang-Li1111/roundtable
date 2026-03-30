# Roundtable

A Claude Code plugin that spawns multiple agents with adversarial perspectives to debate a topic through structured discussion.

## What it does

When you face a complex technical problem — unclear root cause, risky architecture decision, or a tough trade-off — a single agent tends to pick a direction and rationalize it. Roundtable fixes this by spawning 3-4 independent agents, each with a conflicting mandate, to attack each other's reasoning until only defensible conclusions survive.

## Use cases

- **Bug root cause analysis** — "The obvious fix didn't work. What's the real cause?"
- **Architecture risk assessment** — "What could go wrong with this design?"
- **Technical decision analysis** — "Should we go with approach A or B?"

## How it works

1. You describe the problem
2. A moderator classifies the scenario and selects debater roles (you can adjust)
3. Each debater is spawned as an independent subagent with its own context window
4. Debaters argue with evidence from your codebase (files, line numbers, git history)
5. Multiple rounds of rebuttal, with you able to interject at any time
6. A structured conclusion report: consensus, unresolved disagreements, risks, recommended actions

## Installation

Clone and register as a Claude Code plugin:

```bash
git clone https://github.com/Zhijiang-Li1111/roundtable.git
```

Then add it to your Claude Code plugins directory or reference it in your project settings.

## Triggering

The skill activates when you say things like:
- "Let's roundtable this"
- "What am I missing?"
- "What could go wrong with this design?"
- "Devil's advocate this approach"
- "Pros and cons of X vs Y"

## Design decisions

| Decision | Rationale |
|----------|-----------|
| Subagent mode (not Team Mode) | ~3x cheaper token cost, full control over discussion flow |
| Independent context per debater | Same-context role-play produces fake diversity due to attention mechanism blending |
| Max 4 rounds | Beyond 4, arguments repeat rather than deepen |
| Moderator recap each round | Mitigates context drift in multi-turn discussions |
| Convergence monitoring | Same-model agents share training biases; flags suspiciously fast agreement |
| Zero custom tools | Claude Code built-in tools (Read, Grep, Agent) suffice |

## Integration with ironflow

When both plugins are installed, roundtable can be invoked from ironflow's spec-first, systematic-debugging, or receiving-review flows. Neither depends on the other.

## License

MIT
