<!--
Sync Impact Report
- Version change: 1.0.0 → 1.1.0
- Modified principles: none renamed
- Added sections:
  - IX. AI Cost
- Removed sections: none
- Quality Gates and Compliance review now enforce IX
- Follow-up TODOs: none
- Migration: no shipped path calls a paid model. Draft features adopt
  this rule when they are planned.
-->

# GMR Bot Constitution

## Core Principles

### I. Security & Privacy First

Every user input MUST be validated against an explicit schema or allowlist
before it is used. Sensitive data MUST NOT be stored in plain text. The system
MUST retain only the technical metadata required to operate the feature. Stored
and transmitted data MUST be accessible only to authorized components.

**Rationale**: Unvalidated Telegram input and plaintext sensitive data are the
primary paths to unauthorized disclosure.

### II. Configuration & Secrets

Secrets, tokens, and credentials MUST NOT be hardcoded. They MUST NOT be
committed to the repository. They MUST be supplied from outside the source tree.

**Rationale**: A credential in git history remains exposed after deletion from
the working tree.

### III. Architecture & Contracts

Services MUST be loosely coupled. They MUST communicate only through explicitly
defined contracts. A service MUST NOT read or write another service's database.

**Rationale**: Direct database access bypasses the contract other services rely
on and ties their release cycles together.

### IV. Resilience & UX

A failure in one microservice MUST NOT crash the bot. Every error on a user
facing path MUST be handled and MUST produce a clear, user-friendly Telegram
response. The bot MUST NOT stay silent when a request fails.

**Rationale**: In Telegram, silence is indistinguishable from a dead bot.

### V. Rate Limiting & Performance

Every public entry point MUST enforce a rate limit before doing further work.
Work on that path MUST be bounded so one request cannot stall replies to other
users.

**Rationale**: Unlimited traffic and unbounded handlers make Telegram replies
late or unavailable.

### VI. Code Quality & Documentation

Code, public APIs, and architectural decisions MUST be documented in English.
Documentation MUST state the purpose and the contract. Public behavior without
that documentation is incomplete.

### VII. Testing

A new feature or bug fix MUST include automated tests that fail when the
changed behavior regresses. A change without those tests is incomplete.

### VIII. Observability

Each service MUST emit structured logs and MUST record errors in a form that
can be queried later. Logs and error records MUST NOT contain sensitive user
data, including message content, credentials, tokens, or other private
payloads.

### IX. AI Cost

A paid model call MUST be made only when no deterministic step can produce the
required result. The call MUST send only the input that result needs, MUST use
the lowest-cost model that still meets the feature's acceptance criteria, and
MUST bound input size, output size, and retries. The same input MUST NOT be
sent again while a usable result for that input still exists.

**Rationale**: An unbounded prompt, a costlier model than the result needs, or
a repeated call turns one chat feature into an open-ended bill.

**Migration**: No shipped path calls a paid model. Features already specified
adopt this rule when they are planned.

## Platform Constraints

GMR Bot is a Telegram chat assistant and utility toolkit built as
microservices.

- Telegram is the user-facing channel. Validation, rate limits, and
  user-visible errors apply to every public Telegram entry point.
- Services MUST stay independently deployable. The only permitted coupling is a
  published contract.
- Each service is the sole reader and writer of its own data store.

## Quality Gates

A change MUST NOT be accepted as complete unless all of the following hold:

- User input introduced or modified by the change is validated.
- No secret, token, or credential is hardcoded or added to the repository.
- No path reads or writes another service's database.
- A dependency failure returns a clear Telegram reply and does not stop the
  rest of the bot.
- Each public entry point touched by the change enforces a rate limit.
- Automated tests cover the new or fixed behavior.
- Code, APIs, and architectural decisions added by the change are documented
  in English.
- New logs and error records exclude sensitive user data.
- A new or changed paid model call is skipped when a deterministic step can
  produce the result, sends only the input that result needs, uses the
  lowest-cost model that meets the feature's acceptance criteria, bounds input
  size, output size, and retries, and is not repeated while a usable result
  exists.

## Governance

This constitution supersedes conflicting practice, templates, and review
habits. Feature specs, plans, and tasks under `.specify/` MUST conform to it
and MUST NOT relax it.

**Amendments**:

- A change to principles or governance MUST be made by amending this file.
- The amendment MUST bump the version, set Last Amended to the amendment date,
  and leave Ratified unchanged.
- MAJOR: a principle is removed or redefined. MINOR: a principle is added or a
  rule is materially expanded. PATCH: wording changes that do not change
  obligations.
- When the new rule would make existing behavior non-compliant, the amendment
  MUST state the migration expectation.

**Versioning**:

Versions follow MAJOR.MINOR.PATCH as defined under Amendments.

**Compliance review**:

- Authors and reviewers MUST check the change against the Quality Gates before
  it is accepted.
- A review MUST reject a change that stores sensitive data in plain text,
  commits a secret, crosses a service boundary through another service's
  database, fails silently toward the user, skips required tests or English
  documentation, records sensitive user data in logs or error reports, or
  calls a paid model without the bounds in principle IX.
- Day-to-day feature guidance lives in the spec artifacts. This file remains
  the source of non-negotiable rules.

**Version**: 1.1.0 | **Ratified**: 2026-10-27 | **Last Amended**: 2026-10-27
