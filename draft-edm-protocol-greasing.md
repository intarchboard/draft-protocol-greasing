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
long-term viability of protocol extension points. Greasing is suitable for many
protocols but not all; {{Section 3.3 of ?VIABILITY=RFC9170}} discusses the
applicability and limitations of greasing. {{grease-considerations}} provides
additional protocol maintenance considerations.

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

# Considerations for Applying Greasing {#grease-considerations}

Greasing can take many forms, depending on the protocol and the nature of its
extension points.

In cases where a protocol uses registered values (i.e. codepoints) or numbers in
a well defined range, a common approach (see {{GREASE}}, {{Section 18.1 of QUIC}},
or {{Section 7.2.8 of RFC9114}}), is to reserve a subset of the range for the
purposes of greasing. This approach is detailed more thoroughly in {{Section 3.3
of ?VIABILITY=RFC9170}}. However, protocol designers or implementers may find it
difficult to apply those suggestions in abstract. The likely success or
efficacy of this method can be improved by the following suggestions.

It is assumed that endpoint should implement robust and broad extension
handling. When acting as a receiver or a parser, the implementation should not
treat codepoints reserved for the purposes of greasing as individually special.
In other words, rather than implementation looking specifically for reserved
values, it is better to have a "catch all" mechanism that can handle receipt of
unknown extensions gracefully or without error.

In order to exercise receiver capability, it is advisable that senders send
values from the ranges reserved for greasing. However, picking a deterministic
value risks a value becoming ossified itself. One outcome of that is receivers
being written to handle a single expected value rather than the generic handling
described above. One way to help mitigate this is to reserve a sufficiently
large range of values for greasing, and ensure that senders chose values from
that range with diversity and non-determinism. The specific nature of size and
distribution of the grease range needs to accommodate the protocol constraints.
For instance, an 8-bit field can only represent a small range of values and it
may be too costly to dedicate many of them solely for the purpose of greasing.
However, protocols that use 32-bit or 64-bit fields are unlikely to have such
restrictions.

It is beneficial to have a large set of reserved numbers for the purpose of
greasing. However, protocol designers that wish to do so may encounter
difficulties in expressing the large range in their protocol documents and/or in
an IANA registry. One approach to this problem has been to define the set
algorithmically in the protocol definition and request that IANA reserve only a
single entry in the respective table that covers the entire range; see for
example
https://www.iana.org/assignments/http3-parameters/http3-parameters.xhtml#http3-parameters-frame-types.
This range should be protected from registering from any other purpose. Deciding
an appropriate label for this protected range is important. Labelling it simply
"reserved" may be confused with other ranges that are reserved for private or
experimental extensions. An implementer that conflates these two meanings may
cause a new class of interoperability failure. Therefore a label such as
"reserved for greasing" may help to avoid the confusion. If choosing to use an
algorithm to define the set, it is encourage to use unique algorithms. This
again improves the chances that receivers will build robust extension handling
rather that a simple special-case ignore list.

Protocols that do reserve ranges for greasing introduce a new consideration for
real extensions. It is common to want to reserve a block of code points for
iteration and experimentation. Depending how the algorithm spreads numbers
through the full range, any single block of uninterrupted values may be too
small to be usable. This could lead to unintentional use of a greased value.

Since it is intended for receivers to ignore values reserved for greasing,
designers and implementers need to remain aware that unintentional use of
greased values by a sender for a real extension may cause a failure.

Receiver implementations may unintentionally build a reliance on the occurrence of
greasing in the temporal or spatial domain. Senders are advised to
implementation non-determinism of when they use grease in addition to what
values they send.

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
fingerprinting. Any techniques that change parameters  in any way, including but
not limited to those discussed in this document, can affect fingerprinting. A
deeper analysis of this topic has been deemed out of scope.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

This work is a summary of the topics discussed during EDM meetings. The
contributors at those meetings are thanked.
