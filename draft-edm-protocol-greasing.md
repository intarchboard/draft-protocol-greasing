---
title: "Considerations For Maintaining Protocols Using Grease and Variability"
abbrev: "Protocol Greasing"
category: info

docname: draft-edm-protocol-greasing-latest
submissiontype: IAB
number:
date:
consensus: true
v: 3
# area: AREA
# workgroup: WG Working Group
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
  github: "intarchboard/draft-protocol-greasing"
  latest: "https://intarchboard.github.io/draft-protocol-greasing/draft-edm-protocol-greasing.html"

author:
 -
    fullname: Lucas Pardue
    organization: Cloudflare
    email: "lucas@lucaspardue.com"
 -
    fullname: Tommy Pauly
    organization: Apple
    email: "tpauly@apple.com"

normative:

informative:
  RFC9114:
    display: HTTP/3

--- abstract

Active use and maintenance of network protocols is an important way to
ensure that protocols remain interoperable and extensible over time.
Techniques such as intentionally exercising extension points with
non-meaningful values (referred to as "grease") or adding variability
to how protocol elements are used help generate this active use.

Grease and variability are used across various protocols developed
by the IETF. This document discusses considerations when designing
and deploying grease and variability mechanisms, and provides
advice for making them as effective as possible.

--- middle

# Introduction

{{Section 3 of ?VIABILITY=RFC9170}} discusses "active use" as a category
of techniques that protocol designers and implementers employ to ensure
that protocol extension mechanisms are exercised and can be used in the
future. This ability to change (to handle protocol updates and extensions)
is an important factor in the success of protocol deployment, as discussed
in {{?SUCCESS=RFC5218}}.

Active use of protocol features and extensions often requires intentional
efforts beyond what would organically occur in deployments. Some extension
points do not frequently see new values being used, but are still important
to be usable in the future. Some patterns of protocol usage might be
relatively static without specific efforts to ensure that they can change
in the future.

One key techique for intentional use is "grease", or "greasing".
Greasing was initially designed for TLS {{?GREASE=RFC8701}} and was later
adopted by other protocols such as QUIC {{?QUIC=RFC9000}}. In these protocols,
extension codepoints are reserved only for greasing and must be ignored
upon receipt. Greasing is suitable for many protocols but not all; {{Section 3.3 of
?VIABILITY=RFC9170}} discusses the applicability and limitations of greasing.

While it is becoming more common, designing and applying grease is not
necessarily trivial. There are best practices, and some common pitfalls to
avoid, that have been developed by the protocols using grease thus far.
{{grease-considerations}} takes these learnings and provides considerations
for new protocol design and deployment.

Separate from greasing using reserved values, protocol deployments can
intentionally add variability to ensure that network endpoints and
middleboxes do not end up ossifying certain patterns. For example, an HTTP
deployment focused on downloads might want to add support for uploads.
Changing use of the application and transport protocol features can affect
the deployment's network traffic profile. If expectations have been formed
around historical patterns of use, i.e., ossification, introducing change
might lead to deployment problems. {{variability}} presents
considerations about how intentionally increasing the variability of protocols
can mitigate some of these concerns.

Protocol extensions can provide longevity in the face of changing needs or
environment. However, a replacment protocol might be preferred when extensions
are not adequate or feasible. A protocol replacement could aggregate common
extensions and possibly make them mandatory, effectively defining a new baseline
that can simplify deployment and interoperability. A replacement protocol
version may or may not be compatible with other versions. A protocol may or may
not have a mechanism for version selection or agility. {{versions}} presents
considerations about designing for and/or implementing version negotiation and
migration.

# Considerations for Greasing Protocols {#grease-considerations}

Greasing can take many forms, depending on the protocol and the nature of its
extension points. The common pattern across forms of greasing is that values
are generated that have no useful meaning to the protocol and are meant to
be ignored upon receipt. Such values used for the purpose of greasing are
referred to as "grease values" within this document.

More background to this approach is given in {{Section 3.3 of ?VIABILITY=RFC9170}}.

This section provides some practical considerations for how to define and
use greasing, and avoid possible pitfalls.

## Define and Register Grease Value Ranges {#define}

Many protocols that use greasing have a limited set of possible values or
codepoints that can be used in a particular extension point. A common
approach is to reserve a subset of the registrable space for greasing.

The following are some examples of protocols that have reserved codepoints for grease values:

- TLS ({{GREASE}})
- QUIC ({{Section 18.1 of QUIC}})
- HTTP/3 ({{Section 7.2.8 of RFC9114}})
- Privacy Pass ({{?PRIVACYPASS=RFC9577}})

The specifics of how to reserve values depends on the nature of the
available space. For protocols with large possible spaces, it is useful
to have a large set of grease values to increase the chance that
receiver greasing requirements are exercised. The specific
size and distribution of the grease range needs to accommodate the
protocol constraints. For instance, protocols that use 8-bit fields may
find it too costly to dedicate many grease values, while 32-bit or 64-bit
fields are likely to have no such limitations.

