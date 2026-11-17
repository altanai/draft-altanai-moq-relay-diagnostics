---
title: "Client-Visible Relay Diagnostics for Media over QUIC Transport (MOQT)"
abbrev: "MOQT Relay Diagnostics"
category: info
docname: draft-altanai-moq-relay-diagnostics-latest
submissiontype: IETF
date: 2026-08-10
consensus: false
v: 1
area: "Web and Internet Transport"
keyword:
  - MOQ
  - MOQT
  - relay
  - diagnostics
  - observability

author:
  -
    fullname: "Altanai Bisht"
    organization: "Cisco"
    email: "altanai@outlook.com"

normative:
  RFC2119:
  RFC8174:

informative:
  MoQTransport:
    title: "Media over QUIC Transport"
    author:
      org: "IETF MOQ WG"
    date: 2026
    target: "https://datatracker.ietf.org/doc/draft-ietf-moq-transport/"
  MoQMetrics:
    title: "Metrics over MOQT"
    author:
      name: "C. Jennings"
    date: 2025
    target: "https://datatracker.ietf.org/doc/draft-jennings-moq-metrics/"
  QLOG:
    title: "Main Logging Schema for qlog"
    author:
      org: "IETF QUIC WG"
    date: 2025
    target: "https://datatracker.ietf.org/doc/draft-ietf-quic-qlog-main-schema/"
  Issue1692:
    title: "moq-transport issue #1692: client-visible relay diagnostics"
    date: 2026
    target: "https://github.com/moq-wg/moq-transport/issues/1692"
---

--- abstract

Media over QUIC Transport (MOQT) deployments typically place one or more relays between publishers and subscribers. When delivery stalls, degrades, or fails, a client observes only its own leg of the path, and often cannot determine whether the cause lies with the original publisher, an intermediate relay, path conditions, authorization policy, or its own behavior.

This document collects use cases and requirements for exposing a limited, relay-controlled set of diagnostic information to authorized MOQT clients. It examines three candidate mechanisms: an explicit request and response exchange on the control path, diagnostic information carried in existing responses and error codes, and advertisement of diagnostic capability during session establishment. Because such information can reveal deployment topology, operational state, or the activity of other users, authorization, data minimization, and relay discretion over what is disclosed are treated as first-order requirements rather than later refinements.

The goal is diagnostics that are actionable at runtime and interoperable across implementations, not a general-purpose telemetry channel within MOQT. Information intended primarily for offline analysis remains the province of logging and observability systems such as qlog. This document defines the problem space and the design constraints; a concrete wire format is left to a follow-on document once the working group has settled on an approach.

--- middle

# Introduction

Media over QUIC Transport (MOQT) [MoQTransport] distributes media through a
publish/subscribe model in which relays sit between publishers and subscribers.
A subscriber therefore terminates only the last hop of a multi-hop path, and the
signals available to it locally are rarely sufficient to explain why delivery is
degraded or has failed.

This work originated in [Issue1692] and in discussion following IETF 126. Its
purpose is to establish the use cases and requirements first, so that any wire
format that follows is justified by a concrete operational need.

## Note to Readers

This revision is a problem statement and requirements document. It defines no
protocol elements. Comments are welcome on the MOQ working group mailing list.

## Conventions and Definitions

{::boilerplate bcp14-tagged}

# Problem Statement

MOQT endpoints can observe events on their own sessions, including the
messages they send and receive and the objects delivered to them. A subscriber
does not automatically observe the state of a relay's other sessions, its
upstream path, or the policy decisions made inside the relay. A publisher has a
similarly limited view of downstream relay and subscriber behavior. Encryption
and relay boundaries are intentional parts of the architecture; diagnostics
must not assume that an endpoint can inspect every hop.

As a result, a client may know that an object did not arrive before an
application deadline, but not whether the relay received it, whether the relay
was able to obtain it upstream, whether local policy prevented forwarding, or
whether the object was delayed or lost on the client-facing path. The same
visible symptom can have causes at different points in the delivery chain.
Some causes are also outside the relay's knowledge: a relay cannot reliably
describe a publisher's internal state or a network segment it does not
observe.

The problem is therefore not to expose a complete view of relay internals. It
is to let an authorized client obtain a small, carefully scoped explanation of
the relay's own handling of a request or delivery when that explanation can
help the client or operator choose a next action. A diagnostic is an
observation reported by one relay. It is not a proof of end-to-end delivery,
an attestation of another endpoint's behavior, or a guarantee that the relay
has complete information.

