---
title: 'Engineering Philosophy'
description: 'How we think about building systems — principles that transcend any specific tech stack'
---

# Engineering Philosophy

These principles guide every engineering decision across all Breezy Bays Labs projects. They are deliberately tech-stack-agnostic — the specific tools change per project, but the thinking behind them stays the same.

---

## Schema-First: Data Describes Itself

Define the schema as the single source of truth. Derive everything else from it.

Every data structure that crosses a boundary — API contracts, configuration files, database models, form inputs — starts as a validated schema. Types are inferred from schemas, not declared separately. When multiple consumers need the same data, they read from one source.

**Four sub-principles:**

1. **Single source of truth.** One schema definition, many consumers. Never maintain parallel type definitions that can drift.
2. **Validate at boundaries.** Trust validated data internally. Catch errors at system edges — user input, external APIs, database results. Internal function calls between trusted layers don't need redundant validation.
3. **Generate, don't duplicate.** When multiple artifacts need the same data shape, generate them from the schema rather than writing them by hand.
4. **Data describes itself.** Every configuration entry carries metadata about what it is and how it should be used. Self-documenting structures over external documentation that rots.

**Decision test:** If data meets two or more of these criteria, it should be schema-driven: multiple consumers, user-facing, needs validation, extensible, cross-platform.

**Schema before storage.** The domain schema — what valid data looks like in your system — is defined before the database schema that stores it. Storage serves the domain model; the domain model does not contort to fit storage. When the two shapes diverge, the domain schema wins and the storage layer adapts. This ordering prevents a common trap: designing tables first and then forcing the UI to work around storage-shaped data.

---

## Contracts at Every Boundary

An API contract is the shape of data at a system boundary — defined before implementation begins. When two parts of a system communicate, both sides agree on what goes in, what comes out, and what can go wrong.

**Why contracts matter:** Without explicit contracts, each layer invents its own assumptions about data shape. The external API returns a field called `colorFamily`; the database column is `color_family_name`; the UI component expects `colorFamilyName`. Three representations of the same concept, drifting independently. Contracts make these translations explicit and locate them in one place.

### Three Levels of Contract

Every system has contracts at three levels, whether they're designed intentionally or discovered through bugs:

| Level | What it defines | Who controls it |
|-------|----------------|-----------------|
| **External** | The shape of data from outside your system — supplier APIs, webhook payloads, CSV imports, third-party SDKs | The external party (you adapt to them) |
| **Domain** | The shape of data inside your system — what a valid Customer, Quote, or Invoice looks like in your world | You (this is your source of truth) |
| **Surface** | The shape of data your own UI or consumers receive — server action inputs/outputs, route handler responses, event payloads | You (designed for your consumers) |

The external contract is given to you. The domain contract is designed by you. The surface contract is offered by you. All three may represent the same underlying concept — a "garment style" or a "customer" — but in different shapes optimized for different purposes.

### Outside-In, Then Inside-Out

When integrating with external systems, understand the external shape first. Not the documentation — the actual data. Then design inward.

**The sequence:**

1. **Inspect the external response.** What fields actually come back? What are the types, nullability, and edge cases? Where does the external API return empty strings instead of null? Where are fields named differently than you'd expect?
2. **Mirror it in a raw layer.** Store the external data exactly as received — no transformation, no opinions. This is your audit trail and your buffer against upstream changes.
3. **Design the domain contract.** Now, with full knowledge of what's available externally, define what your system needs. The domain schema is shaped by your user journeys and business rules, not by the external API's structure.
4. **Build the adapter.** The adapter is the translation layer between external shape and domain shape. All the messy mapping — field renames, null coercion, nested-to-flat transformations — lives here, in one place.

**Why this order matters:** If you design the domain schema first (inside-out only), you'll discover mismatches when you finally connect to the external system. Fields you assumed would exist don't. Data arrives in unexpected shapes. Null handling is wrong. Outside-in development eliminates these surprises by starting from reality.

### The Adapter Principle

