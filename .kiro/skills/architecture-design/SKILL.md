---
name: architecture-design
description: Analysis, evaluation, and design of software architectures and system design. Use when discussing architecture, evaluating patterns, designing a system, or making structural decisions about the codebase.
---

# Architecture and System Design

## Process

### 1. Understanding the context
Before proposing any solution, clarify:
- Non-negotiable constraints (technological, team, time)
- Scalability requirements: real vs. hypothetical
- Existing stack and migration cost
- Team skills: don't design something they can't maintain

### 2. Trade-off analysis
For each significant architectural decision:
- Present at least 2 alternatives
- Make trade-offs explicit for each (complexity, maintainability, scalability, cost)
- Distinguish between reversible and irreversible decisions
- Be more cautious and detailed on irreversible decisions

### 3. Evaluation
Challenge the proposed architecture on:
- **Simplicity**: is there a simpler solution that meets the real requirements?
- **Bottlenecks**: where are the failure points and critical dependencies?
- **Evolvability**: how easy is it to modify when requirements change?
- **Observability**: is the system monitorable and debuggable in production?

### 4. Decision and documentation
- Document the reasoning behind the choice, not just the choice
- Record discarded alternatives and why
- Identify the assumptions the architecture is based on

## Useful patterns
- Prefer composition over inheritance
- Design for failure, not for success
- Postpone irreversible decisions until you have more information
- Don't optimize prematurely