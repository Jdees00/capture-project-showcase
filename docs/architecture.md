# Architecture

How the Capture booking system is put together, and why.

This describes the design at a level useful to a reader evaluating the work.
It deliberately omits hostnames, resource identifiers, binding names, endpoint
paths and configuration values. Those live in the private production
repository.

---

## Shape of the system

The whole thing runs on one edge platform. There is no separate application
server, no container to keep alive, and no scheduled job. Requests arrive at
the edge, run a small function, touch a database that lives alongside it, and
return.

```
 ┌───────────────────────────────────────────────────────────────────┐
 │                            EDGE PLATFORM                          │
 │                                                                   │
 │   Static site            Serverless functions        SQL database │
 │   ───────────            ─────────────────────       ──────────── │
 │   Marketing page         Public:                     bookings     │
 │   5-step scheduler         availability              blackouts    │
 │   Booking desk             submit request            rate_limit   │
 │                          Private (authenticated):    audit_log    │
 │                            list bookings             credentials  │
 │                            change a booking                       │
 │                            sign-in / enrolment                    │
 └───────────────┬───────────────────────┬───────────────┬───────────┘
                 │                       │               │
                 ▼                       ▼               ▼
         Calendar provider        Email provider    Spam challenge
         (free/busy only)         (owner alerts)      (optional)
```

### Why this shape

**No server to operate.** A photography business does not want an application
server to keep running. Functions at the edge start when a request arrives and
stop when it finishes.

**The database sits with the functions.** An availability check reads several
tables. Keeping them adjacent keeps that fast enough to feel instant on a
phone at the lake.

**External services are reached outbound only.** Nothing inbound comes from the
calendar or email provider. There are no webhooks to secure and no callback
URLs to keep secret.

---

## One source of operating truth

A single configuration file defines:

- the service catalogue — each session's name, price, duration, and the travel
  and setup buffers held either side of it
- opening hours per weekday
- how much notice a booking needs, and how far ahead one may be made
- the time zone all scheduling is reasoned in
- the deposit policy

Three separate parts of the system read it: the public scheduler, the
server-side availability engine, and the booking desk.

**Why this matters.** When a price, an opening time or a session length
changes, it changes in one place. There is no state in which the page offers a
20-minute session while the calendar holds 15. Two systems disagreeing about
an operating rule is one of the most common and most expensive failure modes in
any operation — this design makes it impossible rather than unlikely.

---

## How a time slot is decided

The hardest logic in the system. A candidate start time is offered only if the
**entire held block** — the session plus travel and setup either side — clears
every one of:

1. the declared opening hours for that weekday
2. the minimum notice period
3. the maximum booking window
4. every busy span on the live calendar
5. every confirmed booking, and its buffers
6. every approved-but-unpaid booking, and its buffers
7. every pending request that has not yet expired, and its buffers
8. any manually blocked date or weather closure

Point 7 is the one most systems miss. A request that has been made but not yet
answered must hold its time, or two customers are told the same slot is free.
It holds that time on a timer, and releases it automatically if it goes
unanswered.

### The same function answers two questions

`"show me the open times this week"` and `"is this exact time still free"` are
the same code path. The second runs again, server-side, at the moment a request
is submitted.

This is the double-check that prevents a race: two customers with the page open
both see 2:00 PM, both click it, and the second submission is refused with an
explanation rather than silently accepted.

### Time zones

All scheduling is reasoned in a single named zone, with explicit handling of
daylight-saving transitions. The test suite asserts behaviour on the exact
spring-forward and fall-back dates, because a one-hour error on a shoot day is
a missed session.

---

## Booking lifecycle

```
                    ┌──────────► Declined
                    │
   Request ──► Pending ──► Approved ──► Confirmed
                    │      (awaiting      (deposit
                    │       deposit)       received)
                    │           │              │
                    ▼           ▼              ▼
                 Expired     Cancelled     Cancelled
              (hold timed
                   out)
```