Every boundary between systems — not just external APIs — gets an adapter. The adapter translates between the shape on one side and the shape on the other.

| Boundary | External side | Domain side | Adapter |
|----------|--------------|-------------|---------|
| Supplier API | Raw API response | Domain entity | Supplier adapter |
| Form submission | Form data / request body | Domain input schema | Input validation + mapping |
| Webhook | Webhook payload | Domain event | Webhook parser |
| CSV import | Raw CSV rows | Domain entities | Import mapper |
| Database | Table rows (storage shape) | Domain entities | Repository |

The repository is an adapter. The form validator is an adapter. The webhook handler is an adapter. Recognizing this unifies how you think about all system boundaries — they all follow the same pattern: receive external shape → validate → transform → emit domain shape.

### Contract-First Design

Define the contract before building the implementation. This applies at every level:

- **Before writing a database migration**, define the domain schema that the table must serve
- **Before writing a server action**, define the input schema (what the UI sends) and the output shape (what comes back)
- **Before integrating an external API**, inspect and document the actual response shape
- **Before building a UI form**, define the validation schema that governs what inputs are accepted

When the contract is defined first, the implementation becomes mechanical — fill in the code that satisfies the contract. When implementation comes first, the contract is discovered retroactively, often inconsistently, and always with more bugs.

---

## The Vertical Engineering Workflow

Each feature vertical follows an 8-step engineering sequence. The steps are ordered by dependency — each one builds on the output of the previous. Skipping steps or reordering them creates rework.

This workflow operates *within* the Build stage of the pipeline. The pipeline stages (Research → Shape → Breadboard → Plan → Build → Review → Wrap-up) determine *what* to build and *when*. This workflow determines *how* to engineer it.

### The 8 Steps

**1. User Journey → Data Requirements**

Start from what the user does, not from what the database needs. Walk through the journey step by step and ask: what data must exist at each moment?

- "User selects a customer" → needs a customer ID and a way to search
- "User picks a garment" → needs garment catalog data accessible by style
- "User enters sizes" → needs a size-to-quantity map
- "User sees pricing" → needs a calculation result derived from all of the above

The user journey defines the data requirements. The data requirements define the schema.

**2. Domain Schema → The Source of Truth**

Define validated schemas for every entity and input. These are the single source of truth for what valid data looks like in your system.

- Entity schemas define what a Customer, Quote, or Job *is*
- Input schemas define what a valid `createQuote` or `updateJob` request looks like
- Output schemas define what the system returns after an operation

The domain schema is designed to serve the user journey — not to mirror the database, not to match an external API, not to satisfy a UI component's prop interface.

**3. State Machine → Legal Transitions**

For any entity with a lifecycle (quotes, jobs, invoices), define the legal state transitions as a state machine. What statuses exist? Which transitions are allowed? What conditions must be met?

```
draft → sent → accepted
                → declined → revised (new draft)
draft → voided
sent → voided
```

The state machine is a domain rule — it lives in the domain layer, not in the database (which only stores the current state) and not in the UI (which only renders transitions the domain allows). Defining it explicitly prevents impossible transitions and makes the lifecycle auditable.

**4. Database Migration → Storage That Serves the Domain**

Design tables, columns, indexes, and constraints to serve the domain schema defined in step 2. The database is an implementation detail of persistence — it stores domain entities, it doesn't define them.

Key principles:
- Column types match domain schema types (validated schemas drive column choices)
- Indexes support the queries the user journey requires (not hypothetical queries)
- Constraints enforce invariants the database can check cheaply (uniqueness, foreign keys, not-null)
- Business rules that require logic beyond constraints live in the domain layer, not in triggers or check constraints

**5. Repository Port → Query Interface**

Define a port interface — the set of queries and mutations the domain needs — without specifying how they're implemented. The port describes *what* the infrastructure must provide, not *how*.

```
findById(id) → Entity | null
findByCustomerId(customerId) → Entity[]
create(input) → Entity
updateStatus(id, newStatus) → Entity
```

