# Brief: <short task name>

The builder treats this brief as binding. It may add caution, never remove it.

**Tier:** light | standard | deep
(Deep is required for security, tenant isolation, payments and other money paths, data migrations and seeds, cross-module event flows, and design-gated UI.)

**Lane:** <absolute path to the lane's git worktree>
**Pinned base:** <commit SHA the work must sit on>

## Goal
One or two sentences on the behaviour that should exist when this is done.

## Scope
- In: <files, modules or endpoints the builder may change>
- Out: <things that look related but must not be touched>

## Read first
- <document>, section <n>
- <document>, section <n>
(Sections, not whole documents.)

## Acceptance
- <observable behaviour 1, written as a test>
- <observable behaviour 2>
- Core invariant needing a mutation-style proof, if any: <invariant, or "none">

## Gates
<commands to run, for example typecheck, lint, unit, integration, build>

## Extra rules for this task
<anything beyond the standing rules, or "none">

## Rulings
<decisions the lead has already made on open questions; add to this as questions come up>
