---
name: software-design-process
description: Software design thinking framework for HA app development. Use when planning a new feature, designing system architecture, decomposing work, or reviewing a design before implementation. Provides a 7-step process (define, UX, technical, testing, plan, ripple effects, broader context) and coding tenets (functions over classes, separate creation from use, type system over strings) that apply across backend and frontend.
---

# Software Design Process

A thinking framework for designing software that's easy to work on. Adapted from Arjan Codes' 7-step design guide. This is a *process* skill — the language-specific reference skills ([[python-backend]], [[react-frontend]], [[ha-app-packaging]]) cover the *how*; this skill covers the *what to think about, and in what order*.

Apply these steps when starting a new feature or when an existing design feels stuck. You don't need to produce a separate document for every step — most outputs go into the plan, PR description, or design discussion. The point is to **not skip the thinking**.

## 1. Define what you're building

Before any code or schema:

- **What is the application or feature?** One sentence.
- **Who is it for?** Specific users, not "everyone."
- **What problem does it solve?** And how do users solve it today?
- **How will it work?** A short narrative of the main flow.
- **What are the main concepts and how do they relate?** Sketch the domain — entities, relationships, lifecycles.

**Zoom out, then zoom in.** First think broadly about what the feature entails (concepts, edges, integrations). Then prioritize: what's the smallest slice that's actually useful (MVP)? What can be delayed? What's the cheapest thing to build first to learn?

**Distill the model.** Removing things is the best refactoring. Ask explicitly: *what should we NOT build?* A simpler model now beats a clever-but-broad one later.

**Prototype in parallel with design.** Don't perfect the design then start coding — interleave. Early implementation surfaces technical limits and informs the model.

## 2. Design the user experience

- **User stories** — happy flow first, then alternative flows (errors, empty states, edge cases).
- **Impact on existing UI structure** — does this change navigation, menus, routing?
- **Wireframes or mockups** — even rough ones, for anything non-trivial.

**Build for the user, not for yourself.** Cautionary pattern: shipping a "very generic powerful" solution because it was satisfying to design — and nobody uses it. **Validate the user need before building.** If you can't name a specific user and the problem they have today, you're probably designing for yourself.

UI changes are expensive — every change means doc updates, user communication, and a chance for regressions. Cost that in before adding surface area.

## 3. Understand the technical needs

- **Storage** — new tables/fields, indexes, migrations.
- **Algorithms or libraries** — what's load-bearing? Any third-party deps?
- **Design** — main modules, types, and how they interact. Use a Mermaid diagram for non-trivial flows (rendered by GitHub and VS Code).
- **Edge cases to handle explicitly** — network failure, partial writes, auth expiry, missing config, malformed input.

### Coding tenets

These apply across Python and JavaScript/TypeScript:

**(a) Prefer functions over classes for business logic.** Use classes where the framework or domain demands them (Pydantic models, SQLAlchemy entities, React error boundaries) — but write *service logic* and *transformations* as plain functions. If you're passing the same arguments through a chain of methods, that's a sign you needed a function, not an object.

**(b) Keep things small and simple.** Small functions, small modules, few instance variables. Easier to read, easier to test.

**(c) Separate creating the thing from using the thing.** Don't construct a dependency inside the function that uses it — pass it in. This makes code testable without mocks, and decouples lifetime from usage.

```python
# Avoid — constructor hidden inside usage; hard to test
def fetch_users():
    client = HttpClient(API_URL, timeout=30)
    return client.get("/users")

# Prefer — caller controls construction
def fetch_users(client: HttpClient):
    return client.get("/users")
```

**(d) Use abstraction to remove dependencies.** Python `Protocol` (or `ABC`) and TypeScript interfaces let callers depend on a shape, not a concrete class. Use sparingly — only where you have a real second implementation or a real testing need.

**(e) Use the type system instead of strings for closed sets.** If a parameter has a known set of valid values (roles, statuses, modes), use an `Enum` (Python) or a literal union type (TypeScript). The compiler/linter rejects bad values before they reach runtime — fewer edge cases to write code and tests for.

```python
# Avoid — string typo becomes a runtime bug
def get_users(role: str): ...
get_users("amdin")  # passes type check, fails at runtime

# Prefer — Enum closes the set
class Role(str, Enum):
    ADMIN = "admin"
    USER = "user"

def get_users(role: Role): ...
```

## 4. Plan testing and security

- **Coverage** — focus on critical paths and security-sensitive code (auth, data access, payment-like flows). Don't chase a blanket percentage; coverage is a measurement, not a goal.
- **Test kinds** — unit (logic), integration (with real-ish DB), end-to-end where the flow crosses the network or the UI.
- **Side effects** — for a new feature, what existing behavior could it break? Where are the seams?
- **Security checks** — input validation at boundaries, auth on every protected route, secrets never logged, edge cases handled (expired tokens, missing claims, unicode, large payloads).

**Adopt a tester's mindset.** Assume every line of code potentially breaks. Designing *for testability* (tenets c and d above) is what makes test coverage feasible without heroic mocking.

**Don't get attached to code.** Treat early implementation as throwaway — that's the cost of learning what to build.

## 5. Plan the work

- **Estimate** — rough time per step. Beginners under-estimate; the more risk factors, the more buffer.
- **Milestones and order** — what's the first slice that can ship/be tested?
- **Migration scripts** — both forward and rollback paths for any schema change.
- **Risk factors and alternatives** — what could derail this? What's plan B if a third-party integration doesn't work?
- **Definition of Done** — required parts vs. nice-to-have. Write this down before starting so scope creep is visible.

## 6. Identify ripple effects

The code is rarely the only thing that needs to change. Check:

- **Documentation** — `DOCS.md`, README, in-app help, `CHANGELOG.md`.
- **User communication** — release notes, migration warnings, deprecated-feature notices.
- **External systems** — webhooks consumers, n8n flows, monitoring dashboards, scheduled jobs, HA blueprints that depend on your API shape.
- **Operational concerns** — backup compatibility, log volume changes, resource usage on RPi-class hardware.

For HA apps specifically: ingress path changes, breaking config options, schema migrations during update — all are user-facing ripples.

## 7. Understand the broader context

- **Current design limitations** — what does this design *not* handle that a future version might need? Write it down so future-you has a starting point.
- **Possible future extensions** — anticipate without designing-for. "If we ever add X, here's the seam we'd extend."
- **Other constraints** — RPi hardware budget, HA Supervisor restart cadence, backup size, dependency licensing.
- **Moonshot ideas** — one or two "what if we could also…" thoughts. Keeps the design open and occasionally produces an epiphany during implementation.

## Output: what to actually produce

You don't need a separate artifact per step. For most features, the design output is:

- A short design note in the PR description or planning doc covering steps 1–3 in 2–3 paragraphs.
- A Mermaid diagram if the data flow is non-trivial.
- A risk + Definition of Done section (step 5) in the plan.
- A ripple-effects checklist (step 6) used as the PR review checklist.

For small changes (bug fix, copy change, one-route addition) you can skim or skip — but **always** check ripple effects (step 6) before merging.
