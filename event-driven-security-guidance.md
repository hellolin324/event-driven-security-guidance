# Security Considerations for Observer Pattern and Event-Driven Code

> **Status:** v1
> 
> **Purpose:** Reusable security guidance for developers and coding agents working with Observer Pattern, pub/sub, application events, message queues, webhooks, event buses, and similar event-driven designs.

## Scope

This document focuses on security issues that can be introduced when application behavior is moved behind events, listeners, subscribers, or background consumers.

It is not intended to replace general guidance for:

* event-driven architecture,
* concurrency,
* transactions,
* reliability,
* queue design,
* retry handling,
* performance,
* schema management.

Applications should incorporate the relevant parts of this document into their own security and architecture documentation.

The main concern is simple:

**A required security control must not become optional just because the application uses events or observers.**

---

## 1. Keep Required Security Controls in the Protected Operation

Before moving logic into an observer or subscriber, determine whether that logic is required for the operation to be allowed.

Examples include:

* authentication,
* authorization,
* MFA enforcement,
* tenant checks,
* resource ownership checks,
* account status checks,
* required security policy checks,
* security-critical input validation,
* creation of an authenticated session.

If the operation must not succeed without the control, the control should not be implemented only as a best-effort observer.

### Bad

```text
Delete account
      ↓
AccountDeleted
      ├── AuthorizationListener
      └── NotificationListener
```

Authorization happens after the protected action.

### Better

```text
Authenticate user
      ↓
Authorize account deletion
      ↓
Apply required policy checks
      ↓
Delete account
      ↓
AccountDeleted
      └── NotificationListener
```

### Security review question

**Could the operation still complete if this listener never runs?**

If yes, anything implemented only in that listener should not be a security control required to approve the operation.

---

## 2. Completed Events Should Represent Completed Security Checks

Event names often describe a completed action:

```text
UserLoggedIn
PaymentAuthorized
AccountVerified
RoleGranted
DocumentApproved
```

The application should not publish such an event until the security checks required for that state have completed successfully.

For example:

```text
Validate credentials
      ↓
Complete required MFA
      ↓
Check account status
      ↓
Create authenticated session
      ↓
UserLoggedIn
```

Do not publish `UserLoggedIn` and depend on later listeners to finish authentication.

If the event represents a request rather than a completed action, name it accordingly:

```text
LoginRequested
AccountDeletionRequested
PaymentAuthorizationRequested
```

The difference between a command/request event and a completed-state event should be clear to both developers and consumers.

---

## 3. Receiving an Event Is Not Authorization

An event tells a consumer that another component sent a message.

It does not automatically prove that the requested action is authorized.

For example:

```text
GenerateReport {
    account_id: "123"
}
```

A background worker with access to many accounts should not assume that the original requester was authorized to access account `123`.

Security-sensitive consumers may need to verify:

* user or service identity,
* tenant,
* resource ownership,
* requested operation,
* authorization scope,
* current permissions.

How this is implemented depends on the application. Possible approaches include:

* checking authorization again,
* looking up current ownership or permissions,
* passing trusted authorization context,
* using a narrowly scoped delegated credential or capability.

Do not pass long-lived credentials in event payloads simply to avoid performing authorization checks.

---

## 4. Watch for Confused-Deputy Problems

A common event-driven security problem occurs when a less-privileged component sends instructions to a more-privileged component.

```text
User-facing service
        ↓
event
        ↓
Privileged background worker
```

Example:

```text
DeleteObject {
    object_id: "victim-object"
}
        ↓
worker with broad storage permissions
        ↓
object deleted
```

If the worker trusts the object ID without checking whether the requester was allowed to delete it, the worker can become a **confused deputy**.

Consumers with elevated privileges should validate the security context of attacker-controlled or user-controlled fields before acting.

Pay particular attention to consumers with:

* administrative API access,
* broad database permissions,
* cloud IAM privileges,
* cross-tenant access,
* filesystem access,
* payment or billing permissions,
* account-management privileges.

---

## 5. Verify External Events Before Publishing Them Internally

Events received across a trust boundary should be authenticated and validated before being treated as trusted internal events.

Examples include:

* webhooks,
* partner integrations,
* externally accessible message endpoints,
* cross-account event buses,
* third-party callbacks.

Preferred flow:

```text
External event
      ↓
Verify sender/signature
      ↓
Validate required security fields
      ↓
Publish trusted internal event
      ↓
Application listeners
```

Avoid:

```text
External event
      ↓
Internal event bus
      ├── SignatureCheckListener
      ├── BusinessListener
      └── AdminListener
```

The application should not depend on the security listener executing first.

For signed webhooks, follow the provider's signature-verification requirements before processing or redistributing the event.

---

## 6. Message Authentication and Authorization Are Different Checks

A valid signature can establish that a message came from a trusted sender and was not modified.

It does not necessarily mean every action described in that message should be allowed.

Example:

```text
Valid partner webhook
      ↓
Modify customer resource
```

The receiver may still need to check:

* which customer owns the resource,
* whether the partner is allowed to perform that operation,
* whether the requested action is allowed for that account,
* whether the resource is still in a valid state.

Treat message authenticity and action authorization as separate security decisions.

---

## 7. Separate Required Audit Records from Audit Delivery

Audit logging often looks like an ordinary side effect, but some audit records are required security evidence.

There is a difference between:

```text
Create the required audit record
```

and:

```text
Forward the record to Splunk, Sentinel, or another SIEM
```

