---
title: "An Alt-Svc Parameter for QUIC Versions"
category: std

docname: draft-duke-httpbis-quic-version-alt-svc-latest
area: "Applications and Real-Time"
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

The HTTP Alternative Services (Alt-Svc) header informs clients of alternate
locations of the same resource. Among its fields, it includes the Application
Layer Protocol Negotiation (ALPN) codepoint of those locations. When the ALPN
is "h3", that indicates to the client that the resource can be accessed via the
QUIC protocol. However, without a priori knowledge of the QUIC version in use,
clients might incur a round-trip latency penalty to complete QUIC version
negotiation, or forfeit desirable properties of a QUIC version. This document
specifies a new parameter for the Alt-Svc header that specifies the supported
QUIC versions at the alternate location, substantially reducing the chance of
this penalty.


--- middle

# Introduction

The HTTP Alterative Services field {{!ALTSVC=I-D.ietf-httpbis-rfc7838bis}}
allows an HTTP server to inform the client of other locations to access a
resource, including via a different protocol. A client might connect using the
protocol with the highest probability success, but prefer the properties of
another. Note that  HTTP/2 {{?I-D.ietf-httpbis-http2bis}} recommends servers
send Alt-Svc in an ALTSVC frame, although the field is valid.

In particular, Alt-Svc includes a codepoint from the Application-Layer Protocol
Negotiation (ALPN) TLS extension {{?RFC7301}}. Originally intend to allow
multiple applications to utilize TLS or DTLS on the same IP address and TCP or
UDP port, ALPN can also usefully identify the transport in an Alt-Svc context.
The "h3" ALPN informs the client that it can use HTTP/3 {{?I-D.ietf-quic-http}}
for access, which in turn requires the QUIC transport protocol {{?RFC8999}}.

QUIC is versioned. A client and server that both support a QUIC version can,
through a negotiation process, geenrally agree on that version in no more than
one round-trip. However, to avoid that penalty clients might use the most
commonly deployed QUIC version (e.g. version 1 {{?RFC9000}}), rather than the
version with the most desirable properties for the client's use case.

One solution is to register new ALPN codepoints for new QUIC versions. However,
this might complicate deployment of new versions and deprecation of old ones:
architecturally, an application should provide its ALPN to its QUIC
implementation. In this case, fully deploying a new version in that
implementation would require updating all applications that use it.

Instead, this document specifies an Alt-Svc parameter that lists the QUIC
versions available to serve the resource. Clients that do not understand this
parameter might default to the most likely version, and/or incur a round-trip
penalty in the event of a mismatch. Clients that do process the parameter will
connect successfully using the most desirable version with high probability.


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