Coding against port interfaces means the domain layer never knows whether data comes from a real database, a mock, or a cache. This is what makes the system testable without infrastructure and portable across implementations.

**6. Repository Implementation → Infrastructure Wiring**

Implement the port interface with concrete infrastructure — an ORM, raw SQL, a cache layer. This is the only layer that knows about specific database technology, connection strings, or query syntax.

The repository is an adapter (see "Contracts at Every Boundary"). It translates between database-shaped rows and domain-shaped entities. If the database stores `created_at` as a timestamp but the domain schema expects an ISO string, the repository handles that translation.

**7. Server Actions → The Surface Contract**

Define what the UI can trigger. Each server action validates its input against a schema, calls domain logic, persists via the repository, and returns a typed result. The server action *is* the API contract between your frontend and your backend.

Server actions are thin — they orchestrate, they don't contain business logic. The pattern: validate input → call domain service → call repository → return result. If a server action grows complex, the complexity belongs in a domain service.

**8. UI Components → Render the Contract**

Build UI to the server action contracts defined in step 7. The UI sends data shaped by input schemas and renders data shaped by output schemas. When the contracts are well-defined, the UI is the least surprising part of the system — it's a visual rendering of data shapes that were already decided.

### Why This Order Matters

The sequence is designed so that each step depends only on completed previous steps:

- Steps 1-3 require no code — they're **research and design** outputs (typically M0)
- Steps 4-6 are the **schema and API** build (typically M1)
- Steps 7-8 are the **feature build** (typically M2+)

When steps are skipped or reordered, the most common failure modes are:
- Building UI before the domain schema exists → UI-shaped data that doesn't match business rules
- Building database tables before the domain schema → storage-shaped entities that force UI contortions
- Building server actions before state machines → impossible transitions that crash at runtime
- Skipping the repository port → infrastructure assumptions leak into domain logic

### Horizontal Engineering Workflow

Horizontal infrastructure — shared capabilities that multiple verticals consume — follows a different sequence. Horizontals don't have a user journey or a UI. They serve other parts of the system.

**The 6 Steps:**

**1. Consumer Analysis → Minimum Viable Interface**

Start from the vertical that pulls this horizontal. What exactly does it need? What is the smallest possible capability that unblocks the consumer? Horizontal infrastructure is built just ahead of the vertical that needs it — not speculatively.

Examine the consumer's requirements concretely:
- "The quoting vertical needs to send a PDF via email" → the email horizontal needs: send one email with one attachment to one recipient
- "The artwork vertical needs to accept file uploads" → the storage horizontal needs: presigned upload URL, metadata persistence, file retrieval by ID

The consumer's needs define the interface. Everything else is deferred.

**2. Interface Contract → The Port Consumers Code Against**

Define the interface that consumer verticals will use — without specifying how it's implemented. This is the horizontal's API contract.

```
sendEmail(to, subject, body, attachments?) → { success } | { error }
uploadFile(file, metadata) → { fileId, url }
emitEvent(type, payload) → void
```

The interface should be the thinnest possible abstraction. If consumer verticals need to know implementation details (which email provider, which storage bucket), the interface is too leaky.

**3. Option Evaluation → Informed Implementation Choice**

For infrastructure decisions, evaluate 2-3 options against concrete criteria:
- **Cost** — free tier limits, scaling costs, per-request pricing
- **Complexity** — setup time, maintenance burden, operational overhead
- **Lock-in** — how hard is it to switch? Is the interface portable?
- **Reliability** — uptime, retry behavior, failure modes
- **Time to implement** — hours, not days

Document the decision and the reasoning. When the next project needs the same capability, the evaluation transfers even if the choice doesn't.

**4. Implementation → Build Behind the Interface**

Implement the concrete infrastructure behind the port interface. The consumer vertical doesn't know or care about implementation details — it codes against the interface from step 2.

This is where specific tools, SDKs, configuration, and provider-specific logic live. If the email provider changes, only this layer changes. The consumer's code is untouched.

**5. First Consumer Integration → Validate the Interface**