**Approved is not Confirmed.** Approving holds the time on the calendar. The
customer becomes confirmed only when the retainer is received. That distinction
is an operating decision — it stops the calendar filling with provisional work.

Every transition writes an audit record: what changed, who changed it, when,
and the stated reason where one applies. That log is the answer to "what
happened with this booking" without reconstructing it from memory.

---

## Calendar integration

A service account reads the owner's calendar through the provider's
**free/busy** interface — times only. The integration never reads event titles,
descriptions, attendees or locations. It cannot: free/busy returns start and
end times and nothing else.

That was a deliberate choice. The system needs to know *when* the owner is
busy. It has no reason to know *why*, so it was never given the ability to find
out.

Authentication is a short-lived signed token, minted per request from a private
key held in encrypted storage. The key is never in the repository, never in
the deployed bundle, and never reaches a browser.

---

## Notification

When a request is stored, one email goes to the owner.

**Design requirements, in priority order:**

1. **A notification failure must never cost a booking.** The email is sent
   after the booking is committed. If the provider is slow or down, the request
   is already saved.
2. **The call is time-bounded.** It is abandoned after a few seconds rather
   than holding the customer's request open.
3. **Every outcome is recorded** — sent, failed, or skipped because the
   provider is not configured — written to the audit log, so a missing email is
   a visible fact rather than a mystery.
4. **Nothing is emailed to the customer automatically.** Every customer-facing
   message is reviewed and sent by a person. The desk prepares the draft; a
   human sends it.

---

## Administrative access

The booking desk holds customer contact details, so it is locked.

**Passkeys (WebAuthn).** Sign-in is Face ID or Touch ID on an enrolled device.
Enrolling a new device needs a one-time code, held in encrypted storage,
validated with a constant-time comparison and refused entirely if it is too
short to be strong. Sessions are short-lived and held in a cookie that
JavaScript cannot read.

**Fails closed.** Every failure path — no credential, a bad signature, an
expired session, missing configuration — returns the same refusal. An
unconfigured deployment is a locked one, never an open one. There is no state
in which forgetting to configure something leaves customer data readable.

**A local development bypass exists and is refused unless the request comes
from the local machine**, so it cannot be left switched on by accident in
production.

---

## Deployment safety

The deployment step is itself a control.

The build tool copies files into the published output from an **explicit
allow-list**: the public page, the admin page, media, and front-end modules.
Everything else — backend functions, shared libraries, the database schema,
documentation, test tooling, configuration examples — is simply not named, so
it is not published.

The build then **inspects its own output** and refuses to finish if anything
server-side has appeared there.

This is defence in depth for the failure that matters most: it is not possible
to accidentally deploy backend code or a configuration file, because the only
way for a file to be published is to be named as publishable.

---

## Testing

Nine suites, 461 checks, all passing.

| Suite | What it proves |
|---|---|
| Availability | Opening hours, buffers, notice period, booking window, conflict detection, daylight-saving transitions, and the admin override that places an out-of-hours session without ever double-booking |
| Booking flow | The complete five-step customer journey in a real browser, including every validation refusal |
| Booking desk | All six queues and every action — approve, decline, payment link, deposit received, confirm, reschedule, a refused clash, cancel |
| Passkey | Enrolment and sign-in with genuine cryptographic signatures, via a virtual authenticator |
| Notification | Success, provider failure, missing configuration, and duplicate suppression |
| Unconfigured deployment | That a system without its calendar connection says so plainly and refuses to take a booking it cannot honour |
| Public site | Rendering, interaction and motion at three screen widths and under reduced-motion settings |

The backend suites run the **real functions against a real SQL database**
rather than mocks. That decision found two genuine production bugs that
mock-based tests would have passed over.

---

## What is deliberately not described here

Hostnames, resource identifiers, binding names, API endpoint paths, the
administrative URL, table-level column definitions, authentication
implementation details, and every configuration value. They are not necessary
to evaluate the design, and publishing them would weaken a live system.