It is recommended to use an algorithm to reserving large sets of values.
For example, {{QUIC}} uses and algorithm of `31 * N + 27` to allocate
transport parameters grease values.

One possible problem with some algorithms is how they will spread out
values over the space, and impact the ability to use or reserve contiguous
blocks of non-grease values. It is common for protocol extension designers
to want to reserve contiguous blocks of codepoints in order to aid
iteration and experimentation. Reserved grease values can end up being
in spaces that would otherwise be used for such contiguous blocks.

Codepoints being used for new reservations, or experimentation, need
to be careful to not unintentionally use grease values. Doing so could
lead to interoperability failures.

### Recommendations for IANA Considerations {#iana-tips}

IANA registries that contain reserved grease values need to indicate that
the values are reserved in order to prevent them from being allocated
for other uses. The specifics of how to represent the reservations
is up to the documents that define the registries.

Some registries list out the reserved grease values specifically, marked as
"Reserved". For example, the TLS registry uses this approach
(https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml).

If an algorithm or pattern is used to define grease values, it is recommended
when possible to instead only define a single entry for the entire grease value
set. This entry should include the pattern or algorithm. This approach is used by
the QUIC registry (https://www.iana.org/assignments/quic/quic.xhtml) and the
HTTP/3 registry (https://www.iana.org/assignments/http3-parameters/http3-parameters.xhtml#http3-parameters-frame-types)

Grease values must not be used or registered for any other purpose. This is in
contrast to other potentially "reserved" values that might be reused or
claimed for a new purpose in the future. To avoid confusion, it is recommended
to label the reservation with a clear identifier, such as "reserved for greasing".

## Use Unpredictable Grease Values

In order to gain the benefits of active use and avoid ossification, grease values
need to be sent in ways that won't become a predictable pattern that implementations
and deployments ossify around.

Implementations that generate grease values should pick unpredictable entries
from the set of reserved grease values. It is most important that values be
unpredictable across the set of all protocol participants for particular deployments.
This can be achieved in multiple ways: for example, an individual client device
might pick random values from the grease value space on each interaction;
alternatively, a single client could pick a specific grease value to use, while
other clients pick other values.

In order to support picking unpredictable values, the set of reserved values
should be large, when possible. See {{define}} for a discussion of how to allocate
grease values.

## Use Grease Values Unpredictably

In addition to selecting unpredictable values, the inclusion of grease itself
can be made unpredictable. Implementations can vary their behavior by including
no grease values, one grease value, or multiple grease values for a given protocol
extension point.

How consistently an frequently to use grease values is a choice that implementations
and deployments need to consider and weigh against several factors.

Deployments of greasing should consider how they expect errors exposed by
using grease values to be noticed and measured.

If grease values are sent too infrequently, so that errors due to sending
grease values blend in with the noise of other errors, it is likely that
no one will notice failures, thus defeating the purpose of greasing.
When grease values are sent more frequently, they will be noticed more.
However, if grease values are sent too consistently, receiver implementations
might end up special-casing grease values.

The patterns for sending grease values can be made more effective by
coordinating between devices sending the values. One example of coordination
is having a "flag day" where implementations start sending grease values
broadly, and measure to see where errors occur.

## Don't Handle Grease Values as a Special Case

Implementations that read and process grease values must ignore the values.
"Ignoring" a value upon receipt can have multiple dimensions, however.
Simply not performing any protocol action based on the grease value isn't
enough to ensure that the protocol will remain extensible. The ignoring
must be handled as a general case of "unknown" or "unhandled" values,
not as a special case for ignored grease values.

This means that grease values can only meaningfully be used for protocol elements
where all unknown values are ignored by default. (Protocols may have ways
to indicate that some specific values are "mandatory" or "critical" in order
to have the protocol interaction succeed, but these must not be marked
for any grease values.)

One pitfall that implementations may encounter when building logic to
handle the receipt of grease values is related to cases where some recognized
non-grease values need to be handled as errors. Consider the following abstract
example:

1. A protocol element has values 1, 2, 3, and 4 defined and registered for
use. The values 13 and 42 are reserved as grease values.
1. In a specific scenario, only the known values 1 and 2 are valid; 3 and 4
are considered errors.
1. An implementation might naively choose to check for the value being 1 or 2, handle
those cases, and send an error otherwise.
1. When grease values are used, the previous logic will flag an error for the grease value.
If this is detected, implementations might choose to work around this by updating
the logic to check for the value being 1 or 2, then check for grease values to ignore,
and then send an error otherwise. This logic is also incorrect since it doesn't
allow for new extensibility.

The correct logic for the above scenario would be to check for the value being 1 or 2,
then explicitly check for the value being 3 or 4 (to handle the error), with a
catch-all for ignored values.

In pseudo-code, the correct logic would work like this, where the grease values
would fall into the final `else` case as ignored values.

~~~
if is_valid_case_one(value):
  handle_case_one()
else if is_valid_case_two(value):
  handle_case_two()
else if is_known_invalid_case(value):
  handle_error()
else:
  ignore_value()
~~~

Implementations need to take care when implementing such logic. Protocol specification
designers should emphasize that grease values must not be special-cased. It is also
recommended to provide example logic or pseudocode in specifications, similar to the example
above, as guidance to implementers on how to correctly process protocol elements like these.
Documents can also provide test vectors, when applicable, that include grease values
to ensure they are processed correctly.

# Deployment Considerations and Incentives for Greasing

Greasing can be used as a tool to improve the active use of existing protocol
elements (which weren't necessarily designed with greasing to begin with, or
weren't deployed with greasing); or as part of new protocol design and deployments.

When greasing isn't used from the beginning of protocol deployment, starting to use
greasing comes with the risk of triggering failures or anomalies. These failures might be innocuous,
but they also might be very impactful and visible to users. This risk creates a
disincentive to deploy greasing in existing systems, since generally the change that
triggers failures is often blamed for the failure. The risk is highest when
adding greasing to a particular protocol flow that doesn't require any
change of behavior or adoption to hit the greasing behavior. For example,
if a service migrates to use a new web server implementation that enables
greasing, while the previous server didn't, some new failures may be hit
if clients react poorly to greasing.

Some approaches to avoid failures due to greasing include:

- Designing, implementing, and using greasing very early on in protocol development
and deployment. This avoids the aforementioned risk of adding greasing late in a deployment.
- Enabling greasing along with other major protocol feature changes or deployment changes.
For example, when upgrading to a new protocol version that requires implementation updates
on multiple systems, greasing can be added for the new version specifically. This approach
works well for situations where the protocol participants are known and already need
to cooperate (such as within an encrypted protocol between two endpoints). This approach
applies less well for situations where non-cooperating entities (such as middleboxes)
are the source of ossification.
- Using heuristics to disable greasing when errors are encountered. For example,
if a client-initiated protocol operation fails multiple times when grease values are used, it can be retried
without any grease values. Alternatively, if a server recognizes a property of a client that always fails
when greasing is used, it could choose to disable greasing when that client is detected.
This reduces the effectiveness of grease values in removing
existing ossification, but can still have benefits for flagging issues in new implementations
when they receive grease values.

# Considerations for Increasing Protocol Variability {#variability}

Greasing can maintain protocol extensibility by falsifying active use of its
extension points. However, greasing alone does not ensure positive use of extension mechanisms. A protocol may
define a wide-ranging extension capability that remains unused in the absence of
real use cases. This can lead to ossification that does not expect extensions,
leading to interoperability problems later on.

Long-term maintenance and interoperability can be ensured by exercising
extension points positively. To some extent this can be thought of as protocol
fuzzing. This might be difficult to exercise because varying the protocol
elements might change the outcome of interactions, leading to real errors.
However, some protocols allow elements to be be safely changed, as shown in the
following examples.

## Example: QUIC frames

QUIC packets contain frames. Receivers might build expectations on the
longitudinal aspects of packets or frames - size, ordering, frequency, etc. A
sender can quite often manipulate these parameters and stay compliant to the
requirements of the QUIC protocol.

A QUIC stream is an ordered reliable byte stream that is serialized as a
sequence of STREAM frames with a length and offset. Receivers are expected to
reassemble frames, which could arrive in any order, into an ordered reliable
byte stream that is readable by applications.

A form of positive testing is for a sender to unpredictably order the STREAM
frames that it transmits. For example, varying the sequence order of offset
values. This allows to exercise the QUIC reassembly features of the receiver
with the expectation that no failure would occur. However, doing this may
introduce delay or stream head-of-line blocking that affects the performance
aspects of a transmission, which may not be acceptable for a given use case. As
such, positive testing might be most appropriate to use in a subset of
connections, or phases within a connection.

# Considerations for Protocol Versions {#versions}

There are intrinsic and well-documented issues related to testing version
negotiation of protocols; see {{?EXTENSIBILITY=RFC6709}} and {{Sections 2.1 and
3.2 of VIABILITY}}. This section will be expanded with advice for protocol
designers and implementers about how to approach these problems.

# Security Considerations

The considerations in {{?MAINTENANCE=RFC9413}}, {{GREASE}}, {{?END-USERS=RFC8890}}, and
{{VIABILITY}} all apply to the topics discussed in this document.

The use of protocol features, extensions, and versions can already allow
fingerprinting {{?PRIVCON=RFC6973}}. Any techniques that change parameters  in any way, including but
not limited to those discussed in this document, can affect fingerprinting. A
deeper analysis of this topic has been deemed out of scope.


# IANA Considerations

This document has no IANA actions itself. Guidance on how other documents can effectively
instruct IANA about protocol greasing is provided in {{iana-tips}}


--- back

# Acknowledgments
{:numbered="false"}

This work is a summary of the topics discussed during EDM meetings. The
contributors at those meetings are thanked.