An application may require the first operation to succeed while allowing the second to happen asynchronously.

Example:

```text
Privileged action
      ↓
Authorization check
      ↓
State change
      +
Required audit record
      ↓
ActionCompleted
      └── SIEM forwarding listener
```

Applications should define which security events:

* must be recorded,
* must be recorded durably,
* may be forwarded asynchronously,
* may be best effort.

Do not move required audit logging into a fire-and-forget listener without considering what happens when that listener fails.

---

## 8. Define Failure Handling for Security-Related Listeners

Not every security-related listener needs to block the original operation.

Different security functions can have different failure requirements.

Example:

```text
LoginSucceeded
      ├── Product analytics          → best effort
      ├── Login notification         → retry
      └── Security telemetry         → durable retry and alert
```

For each security-related listener, determine:

**What happens if this listener permanently fails?**

Possible responses may include:

* reject the operation,
* retry,
* store the work durably for later processing,
* raise an alert,
* enter a degraded operating mode,
* reconcile the missing security data later,
* explicitly accept the risk.

Security-related listener failures should not be silently ignored unless the application has intentionally classified that function as best effort.

---

## 9. Recheck Authorization When Processing Is Delayed

Authorization may change between the time an event is created and the time it is processed.

Examples:

```text
ExportRequested
AdminActionRequested
ShareResourceRequested
PrivilegeChangeRequested
```

Before the consumer runs:

* the user's account may be disabled,
* a role may be removed,
* ownership may change,
* consent may be revoked,
* the resource may move to another tenant.

For security-sensitive operations, determine whether authorization must be checked again at processing time.

Do not assume that authorization that was valid when the event was published remains valid indefinitely.

---

## 10. Limit Sensitive Security Data in Events

Event payloads may be visible to more components than direct function arguments.

They may also be stored in queues, logs, retries, dead-letter queues, tracing systems, or third-party infrastructure.

Avoid placing unnecessary sensitive data in events, especially:

* passwords,
* refresh tokens,
* session tokens,
* API secrets,
* broadly scoped access tokens,
* unnecessary PII.

Prefer identifiers or narrowly scoped security context when possible.

An observer or consumer should receive only the sensitive information it actually requires.

---

## Security Review Checklist

When reviewing code that introduces or changes observers, subscribers, queues, webhooks, or event consumers, check the following.

### Required security controls

* Is authentication, authorization, MFA, ownership checking, or another required security control being moved into a listener?
* Can the protected operation complete if that listener fails or never executes?

### Event meaning

* Does the event describe a request or a completed action?
* If it describes a completed action, were the required security checks completed before publication?

### Authorization

* Does the consumer perform a privileged operation?
* Is the consumer treating receipt of the event as proof of authorization?
* Does it verify tenant, ownership, and resource scope where required?

### Privileged consumers

* Does the consumer have more privileges than the producer?
* Could user-controlled event fields cause the consumer to act outside the requester's permissions?

### Trust boundaries

* Can an untrusted or external system produce the event?
* Is the sender verified before the event enters the trusted internal event flow?

### Security logging

* Is the listener responsible for required security audit data?
* What happens if audit creation or delivery fails?

### Delayed execution

* Could permissions or ownership change before the event is processed?
* Does the operation need authorization to be checked again?

### Sensitive information

* Does the event contain credentials, tokens, secrets, or unnecessary personal information?
* Will the event be stored or exposed to infrastructure that does not need that information?

---

## Patterns That Require Security Review

### Authorization implemented as a listener

```text
ActionCompleted
      └── AuthorizationListener
```

### Security depends on listener order

```text
Event
 ├── SecurityListener
 └── PrivilegedBusinessListener
```

### Event treated as permission

```text
Received DeleteAccountEvent
→ Delete account without checking authorization
```

### Privileged worker trusts user-controlled identifiers

```text
event.resource_id
      ↓
Privileged worker
      ↓
Access resource without ownership check
```

### External event enters the internal bus before verification

```text
Webhook
   ↓
Internal event bus
   ├── VerifySignatureListener
   └── BusinessListener
```

### Required audit logging becomes fire-and-forget

```text
Privileged action
      ↓
publish(AuditEvent)
      ↓
Ignore failure
```

---

## Guidance for Coding Agents

When modifying event-driven code:

1. Identify the security checks required before the protected action is allowed.
2. Do not move those checks into optional or best-effort listeners.
3. Make sure completed-state events are published only after required security checks succeed.
4. Do not treat receipt of an event as authorization.
5. Check privileged consumers for confused-deputy problems.
6. Verify external events before they are distributed to trusted internal consumers.
7. Keep message authentication separate from authorization of the requested action.
8. Determine whether security logging is required or best effort before moving it behind an event.
9. Check whether delayed processing requires authorization to be evaluated again.
10. Avoid placing unnecessary credentials or sensitive data in event payloads.

When applying these rules to an application, document the application's actual security requirements rather than copying generic assumptions from this file.

---

## Summary

Observer and event-driven designs can make application behavior easier to separate and extend. That separation can also make required security checks less visible.

The key security questions are:

* **Was the action authorized before it occurred?**
* **Does the event accurately describe what has already happened?**
* **Does the consumer have enough information to safely perform its action?**
* **Is a privileged consumer being trusted with attacker-controlled instructions?**
* **Was an external event verified before trusted components received it?**
* **Can required security logging disappear because a listener failed?**
* **Could permissions change before delayed processing occurs?**

These questions should be answered by the application's own security design.