Wire the first consumer vertical to the horizontal. This is the moment of truth — does the interface actually serve the consumer's needs?

If the consumer needs something the interface doesn't provide, revise the interface now, before adding more consumers. The first integration is cheap to change; the fifth is not.

**6. Hardening → Reliability for Multiple Consumers**

Horizontals must be more reliable than verticals because multiple consumers depend on them. After the first integration validates the interface:

- Add error handling and retry logic appropriate to the infrastructure type
- Add monitoring or structured logging for operational visibility
- Document failure modes and recovery paths
- Test edge cases the first consumer didn't exercise

**When to stop:** Horizontal hardening follows the same "just ahead" principle as horizontal construction. Harden to the level the current consumers need. Don't build retry queues for an email system that sends 5 emails a week.

---

## Mature by Default: Architecture Earns Trust

Dependencies point inward only. Domain logic imports nothing from infrastructure or UI layers.

We follow clean architecture not because it's fashionable but because it survives change. When a supplier API changes, only the adapter layer moves. When the UI framework updates, domain rules stay untouched. When a new project starts with a different stack, the domain thinking transfers.

**In practice:**
- Code against port interfaces, not implementations
- Composition root is the only place where wiring happens
- SOLID principles are explicitly audited, not just referenced
- Twelve-Factor App compliance for anything that deploys

**The Wave 0 pattern:** Every significant system change starts with a zero-behavior-change foundation. Install dependencies, scaffold structure, wire interfaces — but change no behavior. Then Wave 1+ activates features on top of a stable base. This reduces the blast radius of any single change and gives agents a clean starting point.

---

## Financial Precision Is Non-Negotiable

Never use floating-point arithmetic for money. In any language, on any project.

This is not a preference — it's a constraint on correctness. Floating-point representation errors compound silently through calculations. A customer seeing $0.01 discrepancies erodes trust instantly. Financial code uses arbitrary-precision decimal libraries with explicit rounding.

**Enforcement:** 100% test coverage on all monetary calculation code. Zero tolerance. This is the one area where "good enough" is never good enough.

---

## Every Tool Earns Its Place

No dependency is added "because everyone uses it." Every tool in the stack has a documented reason for its inclusion, a description of when to use it, and — critically — when NOT to use it. Tools that don't earn their keep get removed.

**Decision framework for adding dependencies:**
- Does this solve a problem we actually have today (not theoretically)?
- Is the maintenance burden justified by the value?
- Does it have a clear boundary — can it be replaced without rewriting the system?
- Are we willing to own its behavior in production?

**Versioning posture:** Pin majors for frameworks. Range for utilities. Never upgrade "just because a new version exists." Upgrades happen when they solve a real problem or when security requires it.

---

## Progressive Improvement Over Big-Bang Rewrites

Systems improve incrementally. Each cycle leaves the codebase better than it found it, but never through a ground-up rewrite.

The most dangerous engineering decision is "let's start over." The second most dangerous is "let's do it properly later." The right path is continuous, small improvements that compound:

- Refactor when you're already touching the code, not as a separate project
- Technical debt is acceptable when it's ephemeral — you're going to refactor next week anyway and it won't block anything. Debt is unacceptable when it will persist and compound
- Infrastructure gets built "just ahead" of the vertical that needs it — not speculatively, not after the fact

---

## Structured Logging, Not Print Debugging

All production logs flow through a structured logger with domain context. Raw console output is never acceptable in production code.

Structured logs are machine-readable, searchable, and filterable. They carry context (request ID, user ID, domain) that makes debugging possible without reproducing the exact scenario. When agents write code, they don't debug with print statements — they emit structured events that future sessions can query.

---

## URL State Over Hidden State

Application state that affects what a user sees — filters, search terms, pagination, selected views — lives in the URL. Not in component state, not in a global store, not in session storage.

URL state is shareable, bookmarkable, and survives page reloads. It's the most durable, debuggable form of client state. When an agent needs to understand what a user is looking at, the URL tells the whole story.
