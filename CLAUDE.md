# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Roundtable is a Claude Code plugin that spawns multiple agents with distinct perspectives to debate a topic through structured adversarial discussion. The goal is to expose hidden risks, challenge assumptions, and find root causes that single-agent analysis would miss.

## Plugin Structure

```
.claude-plugin/
  plugin.json              # Plugin manifest
skills/
  roundtable/
    SKILL.md               # Core skill — moderator instructions, role library, discussion flow
```

The entire plugin is a single skill file. The moderator (Claude Code's main agent) orchestrates the discussion; each debater is an independent subagent with its own context window.

## Architecture

- **Moderator**: The main agent following SKILL.md instructions. Classifies scenarios, selects roles, manages rounds, tracks consensus, compiles conclusion reports. Has no opinion — only facilitates.
- **Debaters**: Each spawned as an independent subagent via the Agent tool. Separate context windows ensure genuine perspective diversity (same-context role-play produces convergent thinking). Must cite code/file evidence.
- **Role Library**: 12 predefined roles across 3 scenario types (bug analysis, risk assessment, decision analysis), inline in SKILL.md. Users can add custom roles.

## Key Design Decisions

- **Subagent mode only** (no Team Mode) — full control over discussion flow, fresh context window per debater per round ensures genuine perspective isolation, no platform stability risk from newer APIs.
- **Zero custom tools** — all operations use Claude Code built-in tools (Read, Grep, Agent, etc.).
- **Pressure-driven convergence** — the goal is not to run N rounds, but to force debaters through adversarial pressure until only defensible conclusions survive. Moderator ends the discussion (with user confirmation each round) when positions have been tested and either converged or clearly irreconcilable — not when a round counter expires. 4 rounds is a safety cap, not a target.
- **Moderator recap each round** — structured summary of consensus/disputes at top of each debater prompt mitigates context drift.
- **Convergence monitoring** — same-model agents share training biases and tend toward premature agreement; moderator flags suspiciously fast consensus. Genuine convergence (positions tested under pressure and found defensible) is the signal to conclude.

## Integration

When the ironflow plugin is also installed, roundtable can be invoked from ironflow's spec-first, systematic-debugging, or receiving-review flows. Neither plugin depends on the other.
