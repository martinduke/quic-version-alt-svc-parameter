---
title: "An Alt-Svc Parameter for QUIC Versions"
category: std

docname: draft-duke-httpbis-quic-version-alt-svc-latest
area: "Applications and Real-Time"
ipr: trust200902
workgroup: "HTTP"
keyword: Internet-Draft
stand_alone: yes
venue:
  group: "HTTP"
  type: "Working Group"
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "martinduke/quic-version-alt-svc-parameter"
  latest: "https://martinduke.github.io/quic-version-alt-svc-parameter/draft-duke-httpbis-quic-version-alt-svc.html"

author:
 -
    fullname: Martin Duke
    organization: Google
    email: martin.h.duke@gmail.com

 -
    fullname: Lucas Pardue
    organization: Cloudflare
    email: lucaspardue.24.7@gmail.com

normative:


informative:


--- abstract

HTTP Alternative Services (Alt-Svc) describes how one origin's resource can be
accessed via a different protocol/host/port combination. Alternatives are
advertised by servers using the Alt-Svc header field or the ALTSVC frame. This
includes a protocol name, which reuses Application Layer Protocol Negotiation
(ALPN) codepoints. The "h3" codepoint indicates the availability of HTTP/3. A
client that uses such an alternative first makes a QUIC connection. However,
without a priori knowledge of which QUIC version to use, clients might incur a
round-trip latency penalty to complete QUIC version negotiation, or forfeit
desirable properties of a QUIC version. This document specifies a new Alt-Svc
parameter that specifies alternative supported QUIC versions, which
substantially reduces the chance of this penalty.


--- middle

# Introduction

HTTP Alternative Services (Alt-Svc) {{!ALTSVC=I-D.ietf-httpbis-rfc7838bis}}
describes how one origin's resource can be accessed via a different
protocol/host/port combination. Alternatives are advertised by servers using the
Alt-Svc header field or the ALTSVC frame. This includes a protocol name, which
reuses codepoints from the Application-Layer Protocol Negotiation (ALPN) TLS
extension {{?RFC7301}}. Servers can advertise multiple alternatives, in which
case the order reflects the server's preferences (the first value being the most
preferred).

Clients can ignore alternative services, or pick one at their discretion. A
client might use any details from the advertisement, in addition to out of
band information, in determining if an alternative is suitable or preferred.

While ALPN was originally intend to allow multiple applications to utilize TLS
or DTLS on the same IP address and TCP or UDP port, ALPN can also usefully
identify the transport in an Alt-Svc context. The "h3" ALPN codepoint informs
the client that it can use HTTP/3 {{?I-D.ietf-quic-http}} for access, which in
turn requires the QUIC transport protocol {{?RFC8999}}.

QUIC is versioned. A client and server that both support a QUIC version can,
through a negotiation process, generally agree on that version in no more than
one round-trip. However, to avoid that penalty clients might use the most
commonly deployed QUIC version (e.g. version 1 {{?RFC9000}} at the time of
writing), rather than the version with the most desirable properties for the
client's use case.

To avoid the round-trip, one solution would be to register unique ALPN
codepoints for each HTTP/3 and QUIC version combination. However, this might
complicate deployment of new versions and deprecation of old ones:
architecturally, an application should provide its ALPN to its QUIC
implementation. In this case, fully deploying a new version in that
implementation would require updating all applications that use it.

Instead, this document specifies an Alt-Svc parameter that lists the QUIC
versions available to serve the resource. Clients that do not understand this
parameter will ignore it. They might default to the most likely version, and/or
incur a round-trip penalty in the event of a mismatch. Clients that do process
the parameter will connect successfully using the most desirable version with
high probability.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the Augmented BNF defined in {{!RFC5234}} and structured
fields defined in {{!RFC8941}}.

# The quicv Parameter

This document specifies the "quicv" parameter, which lists the QUIC versions
supported by an endpoint.

```
parameter       = param-key "=" param-value
param-key       = "quicv"
param-value     = version 1*( OWS, "," OWS version)
version         = 8(HEXDIG)
```

The "version" field is the hexadecimal representation of the version field in a
QUIC long header, as indicated in {{RFC8999}}.

Note that each parameter applies to a single entry in the Alt-Svc list. Servers
MUST NOT append a quicv parameter to an ALPN that does not potentially utilize
QUIC.

If the Alt-Svc information resolves to a server pool that inconsistenly supports
different QUIC versions, the parameter SHOULD only advertise versions that are
supported throughout the pool.

# Security Considerations


This document inherits the security considerations of {{ALTSVC}}, especially
the implications of "Changing Protocols" in Section 9.3. There are few
protocol properties guaranteed to hold across all QUIC versions, so endpoints
should be aware what capabilities are intrinsic to the QUIC versions they are
advertising.

This parameter reveals capabilities of the described server, but this
information is already available by inducing the server to generate a QUIC
version negotiation packet.


# IANA Considerations

Please add this entry ot the HTTP Alt-Svc Parameter Registry:

Alt-Svc Parameter: quicv

Reference: This document


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
