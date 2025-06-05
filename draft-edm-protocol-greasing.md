---
title: "Maintaining Protocols Using Grease and Variability"
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

normative:

informative:
  RFC9114:
    display: HTTP/3

--- abstract

Long-term interoperability of protocols is an important goal of the network
standards process. Deployment success can depend on supporting change, which
can include modifying how the protocol is used, extending the protocol, or
replacing the protocol. This document presents concepts, considerations, and
techniques related to protocol maintenance, such as greasing or variability. The
intended audience is protocol designers and implementers.


--- middle

# Introduction

Long-term interoperability of protocols is an important goal of the network
standards process {{?MAINTENANCE=RFC9413}}. Protocol deployment success
{{?SUCCESS=RFC5218}} can depend on supporting change, which
can include modifying how the protocol is used, extending the protocol, or
replacing the protocol.

Greasing, a technique initially designed for TLS {{?GREASE=RFC8701}} and later
adopted by other protocols such as QUIC {{?QUIC=RFC9000}}, can help support the
long-term viability of protocol extension points. In these protocols, extension
codepoints are reserved only for greasing and when received must be ignored.
Greasing is suitable for many protocols but not all; {{Section 3.3 of
?VIABILITY=RFC9170}} discusses the applicability and limitations of greasing.
{{grease-considerations}} provides additional protocol maintenance
considerations.

Applications are built with the intent of serving user needs {{?END-USERS=RFC8890}}, which might only
require support for a subset of protocol features. Adapting to changing user
needs is a maintenance activity. For example, an HTTP deployment focused on
downloads might want to add support for uploads. Changing use of the application
and transport protocol features can affect the deployment's network traffic
profile. If expectations have been formed around historical patterns of use
i.e., ossification, introducing change might lead to deployment problems. {{variability}} presents
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


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Considerations for Greasing Protocols {#grease-considerations}

Greasing can take many forms, depending on the protocol and the nature of its
extension points. The common pattern across forms of greasing is that values
are generated that have no useful meaning to the protocol and are meant to
be ignored upon receipt. Such values used for the purpose of greasing are
referred to as "grease values" within this document.

More background to this approach is given in {{Section 3.3 of ?VIABILITY=RFC9170}}.

This section provides some practical considerations for how to define and
use greasing, and avoid possible pitfalls.

## Define and Register Grease Value Ranges

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
receiver greasing requirements are exercised.

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

IANA registries that contain reserved grease values must indicate that
the values are reserved. The specifics of how to represent the reservations
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
should be large. However, the specific size and distribution
of the grease range needs to accommodate the protocol constraints. For instance,
protocols that use 8-bit fields may find it too costly to dedicate many grease
values, while 32-bit or 64-bit fields are likely to have no such limitations.

## Use Grease Values Unpredictably

In addition to selecting unpredictable values, the inclusion of grease itself
can be made unpredictable. Implementations can vary their behavior by including
no grease values, one grease value, or multiple grease values for a given protocol
extension point.

TODO: Discuss the implications of how often grease values are used; using very
infrequently and very frequently can both be ossifying

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

Implementations need to take care when implementing such logic. Protocol specification
designers should emphasize that grease values must not be special-cased. It is also
recommended to provide example logic or pseudocode in specifications as guidance to
implementers on how to correctly process protocol elements like these.

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

The considerations in {{MAINTENANCE}}, {{GREASE}}, {{END-USERS}}, and
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