Diagnostics can also create risk. A response that reveals relay identity,
upstream relationships, cache contents, load, policy details, or other
clients' activity can assist traffic analysis, targeted attacks, or
cross-tenant inference. A mechanism that permits unlimited queries can add
load to a relay or become a reflection or probing facility. Privacy and
resource limits are therefore central design constraints.

# Use Cases

The following use cases describe the question a client or operator wants to
answer. They do not require a relay to disclose every fact that could help
answer it. In each case, a relay may provide a coarse, request-scoped result,
withhold details, or report that it cannot determine the cause.

## Distinguishing a Local Request Outcome

A subscriber issues a request for a track or a range of content and receives
an error or observes no useful progress. The application needs to distinguish
between a request that was rejected or could not be served at this relay and a
request that was accepted but has not yet produced content. A useful answer
might identify the outcome category and whether retrying the same request is
likely to help. It need not expose the relay's policy rules, cache keys, or
internal request identifiers.

## Diagnosing a Delivery Stall

A subscriber has an active subscription, but expected objects are not arriving
within the application's tolerance. The subscriber or its support tooling
wants to know whether this relay has observed relevant upstream content, has
forwarded content downstream, or cannot determine where progress stopped. A
bounded summary for that subscriber's request can guide whether to wait, retry,
change tracks, or collect local evidence. It should not reveal other
subscribers, shared cache contents, or the identities of upstream peers.

## Explaining a Relay Decision

A relay may decline, defer, or limit work for a request because of its local
authorization or resource policy. An authorized client wants an explanation
that is actionable, such as whether the failure is likely to persist without
a change in credentials or request parameters. A relay can communicate a
coarse outcome without disclosing confidential policy thresholds, tenant
configuration, load measurements, or details that would help an attacker tune
requests around policy.

## Supporting an Operator-Assisted Incident

A user reports a delivery problem and an operator needs to correlate the
client's report with the relay's handling of that user's request. A
request-scoped diagnostic summary or a short-lived correlation value can help
the user and operator discuss the same event. The value should not be a
globally stable client identifier, and it should not enable one client to
query another client's activity. More detailed logs may be available to the
service operator through a separate, access-controlled observability system.

## Learning Whether Diagnostics Are Available

A client may need to know whether a relay supports a diagnostic feature before
requesting it. A capability indication can prevent unsupported requests, but
it should reveal only the feature category needed for interoperability. It
should not fingerprint a relay implementation or disclose deployment
topology.

# Requirements

These requirements describe design goals for a future mechanism. They do not
specify MOQT messages or require relays to expose operational information.

## Diagnostic Scope

* A mechanism needs to identify the request or delivery context to which a
  diagnostic applies, without requiring a stable identifier that can be used
  to correlate a client's activity across sessions.
* A diagnostic needs to distinguish, where the relay has sufficient evidence,
  between a relay-observed outcome and an unknown or unobserved cause.
* A mechanism needs to permit an implementation to return no diagnostic or a
  less detailed result when the relay lacks reliable evidence.
* A diagnostic needs to avoid implying facts about upstream endpoints or
  network segments that the reporting relay cannot observe.

## Privacy and Authorization

* A relay needs to authenticate and authorize access to diagnostics according
  to its deployment's policy. A client authorized to request content is not
  automatically authorized to inspect general relay health or other clients'
  activity.
* Diagnostic responses need to be scoped to the requesting client's own
  session and request, except where a separately defined and authorized
  operator interface is used.
* A mechanism needs to support data minimization: disclose the least
  information useful for the stated diagnostic purpose, and allow the relay to
  omit sensitive fields.
* Diagnostics need not disclose relay topology, upstream identities, cache
  keys or contents, tenant names, policy rules, precise resource utilization,
  or information about other clients.
* Any correlation value needs to be scoped, short-lived, and difficult to use
  as a cross-session or cross-service tracking identifier.
* A client needs to be able to distinguish unsupported, unavailable, and
  withheld information where doing so does not itself create an unacceptable
  information leak. A mechanism can deliberately collapse these outcomes when
  policy requires it.

## Safety and Operational Behavior

* A mechanism needs bounded response size and bounded processing cost.
* A relay needs discretion to rate-limit, defer, or refuse diagnostic requests
  without affecting ordinary content delivery.
* Diagnostic requests need to avoid causing new upstream subscriptions,
  cache fills, or other expensive work solely to generate a diagnostic.
* A diagnostic needs to be clearly identified as a point-in-time report from
  one relay. It may be incomplete, delayed, or based on an approximation.
* Diagnostics need not be enabled by default. Capability discovery and use
  should be compatible with deployments that choose not to expose them.
* Clients need to treat diagnostic information as advisory and continue to
  apply their own timeouts, retry limits, and security policy.

# Candidate Mechanisms

