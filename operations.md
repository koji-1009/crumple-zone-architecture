# Crumple Zone Architecture — Operations

How to deploy CRZ in AI-driven development workflows.

## Document Roles

| Document                        | Reader               | When               |
| ------------------------------- | -------------------- | ------------------ |
| skill/crz.md                    | Implementation agent | During coding      |
| architecture.md + extensions.md | Review agent         | During code review |
| Diff                            | Human operator       | Final gate         |

The skill prompt gives the agent implementation rules. The architecture and extensions documents give the reviewer the reasoning behind those rules.

## Two-Agent Separation

Implementation and review should be separate agents. A single agent writing and reviewing its own code is subject to confirmation bias — it justifies layers it created. Separation ensures the reviewer evaluates structure from principles, not from implementation momentum.

* Implementation agent reads skill/crz.md and writes code
* Review agent reads architecture.md and extensions.md, then evaluates the diff
* Human operator reviews the final diff as the last gate

## Treating CRZ as a Compiler

CRZ is not a set of suggestions — it is a specification checker. Like a compiler, it does not ask "does this work?" but "does this conform to the specification?" Ignoring a CRZ rule because the code "works right now" is the same structural mistake as ignoring a compiler warning because the tests pass.

Implementation agents exhibit a consistent bias: "I know this, therefore it is already handled." Knowledge and execution are not the same. The Post-Application Review exists to close this gap — execute every check mechanically, in order. Skipping a check because a previous check seemed to cover it allows bias to propagate.

## Skill Prompt Authoring

To translate CRZ into a framework-specific skill prompt:

1. Map the reliability layers to framework constructs
   * e.g., Astro: `.astro` component → island, Next.js: server component → client component
2. Define mutation patterns using the framework's conventions
   * e.g., Astro Actions, Next.js Server Actions
3. Specify canonical source placement with framework-specific APIs
4. Include the Post-Application Review checklist

## Scope

See [extensions.md Section 1](extensions.md) for where CRZ applies, where extension patterns are needed, and where a different architecture is required.
