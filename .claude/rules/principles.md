# Development Principles

## Core Principles

- **KISS**: Prefer standard CRUD controllers and conventional routing. No abstractions until complexity demands it. If a junior developer can't understand it in 30 seconds, simplify it.
- **DRY is about knowledge, not code**: Every piece of knowledge has one authoritative representation. But three similar lines are better than a premature abstraction -- duplicate code is cheaper than the wrong abstraction.
- **YAGNI**: Implement only what is currently required. Don't add configuration options, feature flags, or patterns for hypothetical future needs. Start simple, extract later.
- **SRP**: Each class has one reason to change. A model handles persistence, a service handles business logic, a controller handles HTTP orchestration.
- **Dependency Inversion**: Inject collaborators via constructor for testability. High-level business logic should not depend on low-level modules.
- **Composition over inheritance**: Favor modules, concerns, and delegation over deep class hierarchies.
- **Skinny Everything**: Controllers orchestrate (delegate to services, render responses). Models persist (validations, associations, scopes, simple predicates). Services contain business logic. Views display markup with no logic.
- **Callbacks**: Only for data normalization (`before_validation :strip_whitespace`, `before_save :downcase_email`). Side effects (emails, API calls, job enqueuing, creating related records) always belong in services, never in callbacks.
- **No premature abstraction**: Don't create base classes, helpers, or utilities for one-time operations. Extract only when you have 5+ concrete implementations with identical structure.
- **Explicit over implicit**: Clear code wins over magic. Explicit service calls over hidden callbacks. Named methods over metaprogramming.

## The Ladder

Before writing production code, stop at the first rung that holds:

1. **Does this need to exist?** Speculative need, skip it and say so in one line.
2. **Already in this codebase?** A service, query object, concern, helper, or scope that already does it. Re-implementing what lives a few files over is the most common waste — query the knowledge graph to check (see `cli-tools.md`), do not guess.
3. **Rails or Ruby already does it?** `delegate`, `normalizes`, `store_accessor`, `enum`, built-in validators, `has_secure_password`, `Rails.cache`, `Object#presence`, `Hash#dig`, `Time.current`.
4. **Native platform feature covers it?** A database constraint (`null: false`, unique index, foreign key, check constraint) over an app-level guard. A Turbo Frame over a Stimulus controller. `<dialog>` or `<details>` over a custom modal.
5. **An installed dependency solves it?** Solid Queue over a hand-rolled scheduler, Solid Cache over a custom store, Active Storage variants over an image gem, a Pundit scope over hand-rolled filtering. Never add a gem for what a few lines do.
6. **Can it be one line?** One line.
7. **Only then:** the minimum code that works.

The ladder shortens the solution, never the reading. Understand the problem and trace the real flow first, then climb — the smallest change in the wrong place is a second bug, not laziness.

**The ladder does not apply to tests.** TDD is the workflow here: write the failing spec first, cover each state and edge case. "Would a test be overkill?" is not a rung.

## Fix the Cause, Not the Symptom

A bug report names a symptom. Before editing, find every caller of the method you are about to touch. One guard in the shared method is a smaller diff than a guard in each caller — and patching only the path the ticket names leaves every sibling caller broken.

## Never Simplify Away

Input validation at trust boundaries, error handling that prevents data loss, authentication and authorization checks, accessibility basics, or anything the user explicitly asked for. If the user wants the fuller version, build it and do not re-argue.

Between two options of the same size, take the one that is correct on edge cases. Writing less code never means picking the flimsier algorithm.

When a shortcut cuts a real corner with a known ceiling, mark it with a `# NOTE:` naming the ceiling and the upgrade path — `# NOTE: N+1 acceptable at current volume, batch when lists exceed ~100`. An unmarked shortcut is indistinguishable from a mistake.
