---
title: "An Alt-Svc Parameter and SvcParamKey for QUIC Versions"
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

Similarly, clients can retrieve additional instructions about access to services
or resources via DNS SVCB and HTTP Resource Records. This document also defines
a new SvcParamKey for these Resource Records, which specifies the specific QUIC
versions in use.

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
the client that it can use HTTP/3 {{?RFC9114}} for access, which in turn
requires the QUIC transport protocol {{?RFC8999}}.

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

Domain Name System (DNS) Service Binding (SVCB) and HTTPS Resource Records
{{!I-D.ietf-dnsop-svcb-https}} allow the distribution of access instructions
beyond the IP address via DNS. This document also specifies a new SvcParamKey
for these Resource Records to distribute QUIC version information with this
technique.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the Augmented BNF defined in {{!RFC5234}} and imports
`parameter` from {{Section 3 of ALTSVC}}.

# The quicv Parameter

This document specifies the "quicv" Alt-Svc parameter, which lists the QUIC
versions supported by an endpoint, using the hexadecimal representation of the
version field in a QUIC long header, as indicated in {{RFC8999}}. Senders MAY
omit leading zeroes from version numbers.

~~~ abnf
quicv         = version-list
version-list  = DQUOTE version 1*( OWS, "," OWS version-number) DQUOTE
version = 1*8 HEXDIG; hex-encoded QUIC version
~~~

Examples:

~~~
Alt-Svc: h3=":443"; quicv="1"
Alt-Svc: h3=":443"; quicv="709a50c4,1"
Alt-Svc: h3=":443"; quicv="709a50c4,1", h3=":1001"; quicv="709a50c4"
~~~

The order of entries in version-list reflects the server's preference (with the
first value being the most preferred alternative).

Note that the quicv parameter applies to a single associated entry in the
Alt-Svc list. Servers MUST NOT provide a quicv parameter to an entry containing
ALPN codepoint that does not potentially utilize QUIC.

If the Alt-Svc information resolves to a server pool that inconsistently
supports different QUIC versions, the parameter SHOULD only advertise versions
that are supported throughout the pool.

# The quicv SvcParamKey

SVCB and HTTPS Resource Records can include the quicv SvcParamKey. Its
presentation format value and use are identical to the quicv Alt-Svc Parameter.
Its wire format value consists of the version numbers in network byte order.

To include the quicv SvcParamKey in a resource record, it MUST also include at
least one ALPN that can be delivered over QUIC.

For example, consider a service configuration that advertisees two QUIC versions
on the default port, but only one version on a non-default port.

In Alt-Svc, this could be represented as:

~~~
Alt-Svc: h3=":443"; quicv="709a50c4,1", h3=":1001"; quicv="709a50c4"
~~~

As HTTPS RRs, this could be represented as:

~~~
example.com IN HTTPS 1 . alpn=h2,h3 quicv=709a50c4,1
example.com IN HTTPS 1 . alpn=h3 port=1001 quicv=709a50c4
~~~

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

Please add this entry to the HTTP Alt-Svc Parameter Registry:

Alt-Svc Parameter: quicv

Reference: This document

Please add this entry to the Service Binding (SVCB) Parameter Registry:

Number: TBD

Name: quicv

Meaning: Supported QUIC versions

Format Reference: This document

# Acknowledgments
{:numbered="false"}

Thanks to Ben Schwartz for his help with the Resource Record formatting.


--- back

# Change Log

> **RFC Editor's Note:**  Please remove this section prior to
> publication of a final version of this document.

## since draft-duke-httpbis-quic-version-alt-svc-01

* No changes

## since draft-duke-httpbis-quic-version-alt-svc-00

* Added SVCB and HTTPS Resource Records

