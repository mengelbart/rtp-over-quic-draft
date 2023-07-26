---
title: "RTP over QUIC (RoQ)"
docname: draft-ietf-avtcore-rtp-over-quic-latest
category: std
date: {DATE}

ipr: trust200902
area: "Applications and Real-Time"
workgroup: "Audio/Video Transport Core Maintenance"
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]
submissiontype: IETF

author:
 -
    ins: J. Ott
    name: Jörg Ott
    organization: Technical University Munich
    email: ott@in.tum.de
 -
    ins: M. Engelbart
    name: Mathis Engelbart
    organization: Technical University Munich
    email: mathis.engelbart@gmail.com
 -
    ins: S. Dawkins
    name: Spencer Dawkins
    organization: Tencent America LLC
    email: spencerdawkins.ietf@gmail.com

informative:

  3GPP-TS-26.114:
    target: (https://portal.3gpp.org/desktopmodules/Specifications/specificationId=1404)
    title: "IP Multimedia Subsystem (IMS); Multimedia telephony; Media handling and interaction"
    date: 2023-01-05

--- abstract

This document specifies a minimal mapping for encapsulating Real-time Transport
Protocol (RTP) and RTP Control Protocol (RTCP) packets within the QUIC protocol.
This mapping is called RTP over QUIC (RoQ).
It also discusses how to leverage state from the QUIC implementation in the
endpoints, in order to reduce the need to exchange RTCP packets and how to
implement congestion control and rate adaptation without relying on RTCP
feedback.

--- middle

# Introduction

This document specifies a minimal mapping for encapsulating Real-time Transport
Protocol (RTP) {{!RFC3550}} and RTP Control Protocol (RTCP) {{!RFC3550}} packets
within the QUIC protocol ({{!RFC9000}}).
This mapping is called RTP over QUIC (RoQ).
It also discusses how to leverage state
from the QUIC implementation in the endpoints, in order to reduce the need to
exchange RTCP packets, and how to implement congestion control and rate
adaptation without relying on RTCP feedback.

## Background {#background}

The Real-time Transport Protocol (RTP) {{!RFC3550}} is generally used to carry
real-time media for conversational media sessions, such as video conferences,
across the Internet.  Since RTP requires real-time delivery and is tolerant to
packet losses, the default underlying transport protocol has been UDP, recently
with DTLS on top to secure the media exchange and occasionally TCP (and possibly
TLS) as a fallback.

This specification describes an application usage of QUIC ({{?RFC9308}}).
As a baseline, the specification does not expect more than a standard QUIC implementation
as defined in {{!RFC8999}}, {{!RFC9000}}, {{!RFC9001}}, and {{!RFC9002}},
providing a secure end-to-end transport that is also expected to work well through NATs and firewalls.
Beyond this baseline, real-time applications can benefit from QUIC extensions such as unreliable QUIC datagrams
{{!RFC9221}}, which provides additional desirable properties for
real-time traffic (e.g., no unnecessary retransmissions, avoiding head-of-line
blocking).

## Motivations {#motivations}

From time to time, someone asks the reasonable question, "why should anyone implement and deploy RoQ"? This reasonable question deserves a better answer than "because we can". Upon reflection, the following motivations seem useful to state.

The motivations in this section are in no particular order, and this reflects the reality that not all implementers and deployers would agree on "the most important motivations".

### "Always-On" Transport-level Authentication and Encryption {#alwas-on}

Although application-level mechanisms to encrypt RTP and RTCP payloads have existed since the introduction of Secure Real-time Transport Protocol (SRTP) {{?RFC3711}}, encryption of RTP and RTCP header fields and contributing sources has only been defined recently (in Cryptex {{?RFC9335}}, and both SRTP and Cryptex are optional capabilities for RTP.

This is in sharp contrast to "always-on" transport-level encryption in the QUIC protocol, using Transport Layer Security (TLS 1.3) as described in {{?RFC9001}}. QUIC implementations always authenticate the entirety of each packet, and encrypt as much of each packet as is practical, even switching from "long headers", which expose more QUIC header fields needed to establish a connection, to "short headers", which only expose the absolute minimum QUIC header fields needed to identify the connection to the receiver, so that the QUIC payload is presented to the right QUIC application {{?RFC8999}}.

### "Always-On" Internet-Safe Congestion Avoidance

When RTP is carried directly over UDP, as is commonly done, the underlying UDP protocol provides no transport services beyond path multiplexing using UDP ports. All congestion avoidance behavior is up to the RTP application itself, and if anything goes wrong with the application resulting in an RTP sender failing to recognize that it is contributing to path congestion, the "worst case" response is to invoke RTP "circuit breaker" procedures {{?RFC8083}}, resulting in "ceasing transmission", as described in {{Section 4.5 of ?RFC8083}}. Because RTCP-based circuit breakers only detect long-lived congestion, a response based on these mechanisms will not happen quickly.

In contrast, when RTP is carried over QUIC, QUIC implementations maintain their own estimates of key transport parameters needed to detect and respond to possible congestion, and these are independent of any measurements RTP senders and receivers are maintaining. The result is that even if an RTP sender continues to "send", QUIC congestion avoidance procedures (for example, the procedures defined in {{?RFC9002}}) will cause the RTP packets to be buffered and only placed on the network path as part of a response to detected loss. This happens without any action being requied on the part of RTP senders.

While the effect of QUIC's response to congestion means that some RTP packets will arrive at the receiver later than a user of the RTP flow might prefer, it is still preferable to "ceasing transmission" completely until the RTP sender has a reason to believe that restarting the flow will not result in congestion.

Moreover, when a single QUIC connection is used to multiplex both RTP-RTCP and non-RTP packets as described in {{single-path}}, the QUIC connection will still be Internet-safe, with no coordination required.

### Rate Adaptation Based on QUIC Feedback {#ra-quic-feedback}

RTP makes use of a large number of RTP-specific feedback mechanisms because when RTP is carried directly over UDP, there is no other way to receive feedback. Some of these mechanisms are specific to the type of media RTP is sending, but others can be mapped from underlying QUIC implementations that are using this feedback to perform rate adaptation for any QUIC connection, regardless of the application reflected in the QUIC STREAM {{?RFC9000}} and DATAGRAM {{?RFC9221}} frames. This is described in (much) more detail in {{congestion-control}} on rate adaptation, and in {{rtcp-mapping}} on replacing RTCP and RTP header extensions with QUIC feedback.

One word of caution is in order - RTP implementations may rely on at least some minimal periodic RTCP feedback, in order to determine that an RTP flow is still active, and is not causing sustained congestion (as described in {{?RFC8083}}, but since this "periodicity" is measured in seconds, the impact of this "duplicate" feedback on path bandwidth utilization is likely close to zero.

### Path MTU Discovery and RTP Media Coalescence {#mtu-coal}

The minimum Path MTU supported by conformant QUIC implementations is 1200 bytes {{?RFC9000}}, and in addition, QUIC implementations allow senders to use either DPLPMTUD ({{?RFC8899}}) or PMTUD ({{?RFC1191}}, {{?RFC8201}}) to determine the actual MTU size that the receiver and path between sender and receiver support, which can be even larger.

This is especially useful in certain conferencing topologies, where otherwise senders have no choice but to use the lowest path MTU for all conference participants, but even in point-to-point RTP sessions, this also allows senders to piggyback audio media in the same UDP packet as video media, for example, and also allows QUIC receivers to piggyback QUIC ACK frames on any QUIC frames being transmitted in the other direction.

### Multiplexing RTP, RTCP, and Non-RTP Flows on a Single QUIC Connection {#single-path}

In order to conserve ports, especially at NATs and Firewalls, this specification defines a flow identifier, so that multiple RTP flows, RTCP flows, and non-RTP flows can be distinguished even if they are carried on the same QUIC connection. This is described in more detail in {{multiplexing}}.

### Exploiting Multiple Connections {#multiple-paths}

Although there is much interest in multiplexing flows on a single QUIC connection as described in {{single-path}}, QUIC also provides the capability of establishing and validating multiple paths for a single QUIC connection {{?RFC9000}}. Once multiple paths have been validated, a sender can migrate from one path to another with no additional signaling, allowing an endpoint to move from one endpoint address to another without interruption, as long as only a single path is in active use at any point in time.

Connection migration may be desireable for a number of reasons, but to give one example, this allows a sender to distinguish between more costly cellular paths and less costly WiFi paths, with no action required from the application.

### Exploiting New QUIC Capabilities {#new-quic}

In addition to connection migration as described in {{multiple-paths}}, the capability of validating multiple paths for simultaneous active use is under active development in the IETF {{?I-D.draft-ietf-quic-multipath}}. We don't discuss Multipath QUIC further in this document, because the specification hasn't been approved yet, but it's one example of ways that RTP, a mature protocol, can exploit new transport capabilities as they become available.

## What's in Scope for this Specification {#in-scope}

This document defines a mapping for RTP and RTCP over QUIC, called RoQ, and describes ways to reduce the
amount of RTCP traffic by leveraging state information readily available within
a QUIC endpoint. This document also describes different options for implementing
congestion control and rate adaptation for RoQ.

This specification focuses on providing a secure encapsulation of RTP packets
for transmission over QUIC. The expected usage is wherever RTP is used to carry
media packets, allowing QUIC in place of other transport protocols such as TCP,
UDP, SCTP, DTLS, etc. That is, we expect RoQ to be used in contexts in
which a signaling protocol is used to announce or negotiate a media
encapsulation and the associated transport parameters (such as IP address, port
number). RoQ is not intended as a stand-alone media transport,
although QUIC transport parameters could be statically configured.

The above implies that RoQ is targeted at peer-to-peer operation; but
it may also be used in client-server-style settings, e.g., when talking to a
conference server as described in RFC 7667 ({{!RFC7667}}), or, if RoQ
is used to replace RTSP ({{?RFC7826}}), to a media server.

Moreover, this document describes how a QUIC implementation and its API can be
extended to improve efficiency of the RoQ protocol operation.

RoQ does not impact the usage of RTP Audio Video Profiles (AVP)
({{!RFC3551}}), or any RTP-based mechanisms, even though it may render some of
them unnecessary, e.g., Secure Real-Time Transport Prococol (SRTP)
({{?RFC3711}}) might not be needed, because end-to-end security is already
provided by QUIC, and double encryption by QUIC and by SRTP might have more
costs than benefits.  Nor does RoQ limit the use of RTCP-based
mechanisms, even though some information or functions obtained by using RTCP
mechanisms may also be available from the underlying QUIC implementation by
other means.

Between two (or more) endpoints, RoQ supports multiplexing multiple
RTP-based media streams within a single QUIC connection and thus using a single
(destination IP address, destination port number, source IP address, source port
number, protocol) 5-tuple..  We note that multiple independent QUIC connections
may be established in parallel using the same destination IP address,
destination port number, source IP address, source port number, protocol)
5-tuple., e.g. to carry different media channels. These connections would be
logically independent of one another.

## What's Out of Scope for this Specification {#out-of-scope}

This document does not attempt to enhance QUIC for real-time media or define a
replacement for, or evolution of, RTP. Work to map other media transport
protocols to QUIC is under way elsewhere in the IETF.

RoQ is designed for use with point-to-point connections, because QUIC
itself is not defined for multicast operation. The scope of this document is
limited to unicast RTP/RTCP, even though nothing would or should prevent its use
in multicast setups once QUIC supports multicast.

RoQ does not define congestion control and rate adaptation algorithms
for use with media. However, {{congestion-control}} discusses options for how
congestion control and rate adaptation could be performed at the QUIC and/or at
the RTP layer, and how information available at the QUIC layer could be exposed
via an API for the benefit of RTP layer implementation.

> **Editor's note:** Need to check whether {{congestion-control}} will also
> describe the QUIC interface that's being exposed, or if that ends up somewhere
> else in the document.

RoQ does not define prioritization mechanisms when handling different
media as those would be dependent on the media themselves and their
relationships. Prioritization is left to the application using RoQ.

This document does not cover signaling for session setup. SDP for RoQ
is defined in separate documents such as
{{?I-D.draft-dawkins-avtcore-sdp-rtp-quic}}, and can be carried in any signaling
protocol that can carry SDP, including the Session Initiation Protocol (SIP)
({{?RFC3261}}), Real-Time Protocols for Browser-Based Applications (RTCWeb)
({{?RFC8825}}), or WebRTC-HTTP Ingestion Protocol (WHIP)
({{?I-D.draft-ietf-wish-whip}}).

# Terminology and Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

> **Editor's note:** the list of terms below will almost certainly grow in size as the specification matures.

The following terms are used:

Congestion Control:
: A mechanism to limit the aggregate amount of data that has been sent over a
path to a receiver, but has not been acknowledged by the receiver. This prevents
a sender from overwhelming the capacity of a path between a sender and a
receiver, causing some outstanding data to be discarded before the receiver can
receive the data and acknowledge it to the sender.

Datagram:
: Datagrams exist in UDP as well as in QUIC's unreliable datagram extension. If not explicitly noted
differently, the term datagram in this document refers to a QUIC Datagram as defined in
{{!RFC9221}}.

Delay-based or Low-latency congestion control algorithm:
: A congestion control algorithm that aims at keeping queues, and thus the latency, at intermediary network elements as short as possible. Delay-based congestion control algorithms use, for example, an increasing one-way delay as a signal of congestion.

Endpoint:
: A QUIC server or client that participates in an RoQ session.

Frame:
: A QUIC frame as defined in {{!RFC9000}}.

Loss-based congestion control algorithm:
: A congestion control algorithm that uses packet loss, or an Explicit Congestion Notification (ECN) that is interpreted as loss (as in {{?RFC3168}}), as a signal for congestion. Loss-based congestion control algorithms allow senders to send data on a path until packets are dropped by intermediary network elements, which the algorithm treats as a signal of congestion.

Media Encoder:
: An entity that is used by an application to produce a stream of encoded media, which can be
packetized in RTP packets to be transmitted over QUIC.

QUIC congestion controller:
: A software component of an application's QUIC implementation that implements a congestion control algorithm.

Rate Adaptation:
: A congestion control mechanism that helps a sender determine and adjust its sending rate, in order
to maximize the amount of information that is sent to a receiver, without
causing queues to build beyond a reasonable amount, causing "buffer bloat" and
"jitter". Rate adapation is one way to accomplish congestion control for
real-time media, especially when a sender has multiple media streams to the
receiver, because the sum of all sending rates for media streams must not be
high enough to cause congestion on the path these media streams share between
sender and receiver.

Receiver:
: An endpoint that receives media in RTP packets and may send or receive RTCP packets.

RTP congestion controller:
: A software component of an application's RTP implementation that implements a congestion control algorithm.

Sender:
: An endpoint that sends media in RTP packets and may send or receive RTCP packets.

Packet diagrams in this document use the format defined in {{Section 1.3 of RFC9000}} to
illustrate the order and size of fields.

# Protocol Overview

This document introduces a mapping of the Real-time Transport Protocol (RTP) to
the QUIC transport protocol. RoQ allows the use of QUIC streams and
QUIC datagrams to transport real-time data, and thus, the QUIC
implementation MUST support QUIC's datagram extension, if RTP packets
should be sent over QUIC datagrams. Since datagram frames cannot be fragmented,
the QUIC implementation MUST also provide a way to query the maximum datagram
size so that an application can create RTP packets that always fit into a QUIC
datagram frame.

{{!RFC3550}} specifies that RTP sessions need to be transmitted on different
transport addresses to allow multiplexing between them. RoQ uses a
different approach to leverage the advantages of QUIC connections without
managing a separate QUIC connection per RTP session. QUIC does not provide
demultiplexing between different flows on datagrams but suggests that an
application implement a demultiplexing mechanism if required. An example of such
a mechanism are flow identifiers prepended to each datagram frame as described
in {{Section 2.1 of ?I-D.draft-ietf-masque-h3-datagram}}. RoQ uses a
flow identifier to replace the network address and port number to multiplex many
RTP sessions over the same QUIC connection.

A rate adaptation algorithm can be plugged in to adapt the media bitrate to the
available bandwidth. This document does not mandate any specific rate adaptation
algorithm, because the desired response to congestion can be application and codec-specific. For example, adjusting quantization in response to congestion may work well in many cases, but if what's being shared is video that includes text, maintaining readability is important.

As of this writing, the IETF has produced two Experimental-track rate adaptation specifications, Network-Assisted Dynamic Adaptation (NADA)
{{!RFC8698}} and Self-Clocked Rate Adaptation for Multimedia (SCReAM)
{{!RFC8298}}. These rate adaptation algorithms require some feedback about
the network's performance to calculate target bitrates. Traditionally this
feedback is generated at the receiver and sent back to the sender via RTCP.

Since QUIC also collects some metrics about the network's performance, these
metrics can be used to generate the required feedback at the sender-side and
provide it to the rate adaptation algorithm to avoid the additional overhead of the
RTCP stream. This is discussed in more detail in {{rtcp-mapping}}.

## Supported RTP Topologies {#topologies}

RoQ only supports some of the RTP topologies described in
{{?RFC7667}}. Most notably, due to QUIC {{!RFC9000}} being a purely IP unicast
protocol at the time of writing, RoQ cannot be used as a transport
protocol for any of the paths that rely on IP multicast in several multicast
topologies (e.g., *Topo-ASM*, *Topo-SSM*, *Topo-SSM-RAMS*).

Some "multicast topologies" can include IP unicast paths (e.g., *Topo-SSM*,
*Topo-SSM-RAMS*). In these cases, the unicast paths can use RoQ.

RTP supports different types of translators and mixers. Whenever a middlebox
such as a translator or a mixer needs to access the content of RTP/RTCP-packets,
the QUIC connection has to be terminated at that middlebox.

RoQ streams (see {{quic-streams}}) can support much larger RTP
packet sizes than other transport protocols such as UDP can, which can lead to
problems with transport translators which translate from RoQ to RTP
over a different transport protocol. A similar problem can occur if a translator
needs to translate from RTP over UDP to RoQ datagrams, where the MTU
of a QUIC datagram may be smaller than the MTU of a UDP datagram. In both cases,
the translator may need to rewrite the RTP packets to fit into the smaller MTU
of the other protocol. Such a translator may need codec-specific knowledge to
packetize the payload of the incoming RTP packets in smaller RTP packets.

Additional details are provided in the following table.

| RFC 7667 Section | Shortcut Name | RTP over QUIC? | Comments |
| --------         | --------          | --------                   | -------- |
| [3.1](https://datatracker.ietf.org/doc/html/rfc7667#section-3.1) | Topo-Point-to-Point | yes |  |
| [3.2.1.1](https://datatracker.ietf.org/doc/html/rfc7667#section-3.2.1.1) | Topo-PtP-Relay | yes | Note-NAT |
| [3.2.1.2](https://datatracker.ietf.org/doc/html/rfc7667#section-3.2.1.2) | Topo-Trn-Translator | yes | Note-MTU <br> Note-SEC  |
| [3.2.1.3](https://datatracker.ietf.org/doc/html/rfc7667#section-3.2.1.3) | Topo-Media-Translator | yes | Note-MTU |
| [3.2.2](https://datatracker.ietf.org/doc/html/rfc7667#section-3.2.2) | Topo-Back-To-Back | yes  | Note-SEC <br> Note-MTU <br> Note-MCast|
| [3.3.1](https://datatracker.ietf.org/doc/html/rfc7667#section-3.3.1) | Topo-ASM | no | Note-MCast |
| [3.3.2](https://datatracker.ietf.org/doc/html/rfc7667#section-3.3.2) | Topo-SSM | partly | Note-MCast <br> Note-UCast-MCast |
| [3.3.3](https://datatracker.ietf.org/doc/html/rfc7667#section-3.3.3) | Topo-SSM-RAMS | partly | Note-MCast <br> Note-MCast-UCast |
| [3.4](https://datatracker.ietf.org/doc/html/rfc7667#section-3.4) | Topo-Mesh | yes | Note-MCast |
| [3.5.1](https://datatracker.ietf.org/doc/html/rfc7667#section-3.5.1) | Topo-PtM-Trn-Translator | possibly | Note-MCast <br> Note-MTU <br> Note-Topo-PtM-Trn-Translator |
| [3.6](https://datatracker.ietf.org/doc/html/rfc7667#section-3.6) | Topo-Mixer | possibly | Note-MCast <br> Note-Topo-Mixer|
| [3.6.1](https://datatracker.ietf.org/doc/html/rfc7667#section-3.6.1) | Media-Mixing-Mixer | partly | Note-Topo-Mixer |
| [3.6.2](https://datatracker.ietf.org/doc/html/rfc7667#section-3.6.2) | Media-Switching-Mixer | partly | Note-Topo-Mixer |
| [3.7](https://datatracker.ietf.org/doc/html/rfc7667#section-3.7) | Selective Forwarding Middlebox | yes | Note-MCast <br> Note-Topo-Mixer |
| [3.8](https://datatracker.ietf.org/doc/html/rfc7667#section-3.8) | Topo-Video-switch-MCU | yes  |  Note-MTU <br> Note-MCast <br> Note-Topo-Mixer |
| [3.9](https://datatracker.ietf.org/doc/html/rfc7667#section-3.9) | Topo-RTCP-terminating-MCU | yes | Note-MTU <br> Note-MCast <br> Note-Topo-Mixer |
| [3.10](https://datatracker.ietf.org/doc/html/rfc7667#section-3.10) | Topo-Split-Terminal | yes | Note-MCast|
| [3.11](https://datatracker.ietf.org/doc/html/rfc7667#section-3.11) | Topo-Asymmetric | Possibly | Note-Warn, <br> Note-MCast, <br> Note-MTU |

Note-NAT:
: QUIC {{!RFC9000}} doesn't support NAT traversal.

Note-MTU:
: Supported, but may require MTU adaptation.

Note-Sec:
: Note that RoQ provides mandatory security, and other RTP transports
do not. {{sec-considerations}} describes strategies to prevent the inadvertent
disclosure of RTP sessions to unintended third parties.

Note-MCast:
: QUIC {{!RFC9000}} cannot be used for multicast paths.

Note-UCast-MCast:
: The topology refers to a *Distribution Source*, which receives and relays RTP
from a number of different media senders via unicast before relaying it to the
receivers via multicast. QUIC can be used between the senders and the
*Distribution Source*.

Note-MCast-UCast:
: The topology refers to a *Burst Source* or *Retransmission Source*, which
retransmits RTP to receivers via unicast. QUIC can be used between the
*Retransmission Source* and the receivers.

Note-Topo-PtM-Trn-Translator:
: Supports unicast paths between RTP sources and translators.

Note-Topo-Mixer:
: Supports unicast paths between RTP senders and mixers.

Note-Warn:
: Quote from {{?RFC7667}}: *This topology is so problematic and it is so easy to
get the RTCP processing wrong, that it is NOT RECOMMENDED to implement this
topology.*

# Connection Establishment and ALPN {#alpn}

QUIC requires the use of ALPN {{!RFC7301}} tokens during connection setup. RoQ
uses "rtp-mux-quic" as ALPN token in the TLS handshake (see also
{{iana-considerations}}.

Note that the use of a given RTP profile is not reflected in the ALPN token even
though it could be considered part of the application usage.  This is simply
because different RTP sessions, which may use different RTP profiles, may be
carried within the same QUIC connection.

> **Editor's note:** "rtp-mux-quic" indicates that RTP and other protocols may
> be multiplexed on the same QUIC connection using a flow identifier as
> described in {{encapsulation}}. Applications are responsible for negotiation
> of protocols in use by appropriate use of a signaling protocol such as SDP.

> **Editor's note:** This implies that applications cannot use RoQ as
> specified in this document over WebTransport.

## Draft version identification

> **RFC Editor's note:** Please remove this section prior to publication of a
> final version of this document.

RoQ uses the token "rtp-mux-quic" to identify itself in ALPN.

Only implementations of the final, published RFC can identify themselves as
"rtp-mux-quic". Until such an RFC exists, implementations MUST NOT identify
themselves using this string.

Implementations of draft versions of the protocol MUST add the string "-" and
the corresponding draft number to the identifier.  For example,
draft-ietf-avtcore-rtp-over-quic-04 is identified using the string
"rtp-mux-quic-04".

Non-compatible experiments that are based on these draft versions MUST append
the string "-" and an experiment name to the identifier.

# Encapsulation {#encapsulation}

This section describes the encapsulation of RTP/RTCP packets in QUIC.

QUIC supports two transport methods: streams {{!RFC9000}} and
datagrams {{!RFC9221}}. This document specifies mappings of RTP to
both of the transport modes. Senders MAY combine both modes by sending some
RTP/RTCP packets over the same or different QUIC streams and others in QUIC
datagrams.

{{multiplexing}} introduces a multiplexing mechanism that supports multiplexing
RTP, RTCP, and, with some constraints, other non-RTP protocols. {{quic-streams}}
and {{quic-datagrams}} explain the specifics of mapping RTP to QUIC streams and
QUIC datagrams, respectively.

## Multiplexing {#multiplexing}

RoQ uses flow identifiers to multiplex different RTP, RTCP, and
non-RTP data streams on a single QUIC connection. A flow identifier is a QUIC
variable-length integer as described in {{Section 16 of !RFC9000}}. Each flow
identifier is associated with a stream of RTP packets, RTCP packets, or a data
stream of a non-RTP protocol.

In a QUIC connection using the ALPN token defined in {{alpn}}, every QUIC
datagram and every QUIC stream MUST start with a flow identifier. A peer MUST
NOT send any data in a datagram or stream that is not associated with the flow
identifier which started the datagram or stream.

RTP and RTCP packets of different RTP sessions MUST use distinct flow
identifiers. If peers wish to send multiple types of media in a single RTP
session, they MAY do so by following {{?RFC8860}}.

A single RTP session MAY be associated with one or two flow identifiers. Thus,
it is possible to send RTP and RTCP packets belonging to the same session using
different flow identifiers. RTP and RTCP packets of a single RTP session MAY use
the same flow identifier (following the procedures defined in {{?RFC5761}}, or
they MAY use different flow identifiers.

The association between flow identifiers and data streams MUST be negotiated
using appropriate signaling. Applications MAY send data using flow identifiers
not associated with any RTP or RTCP stream. If a receiver cannot associate a
flow identifier with any RTP/RTCP or non-RTP flow, it MAY drop the data flow. If
the data flow was sent on a QUIC stream, the receiver SHOULD send a
STOP\_SENDING frame with the application error code set to
ROQ\_UNKNOWN\_FLOW\_ID.

There are different use cases for sharing the same QUIC connection between RTP
and non-RTP data streams. Peers might use the same connection to exchange
signaling messages or exchange data while sending and receiving media streams.
The semantics of non-RTP datagrams or streams are not in the scope of this
document. Peers MAY use any protocol on top of the encapsulation described in
this document.

Flow identifiers introduce some overhead in addition to the header overhead of
RTP/RTCP and QUIC. QUIC variable-length integers require between one and eight
bytes depending on the number expressed. Thus, it is advisable to use low
numbers first and only use higher ones if necessary due to an increased number
of flows.

## QUIC Streams {#quic-streams}

To send RTP/RTCP packets over QUIC streams, a sender MUST open a
new unidirectional QUIC stream. Streams are unidirectional because there is no
synchronous relationship between sent and received RTP/RTCP packets. A peer that
receives a bidirectional stream with a flow identifier that is associated with
an RTP or RTCP stream, SHOULD stop reading from the stream and send a
STOP\_SENDING frame with the application protocol error code set to
ROQ\_STREAM\_CREATION\_ERROR.

A RoQ sender MAY open new QUIC streams for different packets using the same flow
identifier, for example, to avoid head-of-line blocking.

A receiver MUST be prepared to receive RTP packets on any number of QUIC streams
(subject to its limit on parallel open streams) and SHOULD not make assumptions
which RTP sequence numbers are carried in which streams.

Note: A sender may or may not decide to discontinue using a lower stream number
after starting packet transmission on a higher stream number.

### Stream Encapsulation

{{fig-stream-payload}} shows the encapsulation format for RoQ Streams.

~~~
Payload {
  Flow Identifier (i),
  RTP/RTCP Payload(..) ...,
}
~~~
{: #fig-stream-payload title="RoQ Streams Payload Format"}

Flow Identifier:

: Flow identifier to demultiplex different data flows on the same QUIC
connection.

RTP/RTCP Payload:

: Contains the RTP/RTCP payload; see {{fig-rtp-stream-payload}}

The payload in a QUIC stream starts with the flow identifier followed by one or
more RTP/RTCP payloads. All RTP/RTCP payloads sent on a stream MUST belong to
the RTP session with the same flow identifier.

Each payload begins with a length field indicating the length of the RTP/RTCP
packet, followed by the packet itself, see {{fig-rtp-stream-payload}}.

~~~
RTP/RTCP Payload {
  Length(i),
  RTP/RTCP Packet(..),
}
~~~
{: #fig-rtp-stream-payload title="RTP/RTCP payload for QUIC streams"}

Length:

: A QUIC variable length integer (see {{Section 16 of !RFC9000}}) describing the
length of the following RTP/RTCP packets in bytes.

RTP/RTCP Packet:

: The RTP/RTCP packet to transmit.

### Media Frame Cancellation

QUIC uses RESET\_STREAM and STOP\_SENDING frames to terminate the sending part
of a stream and to request termination of an incoming stream by the sending
peer respectively.

A RoQ sender MAY use RESET\_STREAM if it knows that a packet, which was not yet
successfully and completely transmitted, is no longer needed.

A RoQ receiver that is no longer interested in reading a certain partition of
the media stream MAY signal this to the sending peer using a STOP\_SENDING
frame.

In both cases, the error code of the RESET\_STREAM frame or the STOP\_SENDING
frame MUST be set to ROQ\_FRAME\_CANCELLED.

When a RoQ sender receives a STOP\_SENDING frame for the last open stream
available to send RTP/RTCP-data, the RoQ sender MUST open one or more new QUIC
streams before sending new media frames. Any media frame that has already been
sent on the QUIC stream that received the STOP\_SENDING frame, MUST NOT be sent
again on the new QUIC stream(s).

Note that an RTP receiver cannot request a reset of only a particular media
frame because the sending QUIC implementation might already have sent data for
the following frame on the same stream. In that case, STOP\_SENDING and the
following RESET\_STREAM would also drop the following media frame and thus lead
to unintentionally skipping one or more frames.

> **Editor's note:** A receiver cannot cancel a certain frame but still receive
> retransmissions for a frame the was following on the same stream using
> STOP\_SENDING, because STOP\_SENDING does not include an offset which would
> allow signaling where retransmissions should continue.

> **Editor's note:** Instead of using RESET\_STREAM and STOP\_SENDING frames,
> RoQ senders and receivers might benefit from negotiating the use of the QUIC
> extensions defined in {{?I-D.draft-ietf-quic-reliable-stream-reset-01}} and
> {{?I-D.draft-thomson-quic-enough-00}}. These extensions provide two new frame
> types, the CLOSE\_STREAM and the ENOUGH frame, equivalent to RESET\_STREAM and
> STOP\_SENDING, respectively, but with an additional offset. The offset
> indicates the point to which data will be reliably retransmitted, while
> everything following might be dropped. Using CLOSE\_STREAM and ENOUGH instead
> of RESET\_STREAM and STOP\_SENDING could prevent accidentally stopping
> retransmissions for preceding frames.

A translator that translates between two endpoints, both connected via QUIC,
MUST forward RESET\_STREAM frames received from one end to the other unless it
forwards the RTP packets on QUIC datagrams.

Large RTP packets sent on a stream will be fragmented into smaller QUIC frames.
The QUIC frames are transmitted reliably and in order such that a receiving
application can read a complete RTP packet from the stream as long as the stream
is not closed with a RESET\_STREAM frame. No retransmission has to be
implemented by the application since QUIC frames lost in transit are
retransmitted by QUIC.

### Flow control and MAX\_STREAMS

Opening new streams for new packets MAY implicitly limit the number of packets
concurrently in transit because the QUIC receiver provides an upper bound of
parallel streams, which it can update using QUIC MAX\_STREAMS frames. The number
of packets that have to be transmitted concurrently depends on several factors,
such as the number of RTP streams within a QUIC connection, the bitrate of the
media streams, and the maximum acceptable transmission delay of a given packet.
Receivers are responsible for providing senders enough credit to open new
streams for new packets anytime. As an example, consider a conference scenario
with 20 participants. Each participant receives audio and video streams of every
other participant from a central server. If the sender opens a new QUIC stream
for every frame at 30 frames per second video and 50 frames per second audio, it
will open 1520 new QUIC streams per second. A receiver must provide at least
that many credits for opening new unidirectional streams to the server every
second. In addition, the receiver should also consider the requirements of
protocols into account that are multiplexed with RTP, including RTCP and data
streams. These considerations may also be relevant when implementing signaling
since it may be necessary to inform the receiver about how fast and how much
stream credits it will have to provide to the media-sending peer.

## QUIC Datagrams {#quic-datagrams}

Senders can also transmit RTP packets in QUIC datagrams. QUIC datagrams are an
extension to QUIC described in {{!RFC9221}}. QUIC datagrams can only be used if
the use of the extension was successfully negotiated during the QUIC handshake.
If the use of an extension was signaled using a signaling protocol, but the
extension was not negotiated during the QUIC handshake, a peer MAY close the
connection with the ROQ\_SIGNALING\_ERROR error code.

QUIC datagrams preserve frame boundaries. Thus, a single RTP packet can be
mapped to a single QUIC datagram without additional framing. Senders SHOULD
consider the header overhead associated with QUIC datagrams and ensure that the
RTP/RTCP packets, including their payloads, flow identifier, QUIC, and IP
headers, will fit into path MTU.

{{fig-dgram-payload}} shows the encapsulation format for RoQ
Datagrams.

~~~
Payload {
  Flow Identifier (i),
  RTP/RTCP Packet (..),
}
~~~
{: #fig-dgram-payload title="RoQ Datagram Payload Format"}

Flow Identifier:

: Flow identifier to demultiplex different data flows on the same QUIC
connection.

RTP/RTCP Packet:

: The RTP/RTCP packet to transmit.

RoQ senders need to be aware that QUIC uses the concept of QUIC frames.
Different kinds of QUIC frames are used for different application and control
data types. A single QUIC packet can contain more than one QUIC frame,
including, for example, QUIC stream or datagram frames carrying application data
and acknowledgement frames carrying QUIC acknowledgements, as long as the
overall size fits into the MTU. One implication is that the number of packets a
QUIC stack transmits depends on whether it can fit acknowledgement and datagram
frames in the same QUIC packet. Suppose the application creates many datagram
frames that fill up the QUIC packet. In that case, the QUIC stack might have to
create additional packets for acknowledgement- (and possibly other control-)
frames. The additional overhead could, in some cases, be reduced if the
application creates smaller RTP packets, such that the resulting datagram frame
can fit into a QUIC packet that can also carry acknowledgement frames.

Since QUIC datagrams are not retransmitted on loss (see also
{{transport-layer-feedback}} for loss signaling), if an application wishes to
retransmit lost RTP packets, the retransmission has to be implemented by the
application. RTP retransmissions can be done in the same RTP session or a
separate RTP session {{!RFC4588}} and the flow identifier MUST be set to the
flow identifier of the RTP session in which the retransmission happens.

# Connection Shutdown

Either peers MAY close the connection for variety of reasons. If one of the
peers wants to close the RoQ connection, the peer can use a QUIC
CONNECTION\_CLOSE frame with one of the error codes defined in
{{error-handling}}.

# Congestion Control and Rate Adaptation {#congestion-control}

Like any other application on the internet, RoQ applications need a mechanism to
perform congestion control to avoid overloading the network. While any generic
congestion controller can protect the network, this document takes advantage of
the opportunity to use rate adaptation mechanisms that are designed to provide
superior user experiences for real-time media applications.

A wide variety of rate adaptation algorithms for real-time media have been
developed (for example, "Google Congestion Controller"
{{?I-D.draft-ietf-rmcat-gcc}}). The IETF has defined two algorithms in two
Experimental RFCs (e.g. SCReAM {{?RFC8298}} and NADA {{?RFC8698}}). These rate
adaptation algorithms for RTP are specifically tailored for real-time
transmissions at low latencies, but this section would apply to any rate
adaptation algorithm that meets the requirements described in "Congestion
Control Requirements for Interactive Real-Time Media" {{!RFC8836}}.

This document defines two architectures for congestion control and bandwidth
estimation for RoQ, depending on whether most rate adaptation is performed
within a QUIC implementation at the transport layer, as described in
{{cc-quic-layer}}, or within an RTP application layer, as described in
{{cc-application-layer}}, but this document does not mandate any specific
congestion control or rate adaptation algorithm for either QUIC or RTP.

This document also gives guidance about avoiding problems with "nested"
congestion controllers, in {{nested-CC}}.

This document also discusses congestion control
implications of using shared or multiple separate QUIC connections to send and
receive multiple independent data streams, in {{shared-connections}}.

It is assumed that the congestion controller in use provides a pacing mechanism
to determine when a packet can be sent to avoid bursts. The currently proposed
congestion control algorithms for real-time communications (e.g. SCReAM and
NADA) provide such pacing mechanisms. The use of congestion controllers which
don't provide a pacing mechanism is out of scope of this document.

## Congestion Control at the Transport Layer {#cc-quic-layer}

QUIC is a congestion controlled transport protocol. Senders are required to
employ some form of congestion control. The default congestion control specified
for QUIC in {{!RFC9002}} is similar to TCP NewReno {{?RFC6582}}, but senders are
free to choose any congestion control algorithm as long as they follow the
guidelines specified in {{Section 3 of ?RFC8085}}, and QUIC implementors make
use of this freedom.

If a QUIC implementation is to perform rate adaptation in a way that
accommodates real-time media, one way for the implementation to recognize that
it is carrying real-time media is to be explicitly told that this is the case.
This document defines a new "TLS Application-Layer Protocol Negotiation (ALPN)
Protocol ID", as described in {{alpn}}, that a QUIC implementation can use as a
signal to choose a real-time media-centric rate controller, but this is not
required for ROQ deployments.

If congestion control is to be applied at the transport layer, it is RECOMMENDED
that the QUIC Implementation uses a congestion controller that keeps queueing
delays short to keep the transmission latency for RTP and RTCP packets as low as
possible, such as the IETF-defined SCReAM {{?RFC8298}} and NADA {{?RFC8698}}
algorithms.

Many low latency congestion control algorithms depend on detailed arrival time
feedback to estimate the current one-way delay between sender and receiver. QUIC
does not provide arrival timestamps in its acknowledgments. The QUIC
implementations of the sender and receiver can use an extension to add this
information to QUICs acknowledgment frames, e.g.
{{!I-D.draft-smith-quic-receive-ts}} or {{!I-D.draft-huitema-quic-ts}}.

If congestion control is done by the QUIC implementation, the application needs
a mechanism to query the currently available bandwidth to adapt media codec
configurations. The employed congestion controller of the QUIC connection SHOULD
expose such an API to the application. If a current bandwidth estimate is not
available from the QUIC congestion controller, the sender can either implement
an alternative bandwidth estimation at the application layer as described in
{{cc-application-layer}} or a receiver can feedback the observed bandwidth
through RTCP, e.g., using {{?I-D.draft-alvestrand-rmcat-remb}}.

## Congestion Control at the RTP Application Layer {#cc-application-layer}

RTP itself does not specify a congestion control algorithm, but {{!RFC8888}}
defines an RTCP feedback message intended to enable rate adaptation for
interactive real-time traffic using RTP, and successful rate adaptation will
accomplish congestion control as well.

The rate adaptation algorithms for RTP are specifically tailored for real-time
transmissions at low latencies, as described in {{congestion-control}}. The
available rate adaptation algorithms expose a `target_bitrate` that can be used
to dynamically reconfigure media codecs to produce media at a rate that can be
sent in real-time under the observed network conditions.

If an application cannot access a bandwidth estimation from the QUIC layer, or
the QUIC implementation does not support a delay-based, low-latency congestion
control algorithm, the application can alternatively implement a bandwidth estimation
algorithm at the application layer. Calculating a bandwidth estimation at the
application layer can be done using the same bandwidth estimation algorithms as
described in {{congestion-control}} (NADA, SCReAM). The bandwidth estimation
algorithm typically needs some feedback on the transmission performance. This
feedback can be collected following the guidelines in {{rtcp-mapping}}.

## Resolving Interactions Between QUIC and Application-layer Congestion Control {#nested-CC}

Because QUIC is a congestion-controlled transport, as described in
{{cc-quic-layer}}, and RTP applications can also perform congestion control and
rate adaptation, as described in {{cc-application-layer}}, implementers should
be aware of the possibility that these "nested" congestion control loops, where
both controllers are managing rate adaptation for the same packet stream
independently, may deliver problematic performance. Because this document is
describing a specific case (media transport), we can provide some guidance to
avoid the worst possible problems.

- **Application-limited Media Flows** - if an application chooses RTP as its
  transport mechanism, the goal will be maximizing the user experience, not
  maximizing path bandwidth utilization. If the application is, in fact,
  transmitting media that does not saturate path bandwidth, and paces its
  transmission, more heavy-handed congestion control mechanisms (drastic
  reductions in the sending rate when loss is detected, with much slower
  increases when losses are no longer detected) should rarely come into play. If
  the application chooses ROQ as its transport, sends enough media to saturate
  the path bandwidth, and does not adapt its own sending rate, drastic measures
  will be required in order to avoid sustained or oscillating congestion along
  the path.

- **Awareness of Bufferbloat** - modern general-purpose congestion controllers
  do not adjust  their sending rates based only on packet loss. For example, BBR
  ("Bottleneck Bandwidth and Round-trip propagation time")
  {{?I-D.cardwell-iccrg-bbr-congestion-control}} describes its strategy for
  adapting sending rates in this way:

> (BBR) "uses recent measurements of a transport connection's delivery rate,
> round-trip time, and packet loss rate to build an explicit model of the
> network path. BBR then uses this model to control both how fast it sends data
> and the maximum volume of data it allows in flight in the network at any
> time."

## Sharing QUIC connections {#shared-connections}

Two endpoints may establish channels in order to exchange more than one type of
data simultaneously. The channels can be intended to carry real-time RTP data or
other non-real-time data. This can be realized in different ways.

- One straightforward solution is to establish multiple QUIC connections, one
  for each channel, whether the channel is used for real-time media or
  non-real-time data. This is a straightforward solution, but has the
  disadvantage that transport ports are more quickly exhausted and these are
  limited by the 16-bit UDP source and destination port number sizes
  {{!RFC768}}.
- Alternatively, all real-time channels are mapped to one QUIC connection, while
  a separate QUIC connection is created for the non-real-time channels.

In both cases, the congestion controllers can be chosen to match the demands of
the respective channels and the different QUIC connections will compete for the
same resources in the network. No local prioritization of data across the
different (types of) channels would be necessary.

Although it is possible to multiplex (all or a subset of) real-time and
non-real-time channels onto a single, shared QUIC connection, which can be done
by using the flow identifier described in {{multiplexing}}, the underlying QUIC
implementation will likely use the same congestion controller for all channels
in the shared QUIC connection. For this reason, applications multiplexing
multiple streams in one connection SHOULD implement some form of stream
prioritization or bandwidth allocation.

# Replacing RTCP and RTP Header Extensions with QUIC Feedback {#rtcp-mapping}

Because RTP has so often used UDP as its underlying transport protocol, and
receiving little or no feedback from UTP, RTP implementations rely on feedback from
the RTP Control Protocol (RTCP) so that RTP senders and receivers can exchange
control information to monitor connection statistics and to identify and
synchronize streams.

Because QUIC provides its own transport-level feedback, at least some RTP
transport level feedback can be replaced with current QUIC feedback
{{!rfc9000}}. In adition, RTP-level feedback that is not available in QUIC by
default can be replaced with generally useful QUIC extensions. Examples of these
extentions include:

* Arrival timestamps, which are not part of QUIC's default acknowledgment
  frames, but can be added using {{!I-D.draft-smith-quic-receive-ts}} or
  {{!I-D.draft-huitema-quic-ts}}, or
* Adjusting the frequency of QUIC acknowledgments, using
  {{!I-D.draft-ietf-quic-ack-frequency}}.

When statistics contained in RTCP packets are also available from QUIC, or can
be derived from statistics available from QUIC, it is desireable to provide
these statistics at only one protocol layer. This avoids consumption of
bandwidth to deliver duplicated control information. Because QUIC relies on
certain frames being sent, it is not possible to supress QUIC signaling in favor
of RTCP signaling, so if bandwidth is to be conserved, this must be accomplished
by surpressing RTCP signaling in favor of QUIC signalling.

This document specifies a baseline for replacing some of the RTCP packet types
by mapping the contents to QUIC connection statistics. Future documents can
extend this mapping for other RTCP format types, and can make use of new QUIC
extensions that become available over time.

Most statements about "QUIC" in {{rtcp-mapping}} are applicable to both RTP
encapsulated in QUIC streams and RTP encapsulated in QUIC datagrams. The
differences are described in {{roc-d}} and {{roc-s}}.

> **Editor's Note:** Additional discussion of bandwidth minimization could go in
> this section, or in an earlier proposed section on motivations for defining
> and deploying RoQ.

While RoQ places no restrictions on applications sending RTCP, this
document assumes that the reason an implementor chooses to support RoQ
is to obtain benefits beyond what's available when RTP uses UDP as its
underlying transport layer. It is RECOMMENDED to expose relevant information
from the QUIC layer to the application instead of exchanging additional RTCP
packets, where applicable.

{{transport-layer-feedback}} and {{al-repair}} discuss what information can be
exposed from the QUIC connection layer to reduce the RTCP overhead.

The list of RTCP packets in this section is not exhaustive and similar
considerations SHOULD be taken into account before exchanging any other type of
RTCP control packets using RoQ.

A more complete analysis of RTCP Control Packet Types (in {{control-packets}}),
Generic RTP Feedback (RTPFB) (in {{generic-feedback}}), Payload-specific RTP
Feedback (PSFB) (in {{payload-specific-feedback}}), Extended Reports (in
{{extended-reports}}), and RTP Header Extensions (in {{rtp-header-extensions}}),
including the information that cannot be mapped from QUIC.

## RoQ Datagrams {#roc-d}

QUIC Datagrams are ack-eliciting packets, which means, that an acknowledgment is
triggered when a datagram frame is received. Thus, a sender can assume that an
RTP packet arrived at the receiver or was lost in transit, using the QUIC
acknowledgments of QUIC Datagram frames. In the following, an RTP packet is
regarded as acknowledged, when the QUIC Datagram frame that carried the RTP
packet, was acknowledged.

## RoQ Streams {#roc-s}

For RTP packets that are sent over QUIC streams, an
RTP packet can be considered acknowledged, when all frames which carried
fragments of the RTP packet were acknowledged.

When QUIC Streams are used, the application should be aware that the direct
mapping proposed below may not reflect the real characteristics of the network.
RTP packet loss can seem lower than actual packet loss due to QUIC's automatic
retransmissions. Similarly, timing information might be incorrect due to
retransmissions.

## Transport Layer Feedback {#transport-layer-feedback}

This section explains how some of the RTCP packet types which are used to signal
reception statistics can be replaced by equivalent statistics that are already
collected by QUIC. The following list explains how this mapping can be achieved
for the individual fields of different RTCP packet types.

### Mapping QUIC Feedback to RTCP Receiver Reports ("RR") {#RR-mappings}

Considerations for mapping QUIC feedback into *Receiver Reports* (`PT=201`,
`Name=RR`, {{!RFC3550}}) are:

  * *Fraction lost*: When RTP packets are carried in QUIC datagrams, the fraction of lost packets can be directly inferred from
    QUIC's acknowledgments. The calculation SHOULD include all packets up to the
    acknowledged RTP packet with the highest RTP sequence number. Later packets
    SHOULD be ignored, since they may still be in flight, unless other QUIC
    packets that were sent after the RTP packet, were already acknowledged.
  * *Cumulative lost*: Similar to the fraction of lost packets, the cumulative
    loss can be inferred from QUIC's acknowledgments including all packets up to
    the latest acknowledged packet.
  * *Highest Sequence Number received*: In RTCP, this field is a 32-bit field
    that contains the highest sequence number a receiver received in an RTP
    packet and the count of sequence number cycles the receiver has observed. A
    sender sends RTP packets in QUIC packets and receives acknowledgments for
    the QUIC packets. By keeping a mapping from a QUIC packet to the RTP packets
    encapsulated in that QUIC packet, the sender can infer the highest sequence
    number and number of cycles seen by the receiver from QUIC acknowledgments.
  * *Interarrival jitter*: If QUIC acknowledgments carry timestamps as described
    in one of the extensions referenced above, senders can infer from QUIC acks
    the interarrival jitter from the arrival timestamps.
  * *Last SR*: Similar to RTP arrival times, the arrival time of RTCP Sender
    Reports can be inferred from QUIC acknowledgments, if they include
    timestamps.
  * *Delay since last SR*: This field is not required when the receiver reports
    are entirely replaced by QUIC feedback.

### Mapping QUIC Feedback to RTCP Negative Acknowledgments* ("NACK") {#NACK-mappings}

Considerations for mapping QUIC feedback into *Negative Acknowledgments*
(`PT=205`, `FMT=1`, `Name=Generic NACK`, {{!RFC4585}}) are:

  * The generic negative acknowledgment packet contains information about
    packets which the receiver considered lost. {{Section 6.2.1. of !RFC4585}}
    recommends to use this feature only, if the underlying protocol cannot
    provide similar feedback. QUIC does not provide negative acknowledgments,
    but can detect lost packets based on the Gap numbers contained in QUIC ACK frames {{Section 6 of !RFC9002}}.

### Mapping QUIC Feedback to RTCP ECN Feedback ("ECN") {#ECN-mappings}

Considerations for mapping QUIC feedback into *ECN Feedback* (`PT=205`, `FMT=8`,
`Name=RTCP-ECN-FB`, {{!RFC6679}}) are:

  * ECN feedback packets report the count of observed ECN-CE marks. {{!RFC6679}}
    defines two RTCP reports, one packet type (with `PT=205` and `FMT=8`) and a
    new report block for the extended reports which are listed below. QUIC
    supports ECN reporting through acknowledgments. If the QUIC connection supports
    ECN, the reporting of ECN counts SHOULD be done using QUIC acknowledgments,
    rather than RTCP ECN feedback reports.

### Mapping QUIC Feedback to RTCP Congestion Control Feedback ("CCFB") {#CCFB-mappings}

Considerations for mapping QUIC feedback into *Congestion Control Feedback*
(`PT=205`, `FMT=11`, `Name=CCFB`, {{!RFC8888}}) are:

  * RTP Congestion Control Feedback contains acknowledgments, arrival timestamps
    and ECN notifications for each received packet. Acknowledgments and ECNs can
    be inferred from QUIC as described above. Arrival timestamps can be added
    through extended acknowledgment frames as described in
    {{!I-D.draft-smith-quic-receive-ts}} or {{!I-D.draft-huitema-quic-ts}}.

### Mapping QUIC Feedback to RTCP Extended Report ("XR") {#XR-mappings}

Considerations for mapping QUIC feedback into  *Extended Reports* (`PT=207`,
`Name=XR`, {{!RFC3611}}) are:

  * Extended Reports offer an extensible framework for a variety of different
    control messages. Some of the standard report blocks which can be
    implemented in extended reports such as loss RLE or ECNs can be implemented
    in QUIC, too.
  * Other report blocks SHOULD be evaluated individually, to determine whether
    if the contained information can be transmitted using QUIC instead.

## Application Layer Repair and other Control Messages {#al-repair}

While {{RR-mappings}} presented some RTCP packets that can be replaced by QUIC
features, QUIC cannot replace all of the defined RTCP packet types. This mostly
affects RTCP packet types which carry control information that is to be
interpreted by the RTP application layer, rather than the underlying transport
protocol itself.

* *Sender Reports* (`PT=200`, `Name=SR`, {{!RFC3550}}) are similar to *Receiver
  Reports*, as described in {{RR-mappings}}. They are sent by media senders and
  additionally contain an NTP and a RTP timestamp and the number of packets and
  octets transmitted by the sender. The timestamps can be used by a receiver to
  synchronize streams. QUIC cannot provide a similar control information, since
  it does not know about RTP timestamps. Nor can a QUIC receiver calculate the
  packet or octet counts, since it does not know about lost datagrams. Thus,
  sender reports are required in RoQ to synchronize streams at the
  receiver. The sender reports SHOULD not contain any receiver report blocks, as
  the information can be inferred from the QUIC transport as explained in
  {{RR-mappings}}.

In addition to carrying transmission statistics, RTCP packets can contain
application layer control information, that cannot directly be mapped to QUIC.
Examples of this information may include:

* *Source Description* (`PT=202`, `Name=SDES`), *Bye* (`PT=203`, `Name=BYE`) and
  *Application* (`PT=204`, `Name=APP`) packet types from {{!RFC3550}}, or
* many of the payload specific feedback messages (`PT=206`) defined in
  {{!RFC4585}}, used to control the codec behavior of the sender.

Since QUIC does not provide any kind of application layer control messaging,
QUIC feedback cannot be mapped into these RTCP packet types. If the RTP
application needs this information, the RTCP packet types are used in the same
way as they would be used over any other transport protocol.

## RTCP Control Packet Types {#control-packets}

Several but not all of these control packets and their attributes can be mapped
from QUIC, as described in {{RR-mappings}}. "Mappable from QUIC" has one of
three values: "yes", "QUIC extension required", and "no".

| Name | Shortcut | PT | Defining Document | Mappable from QUIC | Comments |
| ---- | -------- | -- | ----------------- | ---------------- | -------- |
| SMPTE time-code mapping | SMPTETC | 194 | {{?RFC5484}}  | no | |
| Extended inter-arrival jitter report | IJ | 195 | {{?RFC5450}} | QUIC extension required | *IJ* was introduced to improve jitter reports when RTP packets are not sent at the time indicated by their RTP timestamp.  Jitter can be calculated using QUIC timestamps, because QUIC timestamps are added when the QUIC packet is actually sent. |
| Sender Reports | SR | 200 | {{?RFC3550}}  | partly | - NTP timestamps cannot be replaced by QUIC and are required for synchronization (but see note below)<br>- packet and octet counts cannot be provided by QUIC<br>- see below for *RR*s contained in *SR*s |
| Receiver Reports | RR | 201 | {{?RFC3550}} | possibly | - *Fraction Lost*/*Cumulative Lost*/*Highest Sequence Number received* can directly be inferred from QUIC ACKs<br>- *Interarrival Jitter*/*Last SR* need a QUIC timestamp extension. Using QUIC ts is slightly different because it ignores transmission offsets from RTP timestamps, but that seems like a good thing (see *IJ* above) |
| Source description | SDES | 202 | {{?RFC3550}}  | no | |
| Goodbye | BYE | 203 | {{?RFC3550}}  | possibly | using QUIC CONNECTION_CLOSE frame? |
| Application-defined | APP | 204 | {{?RFC3550}}  | no | |
| Generic RTP Feedback | RTPFB | 205 | {{?RFC4585}} | partly | see table below |
| Payload-specific | PSFB | 205 | {{?RFC4585}}  | | see table below |
| extended report | XR | 207 | {{?RFC3611}} | partly | see table below |
| AVB RTCP packet | AVB | | |
| Receiver Summary Information | RSI | 209 | {{?RFC5760}} | |
| Port Mapping | TOKEN | 210 | {{?RFC6284}}  | no? | |
| IDMS Settings | IDMS | 211 | {{?RFC7272}}  | no | |
| Reporting Group Reporting Sources | RGRS | 212 | {{?RFC8861}} | |
| Splicing Notification Message | SNM | 213 | {{?RFC8286}} | no | |

### Notes

* *SR* NTP timestamps: We cannot send NTP timestamps in the same format the SRs
  use, but couldn't a QUIC timestamp extension provide the same information?

## Generic RTP Feedback (RTPFB) {#generic-feedback}

| Name     | Long Name | Document | Mappable from QUIC  | Comments |
| -------- | --------- | -------- | ---------------- | -------- |
| Generic NACK | Generic negative acknowledgement | {{?RFC4585}}   | possibly | Using QUIC ACKs |
| TMMBR | Temporary Maximum Media Stream Bit Rate Request | {{?RFC5104}}  | no | |
| TMMBN | Temporary Maximum Media Stream Bit Rate Notification | {{?RFC5104}}   | no | |
| RTCP-SR-REQ | RTCP Rapid Resynchronisation Request | {{?RFC6051}}  | no | |
| RAMS | Rapid Acquisition of Multicast Sessions | {{?RFC6285}} | no | |
| TLLEI | Transport-Layer Third-Party Loss Early Indication | {{?RFC6642}}  | no? | no way of telling QUIC peer "don't ask for retransmission", but QUIC would not ask that anyway, only RTCP NACK? |
| RTCP-ECN-FB | RTCP ECN Feedback | {{?RFC6679}} | partly | QUIC does not provide info about duplicates |
| PAUSE-RESUME | Media Pause/Resume | {{?RFC7728}} | no | |
| DBI | Delay Budget Information (DBI) | {{3GPP-TS-26.114}} | |
| CCFB | RTP Congestion Control Feedback | {{?RFC8888}} | possibly | - *ECN*/*ACK* natively in QUIC<br>- timestamps require QUIC timestamp extension |

## Payload-specific RTP Feedback (PSFB) {#payload-specific-feedback}

Because QUIC is a generic transport protocol, QUIC feedback cannot replace the
following Payload-specific RTP Feedback (PSFB) feedback.

| Name     | Long Name | Document |
| -------- | --------- | -------- |
| PLI | Picture Loss Indication | {{?RFC4585}} |
| SLI | Slice Loss Indication | {{?RFC4585}} |
| RPSI | Reference Picture Selection Indication | {{?RFC4585}} |
| FIR | Full Intra Request Command | {{?RFC5104}} |
| TSTR | Temporal-Spatial Trade-off Request | {{?RFC5104}} |
| TSTN | Temporal-Spatial Trade-off Notification | {{?RFC5104}} |
| VBCM | Video Back Channel Message | {{?RFC5104}}
| PSLEI | Payload-Specific Third-Party Loss Early Indication | {{?RFC6642}} |
| ROI | Video region-of-interest (ROI) | {{3GPP-TS-26.114}} |
| LRR | Layer Refresh Request Command | {{?I-D.draft-ietf-avtext-lrr-07}}|
| AFB | Application Layer Feedback | {{?RFC4585}} |
| TSRR | Temporal-Spatial Resolution Request | {{?I-D.draft-ietf-avtcore-rtcp-green-metadata}} |
| TSRN | Temporal-Spatial Resolution Notification | {{?I-D.draft-ietf-avtcore-rtcp-green-metadata}} |

## Extended Reports (XR) {#extended-reports}

| Name | Document | Mappable from QUIC  | Comments |
| ---- | -------- | ---------------- | -------- |
| Loss RLE Report Block | {{?RFC3611}} | yes | QUIC ACKs |
| Duplicate RLE Report Block | {{?RFC3611}}  | no | |
| Packet Receipt Times Report Block | {{?RFC3611}}  | possibly | - Could be replaced by QUIC timestamps.<br>- Would not use RTP timestamps.<br>- Only if QUIC timestamps for **every** packet is included (e.g. *draft-smith-quic-receive-ts* but not *draft-huitema-quic-ts*) |
| Receiver Reference Time Report Block | {{?RFC3611}}  | possibly | QUIC timestamps |
| DLRR Report Block | {{?RFC3611}}  | possibly | QUIC ACKs and QUIC timestamps. In general, however, it seems to be useful only to calculate RTT, which is natively available in QUIC. |
| Statistics Summary Report Block | {{?RFC3611}}  | partly | - loss and jitter as described in other reports above.<br>- *TTL*/*HL*/*Duplicates* not available in QUIC |
| VoIP Metrics Report Block | {{?RFC3611}}  | no | as in other reports above, only loss and RTT available |
| RTCP XR | {{?RFC5093}} | no | |
| Texas Instruments Extended VoIP Quality Block | | | |
| Post-repair Loss RLE Report Block | {{?RFC5725}} | no | |
| Multicast Acquisition Report Block | {{?RFC6332}}  | no | |
| IDMS Report Block | {{?RFC7272}} | no | |
| ECN Summary Report | {{?RFC6679}} | partly | QUIC does not provide info about duplicates |
| Measurement Information Block | {{?RFC6776}} | no | |
| Packet Delay Variation Metrics Block | {{?RFC6798}} | no | QUIC timestamps may be used to achieve the same goal? |
| Delay Metrics Block | {{?RFC6843}} | no | QUIC has RTT and can provide timestamps for one-way delay, but no way of informing peers about end-to-end statistics when QUIC is only used on one segment of the path. |
| Burst/Gap Loss Summary Statistics Block | {{?RFC7004}} | | QUIC ACKs? |
| Burst/Gap Discard Summary Statistics Block | {{?RFC7004}}  | no | |
| Frame Impairment Statistics Summary | {{?RFC7004}}   | no | |
| Burst/Gap Loss Metrics Block | {{?RFC6958}} | | QUIC ACKs? |
| Burst/Gap Discard Metrics Block | {{?RFC7003}} | no | |
| MPEG2 Transport Stream PSI-Independent Decodability Statistics Metrics Block | {{?RFC6990}} | no | |
| De-Jitter Buffer Metrics Block | {{?RFC7005}} | no | |
| Discard Count Metrics Block | {{?RFC7002}} | no | |
| DRLE (Discard RLE Report) | {{?RFC7097}} | no | |
| BDR (Bytes Discarded Report) | {{?RFC7243}} | no | |
| RFISD (RTP Flows Initial Synchronization Delay) | {{?RFC7244}}  | no | |
| RFSO (RTP Flows Synchronization Offset Metrics Block) | {{?RFC7244}} | no | |
| MOS Metrics Block | {{?RFC7266}}  | no | |
| LCB (Loss Concealment Metrics Block) | {{?RFC7294, Section 4.1}} | no | |
| CSB (Concealed Seconds Metrics Block) | {{?RFC7294, Section 4.1}}  | no | |
| MPEG2 Transport Stream PSI Decodability Statistics Metrics Block | {{?RFC7380}}  | no | |
| Post-Repair Loss Count Metrics Report Block | {{?RFC7509}} | no | |
| Video Loss Concealment Metric Report Block | {{?RFC7867}} | no | |
| Independent Burst/Gap Discard Metrics Block | {{?RFC8015}}  | no | |

## RTP Header extensions {#rtp-header-extensions}

Like the payload-specific feedback packets, QUIC cannot directly replace the
control information in the following header extensions. RoQ does not place
restrictions on sending any RTP header extensions. However, some extensions,
such as Transmission Time offsets {{?RFC5450}} are used to improve network
jitter calculation, which can be done in QUIC if a timestamp extension is used.

### Compact Header Extensions

| Extension URI | Description | Reference | QUIC |
| ------------- | ----------- | --------- | ---- |
| urn:ietf:params:rtp-hdrext:toffset | Transmission Time offsets | [RFC5450] | no |
| urn:ietf:params:rtp-hdrext:ssrc-audio-level | Audio Level | [RFC6464] | no |
| urn:ietf:params:rtp-hdrext:splicing-interval | Splicing Interval | [RFC8286] | no |
| urn:ietf:params:rtp-hdrext:smpte-tc | SMPTE time-code mapping | [RFC5484] | no |
| urn:ietf:params:rtp-hdrext:sdes | Reserved as base URN for RTCP SDES items that are also defined as RTP compact header extensions. | [RFC7941] | no |
| urn:ietf:params:rtp-hdrext:ntp-64 | Synchronisation metadata: 64-bit timestamp format | [RFC6051] | no |
| urn:ietf:params:rtp-hdrext:ntp-56 | Synchronisation metadata: 56-bit timestamp format | [RFC6051] | no |
| urn:ietf:params:rtp-hdrext:encrypt | Encrypted extension header element | [RFC6904] | no, but maybe irrelevant? |
| urn:ietf:params:rtp-hdrext:csrc-audio-level | Mixer-to-client audio level indicators | [RFC6465] | no |
| urn:3gpp:video-orientation:6 | Higher granularity (6-bit) coordination of video orientation (CVO) feature, see clause 6.2.3 | [3GPP TS 26.114, version 12.5.0] | probably not(?) |
| urn:3gpp:video-orientation | Coordination of video orientation (CVO) feature, see clause 6.2.3 | [3GPP TS 26.114, version 12.5.0] | probably not(?) |
| urn:3gpp:roi-sent | Signalling of the arbitrary region-of-interest (ROI) information for the sent video, see clause 6.2.3.4 | [3GPP TS 26.114, version 13.1.0] | probably not(?) |
| urn:3gpp:predefined-roi-sent | Signalling of the predefined region-of-interest (ROI) information for the sent video, see clause 6.2.3.4 | [3GPP TS 26.114, version 13.1.0] | probably not(?) |

### SDES Compact Header Extensions

| Extension URI | Description | Reference | QUIC |
| ------------- | ----------- | --------- | ---- |
| urn:ietf:params:rtp-hdrext:sdes:cname | Source Description: Canonical End-Point Identifier (SDES CNAME) | [RFC7941] | no |
| urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id | RTP Stream Identifier | [RFC8852] | no |
| urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id | RTP Repaired Stream Identifier | [RFC8852] | no |
| urn:ietf:params:rtp-hdrext:sdes:CaptId | CLUE CaptId | [RFC8849] | no |
| urn:ietf:params:rtp-hdrext:sdes:mid | Media identification | [RFC9143] | no |

# Error Handling {#error-handling}

The following error codes are defined for use when abruptly terminating streams,
aborting reading of streams, or immediately closing RoQ connections.

ROQ\_NO\_ERROR (0x????):
: No error. This is used when the connection or stream needs to be closed, but
there is no error to signal.

ROQ\_INTERNAL\_ERROR (0x????):
: An internal error has occured in the RoQ stack.

ROQ\_PACKET\_ERROR (0x????):
: Invalid payload format, e.g., length does not match packet, invalid flow id
encoding, non-RTP on RTP-flow ID, etc.

ROQ\_STREAM\_CREATION\_ERROR (0x????):
: The endpoint detected that its peer created a stream that violates the ROQ protocol.

ROQ\_FRAME\_CANCELLED (0x????):
: A receiving endpoint is using STOP_SENDING on the current stream to request
new frames be sent on new streams. Similarly, a sender notifies a receiver that
retransmissions of a frame were stopped using RESET\_STREAM and new frames will
be sent on new streams.

ROQ\_UNKNOWN\_FLOW\_ID (0x????):
: An endpoint was unable to handle a flow identifier, e.g., because it was not
signalled or because the endpoint does not support multiplexing using arbitrary
flow identifiers.

ROQ\_SIGNALING\_ERROR (0x????):
: Parameters that were set during out of band signaling are incompatible with
the connection properties that were negotiated when the connection was
established using QUIC transport parameters.

# API Considerations {#api-considerations}

The mapping described in the previous sections poses some interface requirements
on the QUIC implementation. Although a basic mapping should work without any of
these requirements most of the optimizations regarding rate adaptation and
RTCP mapping require certain functionalities to be exposed to the application.
The following to sections contain a list of information that is required by an
application to implement different optimizations ({{quic-api-read}}) and
functions that a QUIC implementation SHOULD expose to an application
({{quic-api-write}}).

Each item in the following list can be considered individually. Any exposed
information or function can be used by RoQ regardless of whether the
other items are available. Thus, RoQ does not depend on the
availability of all of the listed features but can apply different optimizations
depending on the functionality exposed by the QUIC implementation.

## Information to be exported from QUIC {#quic-api-read}

This section provides a list of items that an application might want to export
from an underlying QUIC implementation. It is thus RECOMMENDED that a QUIC
implementation exports the listed items to the application.

* *Maximum Datagram Size*: The maximum datagram size that the QUIC connection
  can transmit.
* *Datagram Acknowledgment and Loss*: {{Section 5.2 of !RFC9221}} allows QUIC
  implementations to notify the application that a QUIC Datagram was
  acknowledged or that it believes a datagram was lost. The exposed information
  SHOULD include enough information to allow the application to maintain a
  mapping between the datagram that was acknowledged/lost and the RTP packet
  that was sent in that datagram.
* *Stream States*: The QUIC implementation SHOULD expose to a sender, how much
  of the data that was sent on a stream was successfully delivered and how much
  data is still outstanding to be sent or retransmitted.
* *Arrival timestamps*: If the QUIC connection uses a timestamp extension like
  {{I-D.draft-smith-quic-receive-ts}} or {{I-D.draft-huitema-quic-ts}}, the
  arrival timestamps or one-way delays SHOULD be exposed to the application.
* *Bandwidth Estimation*: If congestion control is done at the transport layer
  in the QUIC implementation, the QUIC implementation SHOULD expose an
  estimation of the currently available bandwidth to the application. Exposing
  the bandwidth estimation avoids the implementation of an additional bandwidth
  estimation algorithm in the application.
* *ECN*: If ECN marks are available, they SHOULD be exposed to the application.

## Functions to be exposed by QUIC {#quic-api-write}

This sections lists functions that a QUIC implementation SHOULD expose to an
application to implement different features of the mapping described in the
previous sections of this document.

* *Cancel Streams*: To allow an application to cancel (re)transmission of
  packets that are no longer needed, the QUIC implementation MUST expose a way
  to cancel the corresponding QUIC streams.
* *Configure Congestion Controller*: If congestion control is to be implemented
  at the QUIC connection layer as described in {{cc-quic-layer}}, the QUIC
  implementation SHOULD expose an API to allow the application to configure the
  specifics of the congestion controller.

# Discussion

## Impact of Connection Migration

RTP sessions are characterized by a continuous flow of packets into either or
both directions.  A connection migration may lead to pausing media
transmission until reachability of the peer under the new address is validated.
This may lead to short breaks in media delivery in the order of RTT and, if
RTCP is used for RTT measurements, may cause spikes in observed delays.
Application layer congestion control mechanisms (and also packet repair schemes
such as retransmissions) need to be prepared to cope with such spikes.

If a QUIC connection is established via a signaling channel, this signaling
may have involved Interactive Connectivity Establishment (ICE) exchanges to
determine and choose suitable (IP address, port number) pairs for the QUIC
connection.  Subsequent address change events may be noticed by QUIC via its
connection migration handling and/or at the ICE or other application layer,
e.g., by noticing changing IP addresses at the network interface.  This may
imply that the two signaling and data "layers" get (temporarily) out of sync.

> **Editor's Note:** It may be desirable that the API provides an indication
> of connection migration event for either case.

## 0-RTT considerations

For repeated connections between peers, the initiator of a QUIC connection can
use 0-RTT data for both QUIC streams and datagrams. As such packets are subject to
replay attacks, applications shall carefully specify which data types and operations
are allowed.  0-RTT data may be beneficial for use with RoQ to reduce the
risk of media clipping, e.g., at the beginning of a conversation.

This specification defines carrying RTP on top of QUIC and thus does not finally
define what the actual application data are going to be.  RTP typically carries
ephemeral media contents that is rendered and possibly recorded but otherwise
causes no side effects. Moreover, the amount of data that can be carried as 0-RTT
data is rather limited.  But it is the responsibility of the respective application
to determine if 0-RTT data is permissible.

> **Editor's Note:** Since the QUIC connection will often be created in the context
> of an existing signaling relationship (e.g., using WebRTC or SIP), specific 0-RTT
> keying material could be exchanged to prevent replays across sessions.  Within
> the same connection, replayed media packets would be discarded as duplicates by
> the receiver.

# Security Considerations {#sec-considerations}

RoQ is subject to the security considerations of RTP described in
{{Section 9 of !RFC3550}} and the security considerations of any RTP profile in
use.

The security considerations for the QUIC protocol and datagram extension
described in {{Section 21 of !RFC9000}}, {{Section 9 of !RFC9001}}, {{Section 8
of !RFC9002}} and {{Section 6 of !RFC9221}} also apply to RoQ.

Note that RoQ provides mandatory security, and other RTP transports do
not. In order to prevent the inadvertent disclosure of RTP sessions to
unintended third parties, RTP topologies described in {{topologies}} that
include middleboxes supporting both RoQ and non-RoQ paths
MUST forward RTP packets on non-RoQ paths using a secure AVP profile
({{?RFC3711}}, {{?RFC4585}}, or another AVP profile providing equivalent
RTP-level security), whether or not RoQ senders are using a secure AVP
profile for those RTP packets.

# IANA Considerations {#iana-considerations}

## Registration of a RoQ Identification String

This document creates a new registration for the identification of RoQ
in the "TLS Application-Layer Protocol Negotiation (ALPN) Protocol IDs" registry
{{?RFC7301}}.

The "rtp-mux-quic" string identifies RoQ:

  Protocol:
  : RTP over QUIC (RoQ)

  Identification Sequence:
  : 0x72 0x74 0x70 0x2D 0x6F 0x76 0x65 0x72 0x2D 0x71 0x75 0x69 0x63 ("rtp-mux-quic")

  Specification:
  : This document

## RoQ Error Codes

This document establishes a registry for RoQ error codes.

| Name                          | Value  | Description                            | Specification  |
| ----------------------------- | ------ | -------------------------------------- | -------------- |
| ROQ\_NO\_ERROR                | 0x???? | No Error                               | TODO: This doc |
| ROQ\_INTERNAL\_ERROR          | 0x???? | Internal Error                         | TODO: This doc |
| ROQ\_PACKET\_ERROR            | 0x???? | Invalid payload format                 | TODO: This doc |
| ROQ\_STREAM\_CREATION\_ERROR  | 0x???? | Invalid stream type                    | TODO: This doc |
| ROQ\_FRAME\_CANCELLED         | 0x???? | Frame cancelled                        | TODO: This doc |
| ROQ\_UNKNOWN\_FLOW\_ID        | 0x???? | Unknown Flow ID                        | TODO: This doc |
| ROQ\_SIGNALING\_ERROR         | 0x???? | Externally signalled requirement unmet | TODO: This doc |

--- back

# List of optional QUIC Extensions

The following is a list of QUIC protocol extensions that might be beneficial for
RoQ, but are not required by RoQ.

* *An Unreliable Datagram Extension to QUIC* {{?RFC9221}}. Without support for
  unreliable datagrams, RoQ cannot use the encapsulation specified in
  {{quic-datagrams}}, but can still use QUIC streams as specified in
  {{quic-streams}}.
* A version of QUIC receive timestamps can be helpful for improved jitter
  calculations and congestion control.
  * *Quic Timestamps For Measuring One-Way Delays*
    {{?I-D.draft-huitema-quic-ts}}
  * *QUIC Extension for Reporting Packet Receive Timestamps*
    {{?I-D.draft-smith-quic-receive-ts}}
* *QUIC Acknowledgement Frequency* {{?I-D.draft-ietf-quic-ack-frequency}} can
  be used by a sender to optimize the acknowledgement behaviour of the receiver,
  e.g., to optimize congestion control.
* *Signaling That a QUIC Receiver Has Enough Stream Data*
  {{?I-D.draft-thomson-quic-enough}} and *Reliable QUIC Stream Resets*
  {{?I-D.draft-ietf-quic-reliable-stream-reset}} would allow RoQ senders and
  receivers to use versions of CLOSE\_STREAM and STOP\_SENDING that contain
  offsets. The offset could be used to reliably retransmit all frames up to a
  certain frame that should be cancelled before resuming transmission of further
  frames on new QUIC streams.

# Experimental Results

An experimental implementation of the mapping described in this document can be
found on [Github](https://github.com/mengelbart/rtp-over-quic). The application
implements the RoQ Datagrams mapping and implements SCReAM congestion
control at the application layer. It can optionally disable the builtin QUIC
congestion control (NewReno). The endpoints only use RTCP for congestion control
feedback, which can optionally be disabled and replaced by the QUIC connection
statistics as described in {{transport-layer-feedback}}.

Experimental results of the implementation can be found on
[Github](https://github.com/mengelbart/rtp-over-quic-mininet), too.


# Acknowledgments
{:numbered="false"}

Early versions of this document were similar in spirit to
{{?I-D.draft-hurst-quic-rtp-tunnelling}}, although many details differ. The
authors would like to thank Sam Hurst for providing his thoughts about how QUIC
could be used to carry RTP.

The authors would like to thank Bernard Aboba, David Schinazi, Lucas Pardue,
Sergio Garcia Murillo,  Spencer Dawkins, and Vidhi Goel for their valuable
comments and suggestions contributing to this document.
