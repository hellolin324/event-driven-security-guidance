# Event-Driven Security Guidance

Reusable AppSec guidance for developers and AI coding agents working with Observer Pattern, pub/sub, application events, message queues, webhooks, event buses, and similar event-driven designs.

## Why this exists

Event-driven designs make it easy to move application behavior behind listeners, subscribers, and background consumers.

That can also make required security controls easier to accidentally move out of the operation they are supposed to protect.

For example:

```text
Authenticate
    ↓
Authorize
    ↓
Create session
    ↓
UserLoggedIn
    ├── Analytics
    └── Notification
```

is very different from:

```text
UserLoggedIn
    ├── CreateSession
    ├── AuthorizationCheck
    ├── Analytics
    └── Notification
```

The second design may have turned authentication and authorization requirements into optional side effects.

This project focuses on identifying those kinds of problems.

## What this covers

The guidance addresses security questions such as:

* Should a security check happen before an event is published?
* Is a consumer treating receipt of an event as authorization?
* Can a privileged consumer become a confused deputy?
* Can an external event enter an internal event flow before its sender is verified?
* Has required audit logging become fire-and-forget?
* Should authorization be checked again when an event is processed later?
* Is sensitive security context being unnecessarily propagated through events?

It is intentionally **not** a general guide to event-driven architecture, distributed systems, concurrency, retries, message ordering, or broker reliability.

## Main guidance

See:

[`event-driven-security-guidance.md`](./event-driven-security-guidance.md)

That file contains the reusable security guidance.

## How developers and security engineers can use it

Use the guidance when designing or reviewing:

* Observer Pattern implementations
* application events
* event listeners
* pub/sub systems
* background workers
* message queues
* event buses
* webhook handlers
* asynchronous privileged operations

Relevant sections can be incorporated into an application's:

* `SECURITY.md`
* threat model
* architecture documentation
* secure coding standard
* design review checklist
* pull request review process

Not every section will apply to every application.

The intent is to identify the security concerns relevant to the application's actual architecture and document or enforce those requirements there.

## How AI coding agents can use it

This guidance can also be provided to coding agents as security context when they create, refactor, or review event-driven code.

Agents should use it to identify applicable security concerns and adapt them to the application's actual:

* authentication model
* authorization model
* trust boundaries
* tenant model
* event infrastructure
* privilege boundaries
* audit requirements

See [`AGENT-USAGE.md`](./AGENT-USAGE.md) for suggested integration instructions.

## Using only part of this guidance

The document is intentionally modular.

An application using only an in-process Observer Pattern may primarily need the sections covering:

* required security controls
* event meaning
* authorization
* privileged consumers

A webhook-heavy application may also need:

* trust-boundary verification
* message authentication versus authorization

Applications with asynchronous privileged workers may place more emphasis on:

* confused-deputy risks
* delayed authorization
* audit handling
* sensitive event data

Teams are encouraged to extract, shorten, rewrite, or incorporate relevant sections into their existing security documentation.

## Core review question

When event-driven behavior is introduced or changed, ask:

> **Did something that was previously required for an operation to be secure become optional because it was moved behind an event?**

## Status

This guidance is intended to evolve as additional event-driven security patterns and failure cases are identified.

Feedback, examples, and proposed improvements are welcome.
