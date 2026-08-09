# Using This Guidance With AI Coding Agents

`event-driven-security-guidance.md` can be provided to AI coding agents when they create, modify, refactor, or review event-driven code.

The document should be treated as **security review context**, not as a set of requirements that automatically applies to every application.

## Recommended instruction

When providing this repository or file to a coding agent, use an instruction similar to:

> Read `event-driven-security-guidance.md` before modifying or reviewing event-driven code.
>
> Determine which security considerations apply to this application's actual architecture.
>
> Pay particular attention to authentication, authorization, trust boundaries, privileged consumers, audit requirements, delayed processing, and sensitive data carried in events.
>
> Do not mechanically implement every recommendation in the document.
>
> Preserve or strengthen the application's existing security requirements. If an event-driven change introduces a new security assumption, trust boundary, privilege relationship, or authorization dependency, document it in the application's appropriate security or architecture documentation.

## For code generation

When an agent is implementing new event-driven functionality:

> Review the proposed event flow using `event-driven-security-guidance.md`.
>
> Identify security checks that must occur before the event is emitted, security checks that belong in the consumer, and operations that are only optional reactions.
>
> Do not move required authentication, authorization, ownership, tenant, policy, or audit requirements into best-effort listeners unless the application's security design explicitly permits it.

## For code review

When an agent is reviewing an existing change:

> Review event producers and security-relevant consumers using `event-driven-security-guidance.md`.
>
> Look specifically for:
>
> * required security checks moved behind listeners,
> * events treated as authorization,
> * privileged consumers trusting user-controlled identifiers,
> * external events entering trusted flows before verification,
> * authentication being confused with authorization,
> * required audit records becoming best-effort,
> * stale authorization during delayed processing,
> * unnecessary credentials or sensitive data in event payloads.
>
> Report only findings that apply to the actual architecture.

## Incorporating the guidance into an application's own documentation

Agents may use this document as a source when updating:

* `SECURITY.md`
* `AGENTS.md`
* `CLAUDE.md`
* threat models
* architecture documentation
* secure coding standards
* pull request checklists

The preferred approach is to **adapt the relevant requirements to the application**, rather than copying the entire document.

For example, an application using authenticated webhooks and asynchronous administrative workers might incorporate only the guidance covering:

* verification before internal fan-out,
* authentication versus authorization,
* confused-deputy risks,
* delayed authorization,
* audit handling.

An application using only local UI observers may require far fewer sections.

## Intended behavior

The goal is for the agent to reason about the application's security model, not simply recognize security keywords.

A useful result should explain:

1. which guidance applies,
2. why it applies to the application's architecture,
3. where the security control should be enforced,
4. what could happen if the control is omitted,
5. whether code, tests, or documentation should change.
