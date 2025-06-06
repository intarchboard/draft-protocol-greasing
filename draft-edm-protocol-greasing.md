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
necessarily trivial. There are best practices and some common pitfalls to
avoid that have been developed by the protocols using grease thus far.
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


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Considerations for Applying Greasing {#grease-considerations}

Greasing can take many forms, depending on the protocol and the nature of its
extension points.

Many protocols register values, codepoints, or numbers in a limited space. A
common approach that has developed in more recent protocols is to reserve a subset of the space for greasing (see
{{GREASE}}, {{Section 18.1 of QUIC}}, or {{Section 7.2.8 of RFC9114}}). Values
reserved for the purpose of greasing are herein referred to as grease values.
Implementations that receive grease values are required to ignore them. More
background to this approach is given in {{Section 3.3 of ?VIABILITY=RFC9170}}.
This section provides concrete suggestions for its usage.

## Don't Handle Grease Values as a Special Case

It is assumed that endpoints should implement robust and broad extension
handling. A receiver or a parser implementation should not treat grease values
as individually special. Instead of identifying each grease value explicitly,
it is better to have a "catch all" mechanism that can handle receipt of unknown
extensions, whether grease values or not, gracefully or without error.

## Use Unpredictable Grease Values

It is recommended that senders pick an unpredictable grease value to include in
relevant protocol elements. This ensures that receiver greasing requirements are
exercised. Using predictable grease values risks ossification. To increase the
variety of grease values, it is advised to reserve a large range. However, the
specific size and distribution of the grease range needs to accommodate the
protocol constraints. For instance, protocols that use 8-bit fields may find it
too costly to dedicate many grease values, while 32-bit or 64-bit fields are
likely to have no limitations.

## Use Grease Values at Unpredictable Times

It is recommended that senders use grease values at unpredictable times or
sequence points during protocol interactions. This avoids receivers
unintentionally ossifying on the occurrence of greasing in the temporal or
spatial domain.

## Define and Register Grease Value Ranges

It is recommended that large grease value sets are allocated in protocol
documents by defining a unique algorithm, to increase the chance that
receiver greasing requirements are exercised. However, the choice of algorithm
needs to consider the spread of values and the size of contiguous blocks between
grease values. It is common for protocol extension designers to want to reserve
a contiguous block of code points in order to aid iteration and experimentation.
Small contiguous blocks increase the chance that such reservations might
unintentionally use grease values, which could lead to interoperability
failures.

### Effectively Instructing IANA about grease {#iana-tips}

Protocol designers might ask IANA to create new registries for their extension
points. When greasing, it is recommended that only a single entry for the entire
grease value set is registered. When an algorithm has been used, it should be
included in the entry; see for example
https://www.iana.org/assignments/http3-parameters/http3-parameters.xhtml#http3-parameters-frame-types.

Grease values must not be used or registered for any other purpose. Registries
should include a label to identify the protected grease value range; a label of
"reserved" may be confused with other ranges that are reserved for private or
experimental extensions. An implementer that conflates these two meanings may
cause a new class of interoperability failure. Therefore a label such as
"reserved for greasing" may help to avoid the confusion.


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