## Explicit Request and Response on the Control Path

An explicit request lets a client ask for a diagnostic when a particular
request is stalled or has failed. It can carry a request-scoped context and
allow the relay to return a result only when asked. This avoids sending
diagnostics for every request and gives the relay an opportunity to apply
authorization and rate limits.

The mechanism also adds a new request/response exchange and associated state.
It needs rules for matching a diagnostic to a request, handling unavailable
or withheld information, and preventing repeated queries from becoming a
resource or probing attack. A diagnostic request must not trigger expensive
upstream work solely to answer the query.

## Diagnostic Information in Existing Responses and Error Codes

Adding a small diagnostic category to an existing response or error can make
an outcome immediately actionable and avoid a separate round trip. It is a
natural fit when the relay is already returning a result for the request.

This approach is less suitable for later investigation of an ongoing stall,
and risks turning response fields into a channel for detailed operational
data. Error details may also be propagated or cached by relays. Any details
would therefore need to remain compact, request-specific, privacy-preserving,
and safe to forward or retain for the lifetime defined by the base protocol.

## Capability Advertisement During Session Establishment

Capability advertisement can tell a client whether a relay understands a
diagnostic mechanism before the client uses it. It can support gradual
deployment and avoid sending requests that cannot be understood.

An advertisement says that a capability is supported; it does not authorize a
particular client or guarantee that a relay will disclose a diagnostic for a
particular request. Detailed capability lists can fingerprint implementations
or deployments, so discovery should identify only the protocol feature needed
for interoperability.

The mechanisms are not necessarily exclusive. A later design could use
capability discovery together with either an explicit query or compact
response details. Selection requires working-group agreement on the target
use cases and privacy model.

# Relationship to Other Work

This work concerns request-scoped explanations that can help an endpoint make
a runtime decision or help a user report a problem. It is not a general
metrics format, a relay management API, or a logging protocol.

[MoQMetrics] concerns metrics carried over MOQT. A metrics stream can expose
ongoing measurements for an authorized purpose; the use cases here instead
focus on a limited explanation associated with a particular request or
delivery symptom. The working group should ensure that any future design
complements metrics work and does not create overlapping ways to export the
same operational data.

[QLOG] and similar observability systems are useful for detailed event
recording and offline analysis. They are generally configured and consumed by
operators or developers, and are not a substitute for a minimal runtime
answer to a client. This document does not define how qlog is collected,
stored, or shared.

The following are out of scope: exposing a relay's complete internal state;
distributing network-wide or cross-tenant telemetry to ordinary clients;
proving end-to-end delivery; standardizing operator dashboards; and defining
a general-purpose query language for relay state.

# Security Considerations

Relay diagnostics create a deliberate information channel from a relay to a
client. The channel can reveal operational state, deployment relationships,
traffic patterns, and policy decisions. An attacker may use repeated or
carefully varied requests to infer whether content is cached, whether another
client is active, how a tenant is configured, or when a relay is under load.
Even coarse fields can become identifying when combined with timing and
client-side observations.

A future protocol design should apply data minimization at the relay, not rely
only on clients to protect received data. Diagnostics should be tied to the
requesting client's authorization context and scoped to that client's own
request. General relay health, cross-client statistics, and operator
correlation should use separately authorized interfaces. Deployments may
choose not to provide client-visible diagnostics or may provide only a small
set of coarse outcomes.

Implementations should avoid returning stable relay or client identifiers,
precise timestamps, exact cache keys, upstream addresses, internal error text,
or raw policy and resource measurements unless a separate, explicit trust
model justifies them. Any event correlation value should be short-lived and
non-reusable across sessions. A relay may deliberately make "unknown",
"withheld", and "not available" indistinguishable if differentiating them
would expose sensitive policy or state.

Diagnostic queries can consume CPU, memory, or control-stream capacity, and
may be abused to probe policy or amplify traffic. Responses and processing
need limits; relays need rate limiting and discretion to refuse queries.
Generating a diagnostic should not trigger content retrieval, cache filling,
or upstream subscription work. Where diagnostic messages can be replayed,
cached, or propagated by other relays, their scope and lifetime need to be
defined so that one client's result is not disclosed to another.

Clients should treat diagnostics as advisory, potentially incomplete
observations. They should not infer that a named upstream component is at
fault based solely on a relay response, and should not use the absence of a
diagnostic as proof that delivery is healthy. Operators should consider the
privacy of diagnostic data in logs, support tickets, and telemetry pipelines.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

The author thanks the participants in the Media over QUIC Transport working
group for discussion of relay diagnostics and issue #1692. Additional
acknowledgments will be added as the document develops.
