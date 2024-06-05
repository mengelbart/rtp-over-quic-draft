---
title: "RTP over QUIC (RoQ)"
docname: draft-ietf-avtcore-rtp-over-quic-latest
category: exp
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
    target: https://portal.3gpp.org/desktopmodules/Specifications/SpecificationDetails.aspx?specificationId=1404
    title: "IP Multimedia Subsystem (IMS); Multimedia telephony; Media handling and interaction"
    date: 2023-01-05

  Copa:
    target: https://web.mit.edu/copa/
    title: "Copa: Practical Delay-Based Congestion Control for the Internet"
    date: 2018

  IANA-RTCP-PT:
    target: https://www.iana.org/assignments/rtp-parameters/rtp-parameters.xhtml#rtp-parameters-4
    title: "RTCP Control Packet Types (PT)"

  IANA-RTCP-XR-BT:
    target: https://www.iana.org/assignments/rtcp-xr-block-types/rtcp-xr-block-types.xhtml#rtcp-xr-block-types-1
    title: "RTCP XR Block Type"

  IANA-RTCP-FMT-RTPFB-PT:
    target: https://www.iana.org/assignments/rtp-parameters/rtp-parameters.xhtml#rtp-parameters-8
    title: "FMT Values for RTPFB Payload Types"

  IANA-RTCP-FMT-PSFB-PT:
    target: https://www.iana.org/assignments/rtp-parameters/rtp-parameters.xhtml#rtp-parameters-9
    title: "FMT Values for PSFB Payload Types"

  IANA-RTP-CHE:
    target: https://www.iana.org/assignments/rtp-parameters/rtp-parameters.xhtml#rtp-parameters-10
    title: "RTP Compact Header Extensions"

  IANA-RTP-SDES-CHE:
    target: https://www.iana.org/assignments/rtp-parameters/rtp-parameters.xhtml#sdes-compact-header-extensions
    title: "RTP SDES Compact Header Extensions"

  IEEE-1733-2011:
    target: https://standards.ieee.org/ieee/1733/4748/
    title: "IEEE 1733-2011 Standard for Layer 3 Transport Protocol for Time-Sensitive Applications in Local Area Networks"

  VJMK88:
    target: https://ee.lbl.gov/papers/congavoid.pdf
    title: "Congestion Avoidance and Control"
    date: November 1988

  RTP-over-QUIC:
    target: https://github.com/mengelbart/rtp-over-quic
    title: RTP over QUIC

  RoQ-Mininet:
    target: https://github.com/mengelbart/rtp-over-quic-mininet
    title: Congestion Control for RTP over QUIC Simulations

  roq:
    target: https://github.com/mengelbart/roq
    title: RTP over QUIC (RoQ)

  gst-roq:
    target: https://github.com/bbc/gst-roq
    title: RTP-over-QUIC elements for GStreamer

  quic-go:
    target: https://github.com/quic-go/quic-go
    title: A QUIC implementation in pure Go

--- abstract

This document specifies a minimal mapping for encapsulating Real-time Transport
Protocol (RTP) and RTP Control Protocol (RTCP) packets within the QUIC protocol.
This mapping is called RTP over QUIC (RoQ).

This document also discusses how to leverage state that is already available
from the QUIC implementation in the endpoints, in order to reduce the need to
exchange RTCP packets, and describes different options for implementing congestion control and rate
adaptation for RTP without relying on RTCP feedback.

--- middle

# Introduction

This document specifies a minimal mapping for encapsulating Real-time Transport
Protocol (RTP) {{!RFC3550}} and RTP Control Protocol (RTCP) {{!RFC3550}} packets
within the QUIC protocol ({{!RFC9000}}).
This mapping is called RTP over QUIC (RoQ).

This document also discusses how to leverage state that is already available
from the QUIC implementation in the endpoints, in order to reduce the need to
exchange RTCP packets, and describes different options for implementing congestion control and rate
adaptation for RTP without relying on RTCP feedback.

## Background {#background}

The Real-time Transport Protocol (RTP) {{!RFC3550}} is generally used to carry
real-time media for conversational media sessions, such as video conferences,
across the Internet.  Since RTP requires real-time delivery and is tolerant to
packet losses, the default underlying transport protocol has historically been
UDP {{?RFC0768}}, but a large variety of other underlying transport protocols
have been defined for various reasons (e.g., securing media exchange, or
providing a fallback when UDP is blocked along a network path). This document
describes RTP over QUIC, providing one more underlying transport protocol. The
reasons for using QUIC as an underlying transport protocol are given in
{{motivations}}.

This document describes an application usage of QUIC ({{?RFC9308}}).
As a baseline, the document does not expect more than a standard QUIC implementation
as defined in {{!RFC8999}}, {{!RFC9000}}, {{!RFC9001}}, and {{!RFC9002}},
providing a secure end-to-end transport.
Beyond this baseline, real-time applications can benefit from QUIC extensions such as unreliable DATAGRAMs
{{!RFC9221}}, which provides additional desirable properties for
real-time traffic (e.g., no unnecessary retransmissions, avoiding head-of-line
blocking).

## What's in Scope for this Document {#in-scope}

This document focuses on providing a secure encapsulation of RTP and RTCP packets
for transmission over QUIC. The expected usage is wherever RTP is used to carry
media packets, allowing QUIC in place of other underlying transport protocols.
We expect RoQ to be used in contexts where a signaling protocol is used to announce or negotiate a media
encapsulation for RTP and the associated transport parameters (such as IP address, port
number). RoQ does not provide a stand-alone media transport capability, because at a minimum, media
transport parameters would need to be statically configured.

RoQ can be used in many of the point-to-point and multi-endpoint RTP topologies described in {{!RFC7667}}, and can be used with both decentralized and centralized control topologies.
When RoQ is used in a decentralized topology, RTP packets are exchanged directly between ultimate RTP endpoints.
When RoQ is used in a centralized topology, RTP packets transit one or more middleboxes which might function as mixers or translators between ultimate RTP endpoints.
RoQ can also be used in RTP client-server-style settings, e.g., when talking to a
conference server as described in RFC 7667 ({{!RFC7667}}), or, if RoQ
is used to replace RTSP ({{?RFC7826}}), to a media server.

Moreover, this document describes how a QUIC implementation and its API can be
extended to improve efficiency of the RoQ protocol operation.

RoQ does not limit the usage of RTP Audio Video Profiles (AVP)
({{!RFC3551}}), or any RTP-based mechanisms, although it might render some of
them unnecessary, e.g., Secure Real-Time Transport Protocol (SRTP)
({{?RFC3711}}) might not be needed, because end-to-end security is already
provided by QUIC, and double encryption by QUIC and by SRTP might have more
costs than benefits.  Nor does RoQ limit the use of RTCP-based
mechanisms, although some information or functions provided by using RTCP
mechanisms might also be available from the underlying QUIC implementation.

RoQ supports multiplexing multiple
RTP-based media streams within a single QUIC connection and thus using a single
(destination IP address, destination port number, source IP address, source port
number, protocol) 5-tuple. We note that multiple independent QUIC connections
can be established in parallel using the same 5-tuple., e.g. to carry different media channels. These connections would be
logically independent of one another.

## What's Out of Scope for this Document {#out-of-scope}

This document does not enhance QUIC for real-time media or define a
replacement for, or evolution of, RTP. Work to map other media transport
protocols to QUIC is under way elsewhere in the IETF.

This document does not specify RoQ for point-to-multipoint applications, because QUIC
itself is not defined for multicast operation. The scope of this document is
limited to unicast RTP, even though nothing would prevent its use
in multicast setups if future QUIC extensions support multicast.

RoQ does not define new congestion control and rate adaptation algorithms
for use with RTP media, and does not specify the use of particular congestion control and rate adaptation algorithms for use with RTP media. However, {{congestion-control}} discusses multiple ways that congestion control and rate adaptation could be performed at the QUIC and/or at
the RTP layer, and {{api-considerations}} describes information available at the QUIC layer that could be exposed
via an API for the benefit of RTP layer implementation.

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

> **Note to the Reader:** {{!RFC3550}} actually describes two closely-related protocols - the RTP Data Transfer Protocol {{Section 5 of !RFC3550}}, and the RTP Control Protocol {{Section 6 of !RFC3550}}. In this document, the term "RTP" refers to the combination of RTP Data Transfer Protocol and RTP Control Protocol, because the distinction isn't relevant for encapsulation, and the term "RTCP" always refers to the RTP Control Protocol.

> **Note to the Reader:** the meaning of the terms "congestion avoidance", "congestion control" and "rate adaptation" in the IETF community have evolved over the decades since "slow start" and "congestion avoidance" were added as mandatory to implement in TCP, in {{Section 4.2.2.15 of ?RFC1122}}. Historically, "congestion control" usually referred to "achieving network stability" ({{VJMK88}}), by protecting the network from senders who continue to transmit packets that exceed the ability of the network to carry them, even after packet loss occurs (called "congestion collapse").

> Modern general-purpose "congestion control" algorithms have moved beyond avoiding congestion collapse, and work to avoid "bufferbloat", which causes increasing round-trip delays, as described in {{rate-adaptation-application-layer}}.

> "Rate adaptation" more commonly refers to strategies intended to guide senders on when to send "the next packet", so that one-way delays along the network path remain minimal.

> When RTP runs over QUIC, as described in this document, QUIC is performing congestion control, and the RTP application is responsible for performing rate adaptation.

> In this document, these terms are used with the meanings listed below, with the recognition that not all the references in this document use these terms in the same way.

The following terms are used in this document:

Bandwidth Estimation:
: An algorithm to estimate the available bandwidth of a link in a network. Such
an estimation can be used for rate adaptation, i.e., adapt the rate at which an
application transmits data.

Congestion Control:
: A mechanism to limit the aggregate amount of data that has been sent over a path to a receiver but has not been acknowledged by the receiver.
This prevents a sender from overwhelming the capacity of a path between a sender and a receiver, which might cause intermediaries on the path to drop packets before they arrive at the receiver.

Datagram:
: The term "datagram" is ambiguous. Without a qualifier, "datagram" could refer to a UDP packet, or a QUIC DATAGRAM frame, as defined in QUIC's unreliable DATAGRAM extension {{!RFC9221}}, or an RTP packet encapsulated in UDP, or an RTP packet capsulated in QUIC DATAGRAM frame. This document uses the uppercase "DATAGRAM" to refer to a QUIC DATAGRAM frame and the term RoQ datagram as a short form of "RTP packet encapsulated in a QUIC DATAGRAM frame".

If not explicitly qualified, the term "datagram" in this document refers to an RTP packet, and the uppercase "DATAGRAM" refers to a QUIC DATAGRAM frame. This document also uses the term "RoQ datagram" as a short form of "RTP packet encapsulated in a QUIC DATAGRAM frame".

Endpoint:
: A QUIC client or QUIC server that participates in an RoQ session. "A RoQ endpoint" is used in this document where that seems clearer than "an endpoint" without qualification.

Early data:
: Application data carried in a QUIC 0-RTT packet payload, as defined in {{!RFC9000}}. In this document, the early data would be an RTP packet.

Frame:
: A QUIC frame as defined in {{!RFC9000}}.

Peer:
: The term "peer" is ambiguous, and without a qualifier could be understood to refer to an RTP endpoint, a RoQ endpoint, or a QUIC endpoint. In this document, a "peer" is "the other RoQ endpoint that a RoQ endpoint is communicating with", and does not have anything to do with "peer-to-peer" operation versus "client-server" operation.

Rate Adaptation:
: An application-level mechanism that adjusts the sending rate of an application in response to changing path conditions. For example, an application sending video might respond to indications of congestion by adjusting the resolution of the video it is sending.

Receiver:
: An endpoint that receives media in RTP packets and might send or receive RTCP packets.

Sender:
: An endpoint that sends media in RTP packets and might send or receive RTCP packets.

Stream:
: The term "stream" is ambiguous. Without a qualifier, "stream" could refer to a QUIC stream, as defined in {{!RFC9000}}, a series of media samples, or a series of RTP packets. If not explicitly qualified, the term "stream" in this document refers to a QUIC stream and the term "STREAM" refers to a single QUIC STREAM frame. This document also uses the term "RTP stream" or "RTCP streams" as a short form of "a series of RTP packets" or "a series of RTCP packets", the term "RoQ stream" as a short form of "one or more RTP packets encapsulated in QUIC streams" and the term "media stream" as a short form of "a series of one or more media samples".

Packet diagrams in this document use the format defined in {{Section 1.3 of RFC9000}} to
illustrate the order and size of fields.

# Protocol Overview

This document introduces a mapping of the Real-time Transport Protocol (RTP) to
the QUIC transport protocol. RoQ allows the use of both QUIC streams and
QUIC DATAGRAMs to transport real-time data, and thus, if RTP packets
are to be sent over QUIC DATAGRAMs, the QUIC
implementation MUST support QUIC's DATAGRAM extension.

{{!RFC3550}} specifies that RTP sessions need to be transmitted on different transport addresses to allow multiplexing between them.
RoQ uses a different approach to leverage the advantages of QUIC connections without managing a separate QUIC connection per RTP session.
{{!RFC9221}} does not provide demultiplexing between different flows on DATAGRAMs but suggests that an application implement a demultiplexing mechanism if required.
An example of such a mechanism would be flow identifiers prepended to each DATAGRAM frame as described in {{Section 2.1 of ?I-D.draft-ietf-masque-h3-datagram}}.
RoQ uses a flow identifier to replace the network address and port number to multiplex many RTP sessions over the same QUIC connection.

An RTP application is responsible for determining what to send in an encoded media stream, and how to send that encoded media stream within a targeted bitrate.

This document does not mandate how an application determines what to send in an encoded media stream, because decisions about what to send within a targeted bitrate, and how to adapt to changes in the targeted bitrate, can depend on the application and on the codec in use. For example, adjusting quantization in response to changing network conditions might work well in many cases, but if what's being shared is video that includes text, maintaining readability is important.

As of this writing, the IETF has produced two Experimental-track congestion control documents for real-time media, Network-Assisted Dynamic Adaptation (NADA) {{!RFC8698}} and Self-Clocked Rate Adaptation for Multimedia (SCReAM) {{!RFC8298}}.
These congestion control algorithms use feedback about the network's performance to calculate target bitrates. When these algorithms are used with RTP, the necessary feedback is generated at the receiver and sent back to the sender via RTCP.

Since QUIC itself collects some metrics about the network's performance, these QUIC
metrics can be used to generate the required feedback at the sender-side and
provide it to the congestion control algorithm to avoid the additional overhead of the
RTCP stream. This is discussed in more detail in {{rtcp-mapping}}.

## Motivation {#motivations}

From time to time, someone asks the reasonable question, "why would anyone implement and deploy RoQ"? This reasonable question deserves a better answer than "because we can". Upon reflection, the following motivations seem useful to state.

The motivations in this section are in no particular order, and this reflects the reality that not all implementers and deployers would agree on "the most important motivations".

### "Always-On" Transport-level Authentication and Encryption {#alwas-on}

Although application-level mechanisms to encrypt RTP payloads have existed since the introduction of the Secure Real-time Transport Protocol (SRTP) {{?RFC3711}}, the additional encryption of RTP header fields and contributing sources has only been defined recently (in Cryptex {{?RFC9335}}), and both SRTP and Cryptex are optional capabilities for RTP.

This is in sharp contrast to "always-on" transport-level encryption in the QUIC protocol, using Transport Layer Security (TLS 1.3) as described in {{?RFC9001}}. QUIC implementations always authenticate the entirety of each packet, and encrypt as much of each packet as is practical, even switching from "long headers", which expose the QUIC header fields needed to establish a connection, to "short headers", which only expose the absolute minimum QUIC header fields needed to identify an existing connection to the receiver, so that the QUIC payload is presented to the correct QUIC application {{?RFC8999}}.

### "Always-On" Internet-Safe Congestion Control

When RTP is carried directly over UDP, as is commonly done, the underlying UDP protocol provides multiplexing using UDP ports, but no transport services beyond multiplexing are provided to the application. All congestion control behavior is up to the RTP application itself, and if anything goes wrong with the application and this condition results in an RTP sender failing to recognize that it is contributing to path congestion, the "worst case" response is to invoke the RTP "circuit breaker" procedures {{?RFC8083}}. These procedures result in "ceasing transmission", as described in {{Section 4.5 of ?RFC8083}}. Because RTCP-based circuit breakers only detect long-lived congestion, a response based on these mechanisms will not happen quickly.

In contrast, when RTP is carried over QUIC, QUIC implementations maintain their own estimates of key transport parameters needed to detect and respond to possible congestion, and these estimates are independent of any measurements RTP senders and receivers are maintaining. The result is that even if an RTP sender attempts to "send" in the presence of persistent path congestion, QUIC congestion control procedures (for example, the procedures defined in {{?RFC9002}}) will cause the RTP packets to be buffered while QUIC responds to detected packet loss. This happens without RTP senders taking any action, but the RTP sender has no control over this QUIC mechanism.

Moreover, when a single QUIC connection is used to multiplex both RTP and non-RTP packets as described in {{single-path}}, the shared QUIC connection will still be Internet-safe, with no coordination required.

While QUIC's response to congestion ensures that RoQ will be "Internet-safe", from the network's perspective, it is helpful to remember that a QUIC sender responds to detected congestion by delaying packets that are already available to send, to give the path to the QUIC receiver time to recover from congestion.

* If the QUIC connection encapsulates RTP, this means that some RTP packets will be delayed, arriving at the receiver later than a consumer of the RTP flow might prefer.
* If the QUIC connection also encapsulates RTCP, this means that these RTCP messages will also be delayed, and will not be sent in a timely manner. This delay will impact RTT measurements using RTCP and can interfere with a sender's ability to stabilize rate control and achieve audio/video synchronization.

In summary,

* Timely RTP stream-level rate adaptation will give a better user experience by minimizing endpoint queuing delays and packet loss, but
* in the presence of packet loss, QUIC connection-level congestion control will respond more quickly and possibly more smoothly to the end of congestion than RTP "circuit breakers".

### RTP Rate Adaptation Based on QUIC Feedback {#ra-quic-feedback}

When RTP is carried directly over UDP, RTP makes use of a large number of RTP-specific feedback mechanisms because there is no other way to receive feedback.
Some of these mechanisms are specific to the type of media RTP is sending, but others can be mapped from underlying QUIC implementations that are using this feedback to perform congestion control for any QUIC connection, regardless of the application reflected in the payload.
This is described in (much) more detail in {{congestion-control}} on rate adaptation, and in {{rtcp-mapping}} on replacing RTCP and RTP header extensions with QUIC feedback.

One word of caution is in order - RTP implementations might rely on at least some minimal periodic RTCP feedback, in order to determine that an RTP flow is still active, and is not causing sustained congestion (as described in {{?RFC8083}}.
Because the necessary "periodicity" is measured in seconds, the impact of this "duplicate" feedback on path bandwidth utilization is likely close to zero.

### Path MTU Discovery and RTP Media Coalescence {#mtu-coal}

The minimum Path MTU (Maximum Transmission Unit) supported by conformant QUIC implementations is 1200 bytes {{?RFC9000}}. In addition, QUIC implementations allow senders to use either DPLPMTUD ({{?RFC8899}}) or PMTUD ({{?RFC1191}}, {{?RFC8201}}) to determine the actual Path MTU size that the receiver can accept, and that the path between sender and receiver can support. The actual Path MTU can be larger than the Minimum Path MTU.

This is especially useful in certain conferencing topologies, where otherwise senders would have no choice but to use the lowest Path MTU for all conference participants. Even in point-to-point RTP sessions, this also allows senders to piggyback audio media in the same UDP packet as video media, for example, and also allows QUIC receivers to piggyback QUIC ACK frames on any QUIC packets being transmitted in the other direction.

### Multiplexing RTP, RTCP, and Non-RTP Flows on a Single QUIC Connection {#single-path}

This document defines a flow identifier for multiplexing multiple RTP and
RTCP ports on the same QUIC connection to conserve ports, especially at NATs and
firewalls. {{multiplexing}} describes the multiplexing in more detail. Future
extensions could further build on the flow identifier to multiplex RTP with
other protocols on the same connection, as long as these protocols can co-exist
with RTP without interfering with the ability of this connection to carry
real-time media.

### Exploiting Multiple Paths {#multiple-paths}

Although there is much interest in multiplexing flows on a single QUIC connection as described in {{single-path}}, QUIC also provides the capability of establishing and validating multiple paths for a single QUIC connection as described in {{Section 9 of ?RFC9000}}. Once multiple paths have been validated, a sender can migrate from one path to another with no additional signaling, allowing an endpoint to move from one endpoint address to another without interruption, as long as only a single path is in active use at any point in time.

Connection migration could be desirable for a number of reasons, but to give one example, this allows a QUIC connection to survive address changes due to a middlebox allocating a new outgoing port, or even a new outgoing IP address.

The Multipath Extension for QUIC {{?I-D.draft-ietf-quic-multipath}} would allow the application to actively use two or more paths simultaneously, but in all other respects, this functionality is the same as QUIC connection migration.

A sender can use these capabilities to more effectively exploit multiple paths between sender and receiver with no action required from the application, even if these paths have different path characteristics.  Examples of these different path characteristics include senders handling paths differently if one path has higher available bandwidth and the other has lower one-way latency, or if one is a more costly cellular path and the other is a less costly WiFi path.

Some of these differences can be detected by QUIC itself, while other differences must be described to QUIC based on policy, etc. Possible RTP implementation strategies for path selection and utilization are not discussed in this document. Path scheduling APIs to let applications control these mechanisms are a topic for future research and might need further specification in future documents.

### Exploiting New QUIC Capabilities {#new-quic}

The first version of the QUIC protocol described in {{!RFC9000}} has been completed, but extensions to QUIC are still under active development in the IETF. Because of this, using QUIC as a transport for a mature protocol like RTP allows developers to exploit new transport capabilities as they become available.

## RTP with QUIC Streams, QUIC DATAGRAMs, and a Mixture of Both {#streams-and-datagrams}

This document describes the use of QUIC streams and DATAGRAMs as RTP encapsulations but does not take a position on which encapsulation an application ought to use. Indeed, an application can use both QUIC streams and DATAGRAM encapsulations on the same QUIC connection. The choice of encapsulation is left to the application developer, but it is worth noting differences that are relevant when making this choice.

QUIC {{!RFC9000}} was initially designed to carry HTTP {{?RFC9114}} in QUIC streams, and QUIC streams provide what HTTP application developers need - for example, a stateful, connection-oriented, flow-controlled, reliable, ordered stream of bytes to an application. QUIC streams can be multiplexed over a single QUIC connection, using stream IDs to demultiplex incoming messages.

QUIC DATAGRAMs {{!RFC9221}} were developed as a QUIC extension, intended to support applications that do not need reliable delivery of application data. This extension defines two DATAGRAM frame types (one including a length field, the other not including a length field), and these DATAGRAM frames can co-exist with QUIC streams within a single QUIC connection, sharing the connection's cryptographic and authentication context, and congestion controller context.

There is no default relative priority between DATAGRAM frames with respect to each other, and there is no default priority between DATAGRAM frames and QUIC STREAM frames. QUIC implementations can present an API to allow applications to assign relative priorities within a QUIC connection, but this is not mandated by the standard and might not be present in all implementations.

Because DATAGRAMs are an extension to QUIC, they inherit a great deal of functionality from QUIC (much of which is described in {{motivations}}); so much so that it is easier to explain what DATAGRAMs do NOT inherit.

* DATAGRAM frames do not provide any explicit flow control signaling. This means that a QUIC receiver might not be able to commit the necessary resources to process incoming frames, but the purpose for DATAGRAM frames is to carry application-level information that can be lost and will not be retransmitted.
* DATAGRAM frames cannot be fragmented. They are limited in size by the max_datagram_frame_size transport parameter, and further limited by the max_udp_payload_size transport parameter and the Path MTU between endpoints.
* DATAGRAM frames belong to a QUIC connection as a whole. There is no QUIC-level way to multiplex/demultiplex DATAGRAM frames within a single QUIC connection. Any multiplexing identifiers must be added, interpreted, and removed by an application, and they will be sent as part of the payload of the DATAGRAM frame itself.

DATAGRAM frames do inherit the QUIC connection's congestion controller. This means that although there is no frame-level flow control, DATAGRAM frames can be delayed until the controller allows them to be sent or dropped (with an optional notification to the sending application). Implementations can also delay sending DATAGRAM frames to maintain consistent packet pacing (as described in {{Section 7.7 of ?RFC9002}}), and can allow an application to specify a sending expiration time, but these capabilities are not mandated by the standard and might not be present in all implementations.

Because DATAGRAMs are an extension to QUIC, a RoQ endpoint cannot assume that its peer supports this extension.
The RoQ endpoint might discover that its peer does not support DATAGRAMs in one of two ways:

* as part of the signaling process to set up QUIC connections, or
* during negotiation of the DATAGRAM extension during the QUIC handshake.

When either of these situations happen, the RoQ endpoint needs to make a decision about what to do next.

* If the use of DATAGRAMs was critical for the application, the endpoint can simply close the QUIC connection, allowing someone or something to correct this mismatch, so that DATAGRAMs can be used.
* If the use of DATAGRAMs was not critical for the application, the endpoint can negotiate the use of QUIC streams instead.

## Supported RTP Topologies {#topologies}

RoQ supports only some of the RTP topologies described in
{{?RFC7667}}. Most notably, due to QUIC {{!RFC9000}} being a purely IP unicast
protocol at the time of writing, RoQ cannot be used as a transport
protocol for any of the paths that rely on IP multicast in several multicast
topologies (e.g., *Topo-ASM*, *Topo-SSM*, *Topo-SSM-RAMS*).

Some "multicast topologies" can include IP unicast paths (e.g., *Topo-SSM*,
*Topo-SSM-RAMS*). In these cases, the unicast paths can use RoQ.

RTP supports different types of translators and mixers. Whenever a middlebox
needs to access the content of QUIC frames (e.g., *Topo-PtP-Translator*, *Topo-PtP-Relay*, *Topo-Trn-Translator*, *Topo-Media-Translator*),
the QUIC connection will be terminated at that middlebox.

RoQ streams (see {{quic-streams}}) can support much larger RTP
packet sizes than other transport protocols such as UDP can, which can lead to
problems when transport translators which translate from RoQ to RTP
over a different transport protocol. A similar problem can occur if a translator
needs to translate from RTP over UDP to RoQ over DATAGRAMs, where the max_datagram_frame_size
of a QUIC DATAGRAM can be smaller than the MTU of a UDP datagram. In both cases,
the translator might need to rewrite the RTP packets to fit into the smaller MTU
of the other protocol. Such a translator might need codec-specific knowledge to
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
: Not supported, because QUIC {{!RFC9000}} does not support NAT traversal.

Note-MTU:
: Supported, but might require MTU adaptation.

Note-Sec:
: Note that because RoQ uses QUIC as its underlying transport, and QUIC authenticates the entirety of each packet and encrypts as much of each packet as is practical, RoQ secures both RTP headers and RTP payloads, while other RTP transports
do not. {{sec-considerations}} describes strategies to prevent the inadvertent
disclosure of RTP sessions to unintended third parties.

Note-MCast:
: Not supported, because QUIC {{!RFC9000}} does not support IP multicast.

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
: Supported for IP unicast paths between RTP sources and translators.

Note-Topo-Mixer:
: Supported for IP unicast paths between RTP senders and mixers.

Note-Warn:
: Quote from {{?RFC7667}}: *This topology is so problematic and it is so easy to get the RTCP processing wrong, that it is NOT RECOMMENDED to implement this topology.*

# Connection Establishment and Application-Layer Protocol Negotiation {#alpn}

QUIC requires the use of Application-Layer Protocol Negotiation (ALPN) {{!RFC7301}} tokens during connection setup. RoQ
uses "roq" as the ALPN token, included as part of the TLS handshake (see also
{{iana-considerations}}).

Note that the "roq" ALPN token is not tied to a specific RTP profile, even
though the RTP profile could be considered part of the application usage.  This allows
different RTP sessions, which might use different RTP profiles, to be
carried within the same QUIC connection.

## Draft version identification

> **RFC Editor's note:** Please remove this section prior to publication of a
> final version of this document.

RoQ uses the ALPN token "roq" to identify itself during QUIC connection setup.

Only implementations of the final, published RFC can identify themselves as
"roq". Until such an RFC exists, implementations MUST NOT identify themselves
using this string.

Implementations of draft versions of the protocol MUST add the string "-" and
the corresponding draft number to the identifier.  For example,
draft-ietf-avtcore-rtp-over-quic-09 is identified using the string "roq-09".

Non-compatible experiments that are based on these draft versions MUST append
the string "-" and an experiment name to the identifier.

# Encapsulation {#encapsulation}

This section describes the encapsulation of RTP packets in QUIC.

QUIC supports two transport methods: QUIC streams {{!RFC9000}} and DATAGRAMs {{!RFC9221}}. This document specifies mappings of RTP to both transport modes.
Senders MAY combine both modes by sending some RTP packets over the same or different QUIC streams and others in DATAGRAMs.

{{multiplexing}} introduces a multiplexing mechanism that supports multiplexing multiple RTP sessions and RTCP. {{quic-streams}} and {{quic-datagrams}} explain
the specifics of mapping RTP to QUIC streams and DATAGRAMs, respectively.

## Multiplexing {#multiplexing}

RoQ uses flow identifiers to multiplex different RTP streams on a
single QUIC connection. A flow identifier is a QUIC variable-length integer as
described in {{Section 16 of !RFC9000}}. Each flow identifier is associated with
an RTP stream.

In a QUIC connection using the ALPN token defined in {{alpn}}, every DATAGRAM and every QUIC stream MUST start with a flow identifier.
An endpoint MUST NOT send any data in a DATAGRAM or stream that is not associated with the flow
identifier which started the DATAGRAM or stream.

RTP packets of different RTP sessions MUST use distinct flow
identifiers. If endpoints wish to send multiple types of media in a single RTP
session, they can do so by following the guidance specified in {{?RFC8860}}.

A single RTP session can be associated with one or two flow identifiers. Thus,
it is possible to send RTP and RTCP packets belonging to the same session using
different flow identifiers. RTP and RTCP packets of a single RTP session can use
the same flow identifier (following the procedures defined in {{?RFC5761}}), or
they can use different flow identifiers.

The association between flow identifiers and RTP streams MUST be negotiated
using appropriate signaling. The signaling happens out of band and thus a stream
or DATAGRAM with a given flow identifer can arrive before the signaling
finished. In that case, an endpoint cannot associate the stream or DATAGRAM with
the corresponding RTP stream. The endpoint can buffer streams and DATAGRAMs
using an unknown flow identifier until they can be associated with the
corresponding RTP stream. To avoid resource exhaustion, the buffering endpoint
MUST limit the number of streams and DATAGRAMs to buffer. If the number of
buffered streams exceeds the limit on buffered streams, the endpoint MUST send a
STOP_SENDING with the error code ROQ_UNKNOWN_FLOW_ID. It is an implementation's
choice on which stream to send STOP_SENDING. If the number of buffered DATAGRAMs
exceeds the limit on buffered DATAGRAMs, the endpoint MUST drop a DATAGRAMs. It
is an implementation's choice which DATAGRAMs to drop.

Flow identifiers introduce some overhead in addition to the header overhead of
RTP and QUIC. QUIC variable-length integers require between one and eight
bytes depending on the number expressed. Thus, using low
numbers as session identifiers first will minimize this additional overhead.

## QUIC Streams {#quic-streams}

To send RTP packets over QUIC streams, a sender MUST open at least one new unidirectional QUIC stream.
RoQ uses unidirectional streams, because there is no synchronous relationship between sent and received RTP packets.
An endpoint that receives a bidirectional stream with a flow identifier that is associated with an RTP stream, MUST stop reading from the stream and send a CONNECTION_CLOSE frame with the frame type set to APPLICATION_ERROR and the error code set to ROQ_STREAM_CREATION_ERROR.

The underlying QUIC implementation might be acting as either a QUIC client or QUIC server, so the unidirectional QUIC stream can be either client-initiated or server-initiated, as described in {{Section 2.1 of !RFC9000}}, depending on the role.
The QUIC implementation's role is not controlled by RoQ, and can be negotiated using a separate signaling protocol.

A RoQ sender can open new QUIC streams for different RTP packets using the same flow identifier. This allows RoQ senders to use QUIC streams while avoiding head-of-line blocking.

Because a sender can continue sending on a stream with a lower stream identifier after starting packet transmission on a stream with a higher stream identifier, a RoQ receiver MUST be prepared to receive RoQ packets on any number of QUIC streams (subject to its limit on parallel open streams) and MUST NOT make assumptions about which RTP sequence numbers are carried in any particular stream.

### Stream Encapsulation

{{fig-stream-payload}} shows the encapsulation format for RoQ Streams.

~~~
Payload {
  Flow Identifier (i),
  RTP Payload(..) ...,
}
~~~
{: #fig-stream-payload title="RoQ Streams Payload Format"}

Flow Identifier:

: Flow identifier to demultiplex different data flows on the same QUIC
connection.

RTP Payload:

: Contains the RTP payload; see {{fig-rtp-stream-payload}}

The payload in a QUIC stream starts with the flow identifier followed by one or
more RTP payloads. All RTP payloads sent on a stream MUST belong to
the RTP session with the same flow identifier.

Each payload begins with a length field indicating the length of the RTP
packet, followed by the packet itself, see {{fig-rtp-stream-payload}}.

~~~
RTP Payload {
  Length(i),
  RTP Packet(..),
}
~~~
{: #fig-rtp-stream-payload title="RTP payload for QUIC streams"}

Length:

: A QUIC variable length integer (see {{Section 16 of !RFC9000}}) describing the
length of the following RTP packets in bytes.

RTP Packet:

: The RTP packet to transmit.

### Media Frame Cancellation

QUIC uses RESET\_STREAM and STOP\_SENDING frames to terminate the sending part
of a stream and to request termination of an incoming stream by the sending
peer respectively.

A RoQ receiver that is no longer interested in reading a certain portion of
the media stream can signal this to the sending peer using a STOP\_SENDING
frame.

If a RoQ sender discovers that a packet is no longer needed and knows that the packet has not yet been successfully and completely transmitted, it can use RESET\_STREAM to tell the RoQ receiver that the RoQ sender is discarding the packet.

In both cases, the error code of the RESET\_STREAM frame or the STOP\_SENDING
frame MUST be set to ROQ\_FRAME\_CANCELLED.

STOP\_SENDING is not a request to the sender to stop sending RTP media, only an indication that a RoQ receiver stopped reading the QUIC stream being used to carry that RTP media.
This can mean that the RoQ receiver is no longer able to use the media frames being received because they are "too old".
A sender with additional media frames to send can continue sending them on another QUIC stream.
Alternatively, new media frames can be sent as DATAGRAMs (see {{quic-datagrams}}).
In either case, a RoQ sender resuming operation after receiving STOP_SENDING can continue starting with the newest media frames available for sending. This allows a RoQ receiver to "fast forward" to media frames that are "new enough" to be used.

Any media frame that has already been sent on the QUIC stream that received the STOP\_SENDING frame, MUST NOT be sent again on the new QUIC stream(s) or DATAGRAMs.

Note that an RTP receiver cannot request a reset of a particular media
frame because the sending QUIC implementation might already have sent data for
one or more following media frames on the same stream. In that case, STOP\_SENDING and the
resulting RESET\_STREAM would also discard the following media frames and thus lead
to unintentionally skipping one or more media frames.

A translator that translates between two endpoints, both connected via QUIC,
MUST forward RESET\_STREAM frames received from one end to the other unless it
forwards the RTP packets encapsulated in DATAGRAMs.

QUIC implementations will fragment large RTP packets into smaller QUIC STREAM
frames. The data carried in these QUIC STREAM frames is transmitted reliably and
is delivered to the receiving application in order, so that a receiving application can read a complete RTP packet from
the stream as long as the stream is not closed with a RESET\_STREAM frame. No
retransmission has to be implemented by the application since data that was
carried in QUIC frames that were lost in transit is retransmitted by QUIC.

### Flow control and MAX\_STREAMS {#quic-flow-cc}

In order to permit QUIC streams to open, a RoQ sender MUST configure non-zero minimum values for the number of permitted streams and the initial stream flow-control window.
These minimum values control the number of parallel, or simultaneously active, RTP flows.
Endpoints that excessively restrict the number of streams or the flow-control window of these streams will increase the chance that the sending peer reaches the limit early and becomes blocked.

Opening new streams for new packets can implicitly limit the number of packets
concurrently in transit because the QUIC receiver provides an upper bound of
parallel streams, which it can update using QUIC MAX\_STREAMS frames. The number
of packets that can be transmitted concurrently depends on several factors,
such as the number of RTP streams within a QUIC connection, the bitrate of the
media streams, and the maximum acceptable transmission delay of a given packet.
Receivers are responsible for providing senders enough credit to open new
streams for new packets at any time.

As an example, consider a conference scenario
with 20 participants. Each participant receives audio and video streams of every
other participant from a central RTP middlebox. If the sender opens a new QUIC stream
for every frame at 30 frames per second video and 50 frames per second audio, it
will open 1520 new QUIC streams per second. A receiver must provide at least
that many credits for opening new unidirectional streams to the RTP middlebox every
second.

In addition, the receiver ought to also consider the requirements of RTCP streams.
These considerations can also be relevant when implementing signaling since it
can be necessary to inform the receiver about how many stream
credits it will have to provide to the sending peer, and how rapidly it must provide these stream credits.

## QUIC DATAGRAMs {#quic-datagrams}

Senders can also transmit RTP packets in QUIC DATAGRAMs, using
a QUIC extension described in {{!RFC9221}}.
DATAGRAMs can only be used if the use of the DATAGRAM extension was successfully negotiated during the QUIC handshake.
If the DATAGRAM extension was negotiated using a signaling protocol, but was not also negotiated during the resulting QUIC handshake, an endpoint can close the connection with the ROQ\_EXPECTATION\_UNMET error code.

DATAGRAMs preserve application frame boundaries.
Thus, a single RTP packet can be mapped to a single DATAGRAM without additional framing.
Because QUIC DATAGRAMs cannot be IP-fragmented ({{Section 5 of !RFC9221}}), senders need to consider the header overhead associated with DATAGRAMs, and ensure that the RTP packets, including their payloads, flow identifier, QUIC, and IP headers, will fit into the Path MTU.

{{fig-dgram-payload}} shows the encapsulation format for RoQ
Datagrams.

~~~
Payload {
  Flow Identifier (i),
  RTP Packet (..),
}
~~~
{: #fig-dgram-payload title="RoQ Datagram Payload Format"}

Flow Identifier:

: Flow identifier to demultiplex different data flows on the same QUIC
connection.

RTP Packet:

: The RTP packet to transmit.

RoQ senders need to be aware that QUIC uses the concept of QUIC frames, and QUIC connections use
different kinds of QUIC frames to carry different application and control data types.
A single QUIC packet can contain more than one QUIC frame, including, for example, QUIC STREAM frames or DATAGRAM frames carrying application data and ACK frames carrying QUIC acknowledgments, as long as the overall size fits into the MTU.
One implication is that the number of packets a QUIC stack transmits depends on whether it can fit ACK and DATAGRAM frames in the same QUIC packet.
Suppose the application creates many DATAGRAM frames that fill up the QUIC packet.
In that case, the QUIC stack would need to create additional packets for ACK frames, and possibly other control frames.
The additional overhead could, in some cases, be reduced if the application creates smaller RTP packets, such that the resulting DATAGRAM frame can fit into a QUIC packet that can also carry ACK frames. Another implication is that multiple RTP packets in either QUIC streams or QUIC DATAGRAMs might be encapsulated in a single QUIC packet, which is discussed in more detail in {{coalescing-packets}}.

Since DATAGRAMs are not retransmitted on loss (see also
{{transport-layer-feedback}} for loss signaling), if an application is using DATAGRAMs and wishes to
retransmit lost RTP packets, the application has to carry out that retransmission.
RTP retransmissions can be done in the same RTP session or in a
different RTP session {{!RFC4588}} and the flow identifier MUST be set to the
flow identifier of the RTP session in which the retransmission happens.

# Connection Shutdown

Either endpoint can close the connection for any of a variety of reasons. If one of the
endpoints wants to close the RoQ connection, the endpoint can use a QUIC
CONNECTION\_CLOSE frame with one of the error codes defined in
{{error-handling}}.

# Error Handling {#error-handling}

The following error codes are defined for use when abruptly terminating RoQ streams,
aborting reading of RoQ streams, or immediately closing RoQ connections.

ROQ\_NO\_ERROR (0x00):
: No error. This is used when the connection or stream needs to be closed, but
there is no error to signal.

ROQ\_GENERAL\_ERROR (0x01):
: An error that does not match a more specific error code occurred.

ROQ\_INTERNAL\_ERROR (0x02):
: An internal error has occurred in the RoQ stack.

ROQ\_PACKET\_ERROR (0x03):
: Invalid payload format, e.g., length does not match packet, invalid flow id
encoding, non-RTP on RTP-flow ID, etc.

ROQ\_STREAM\_CREATION\_ERROR (0x04):
: The endpoint detected that its peer created a stream that violates the ROQ protocol, e.g., a bidirectional stream, for sending RTP packets.

ROQ\_FRAME\_CANCELLED (0x05):
: A receiving endpoint is using STOP_SENDING on the current stream to request
new frames be sent on new streams. Similarly, a sender notifies a receiver that
retransmissions of a frame were stopped using RESET\_STREAM and new frames will
be sent on new streams.

ROQ\_UNKNOWN\_FLOW\_ID (0x06):
: An endpoint was unable to handle a flow identifier, e.g., because it was not
signaled or because the endpoint does not support multiplexing using arbitrary
flow identifiers.

ROQ\_EXPECTATION\_UNMET (0x07):
: RoQ out-of-band signaling set expectations for QUIC transport, but the resulting QUIC connection would not meet those expectations.

# Congestion Control and Rate Adaptation {#congestion-control}

Like any other application on the Internet, RoQ applications need a mechanism to
perform congestion control to avoid overloading the network. QUIC is a
congestion-controlled transport protocol. RTP does not mandate a single
congestion control mechanism. RTP suggests that the RTP profile defines
congestion control according to the expected properties of the application's
environment.

This document discusses aspects of transport level congestion control in
{{cc-quic-layer}} and application layer rate control in
{{rate-adaptation-application-layer}}. It does not mandate any specific
congestion control algorithm for QUIC or rate adaptation algorithm for RTP.

This document also gives guidance about avoiding problems with *nested*
congestion controllers in {{rate-adaptation-application-layer}}.

This document also discusses congestion control implications of using shared or
multiple separate QUIC connections to send and receive multiple independent
RTP streams in {{shared-connections}}.

## Congestion Control at the Transport Layer {#cc-quic-layer}

QUIC is a congestion-controlled transport protocol. Senders are required to
employ some form of congestion control. The default congestion control specified
for QUIC in {{!RFC9002}} is similar to TCP NewReno {{?RFC6582}}, but senders are
free to choose any congestion control algorithm as long as they follow the
guidelines specified in {{Section 3 of ?RFC8085}}, and QUIC implementors make
use of this freedom.

Congestion control mechanisms are often implemented at the transport layer of the protocol stack, but can also be implemented at the application layer.

A congestion control mechanism could respond to actual packet loss (detected by timeouts), or to impending packet loss (signaled by mechanisms such as Explicit Congestion Notification {{?RFC3168}}).

For real-time traffic, it is best that the QUIC implementation uses a congestion
controller that aims at keeping queues at intermediary
network elements, and thus latency, as short as possible. Delay-based congestion control algorithms
might use, for example, an increasing one-way delay as a signal of impending
congestion, and adjust the sending rate to prevent continued increases in
one-way delay.

A wide variety of congestion control algorithms for real-time media have been developed (for example, "Google Congestion Controller" {{?I-D.draft-ietf-rmcat-gcc}}).
The IETF has defined two such algorithms in Experimental RFCs (SCReAM {{?RFC8298}} and NADA {{?RFC8698}}).
These algorithms
for RTP are specifically tailored for real-time transmission at low latencies,
but the guidance in this section would apply to any congestion control algorithm that meets the
requirements described in "Congestion Control Requirements for Interactive
Real-Time Media" {{!RFC8836}}.

Some low latency congestion control algorithms depend on detailed arrival time feedback to estimate the current one-way delay between sender and receiver, which is unavailable in QUIC {{!RFC9000}} without extensions.
QUIC implementations can use an extension to add this information to QUIC as described in {{optional-extensions}}.
In addition to these dedicated real-time media congestion-control algorithms, QUIC implementations could support the Low Latency, Low Loss, and Scalable Throughput (L4S) Internet Service {{?RFC9330}}, which limits growth in round-trip delays that result from increasing queuing delays.
While L4S does not rely on a QUIC protocol extension, L4S does rely on support from network devices along the path from sender to receiver.

The application needs a mechanism to query the available bandwidth to adapt
media codec configurations. If the employed congestion controller of the QUIC
connection keeps an estimate of the available bandwidth, it could also expose an API
to the application to query the current estimate. If the congestion controller
cannot provide a current bandwidth estimate to the application, the sender can
implement an alternative bandwidth estimation at the application layer as
described in {{rate-adaptation-application-layer}}.

It is assumed that the congestion controller in use provides a pacing mechanism
to determine when a packet can be sent to avoid bursts and minimize variation in
inter-packet arrival times. The currently proposed congestion control algorithms
for real-time communications (e.g., SCReAM and NADA) provide such pacing
mechanisms, and the QUIC exemplary congestion control algorithm ({{Section 7.7
of !RFC9002}}) recommends pacing for senders.

## Rate Adaptation at the Application Layer {#rate-adaptation-application-layer}

RTP itself does not specify a congestion control algorithm, but {{!RFC8888}}
defines an RTCP feedback message intended to enable rate adaptation for
interactive real-time traffic using RTP, and successful rate adaptation will
accomplish congestion control as well.

If an application cannot access a bandwidth estimation from the QUIC layer, the
application can alternatively implement a bandwidth estimation algorithm at the
application layer. Congestion control algorithms for real-time media such as GCC
{{?I-D.draft-ietf-rmcat-gcc}}, NADA {{?RFC8698}}, and SCReAM {{?RFC8298}} expose
a target bitrate to dynamically reconfigure media codecs to produce media at
the rate of the observed available bandwidth. Applications can use the same
bandwidth estimation to adapt their rate when using QUIC. However, running an
additional congestion control algorithm at the application layer can have
unintended effects due to the interaction of two *nested* congestion
controllers.

If an RTP application paces its media transmission at a rate that does not saturate path bandwidth,
more heavy-handed congestion control mechanisms (drastic
reductions in the sending rate when loss is detected, with much slower increases
when losses are no longer being detected) ought to rarely come into play. If an RTP
application chooses RoQ as its transport, sends enough media to saturate the available
path bandwidth, and does not adapt its sending rate, these drastic measures will be
required to avoid sustained or oscillating congestion along the path.

Thus, applications are advised to only use the bandwidth estimation without
running the complete congestion control algorithm at the application layer
before passing data to the QUIC layer.

The bandwidth estimation algorithm typically needs some feedback on the
transmission performance. This feedback can be collected via RTCP or following
the guidelines in {{rtcp-mapping}} and {{api-considerations}}.

## Sharing QUIC connections {#shared-connections}

Two endpoints can establish channels to exchange more than one type of
data simultaneously. The channels can be intended to carry real-time RTP data or
other non-real-time data. This can be realized in different ways.

- One straightforward solution is to establish multiple QUIC connections, one
  for each channel, whether the channel is used for real-time media or
  non-real-time data.

- Alternatively, all real-time channels are mapped to one QUIC connection, while
  a separate QUIC connection is created for the non-real-time channels.

- A third option is to multiplex all channels, whether real-time or non-real-time, in a single QUIC connection via an
  extension to RoQ.

In the first two cases, the congestion controllers can be chosen to match the
demands of the respective channels and the different QUIC connections will
compete for the same resources in the network. No local prioritization of data
across the different (types of) channels would be necessary.

Although it is possible to multiplex (all or a subset of) real-time and
non-real-time channels onto a single, shared QUIC connection by extending RoQ,
the underlying QUIC implementation will likely use the same congestion
controller for all channels in the shared QUIC connection. For this reason,
applications multiplexing real-time and non-real-time channels in one connection
will need to implement some form of prioritization or bandwidth allocation for
the different channels.

# Guidance on Choosing QUIC Streams, QUIC DATAGRAMs, or a Mixture {#s-d-m-guidance}

As noted in {{streams-and-datagrams}}, this document does not take a position on using QUIC streams, QUIC DATAGRAMs, or a mixture of both, for any particular RoQ use case or application. It does seem useful to include observations that might guide implementers who will need to make choices about that.

One implementation goal might be to minimize processing overhead, for applications that are migrating from RTP over UDP to RoQ. These applications don't rely on any transport protocol behaviors beyond UDP, which can be described as "IP plus multiplexing". The implementers might be motivated by one or more of the advantages of encapsulating RTP in QUIC that are described in {{motivations}}, but they do not need any of the advantages that would apply when encapsulating RTP in QUIC streams. For these applications, simply placing each RTP packet in a QUIC DATAGRAM frame when it becomes available would be sufficient, using no QUIC streams at all.

Another implementation goal might be to prioritize specific types of video frames over other types. For these applications, placing each type of video frame in a separate QUIC stream would allow the RoQ receiver to focus on the most important video frames more easily. This also allows the implementer to rely on QUIC's "byte stream" abstraction, freeing the application from dealing with MTU size restrictions, in contrast to the need to fit RTP packets into QUIC DATAGRAMs. The application might use QUIC streams for all of the RTP packets carried over this specific QUIC connection, with no QUIC DATAGRAMs at all.

Some applications might have implementation goals that don't fit neatly into "QUIC streams only" or "QUIC DATAGRAMs only" categories. For example, another implementation goal might be to use QUIC streams to carry RTP video frames, but to use QUIC DATAGRAMs to carry RTP audio frames, which are typically much smaller. Because humans tend to tolerate inconsistent behavior in video better than inconsistent behavior in audio, the application might add Forward Error Correction {{!RFC6363}} to RTP audio packets and encapsulate the result in QUIC DATAGRAMs, while encapsulating RTP video packets in QUIC streams.

As noted in {{multiplexing}}, all RoQ streams and RoQ datagrams begin with a flow identifier. This allows a RoQ sender to begin by encapsulating related RTP packets in QUIC streams and then switch to carrying them in QUIC DATAGRAMs, or vice versa. RoQ receivers need to be prepared to accept any valid RTP packet with a given flow identifier, whether it started by being encapsulated in QUIC streams or in QUIC DATAGRAMs, and RoQ receivers need to be prepared to accept RTP flows that switch from QUIC stream encapsulation to QUIC DATAGRAMs, or vice versa.

Because QUIC provides a capability to migrate connections for various reasons, including recovering from a path failure ({{Section 9 of !RFC9000}}), when a QUIC connection migrates, a RoQ sender has the opportunity to revisit decisions about which RTP packets are encapsulated in QUIC streams, and which RTP packets are encapsulated in QUIC DATAGRAMs. Again, RoQ receivers need to be prepared for this eventuality.

# Replacing RTCP and RTP Header Extensions with QUIC Feedback {#rtcp-mapping}

Because RTP has so often used UDP as its underlying transport protocol,
receiving little or no transport feedback, existing RTP implementations rely on feedback
from the RTP Control Protocol (RTCP) so that RTP senders and receivers can
exchange control information to monitor connection statistics and to identify
and synchronize media streams.

Because QUIC can provide transport-level feedback, it can replace at least some RTP
transport-level feedback with current QUIC feedback {{!RFC9000}}. In addition,
RTP-level feedback that is not available in QUIC by default can potentially be
replaced with feedback provided by useful QUIC extensions in the future as described in
{{rtcp-quic-ext-examples}}.

When statistics contained in RTCP packets are also available from QUIC or can be
derived from statistics available from QUIC, it is desirable to provide these
statistics at only one protocol layer. This avoids consumption of bandwidth to
deliver equivalent control information at more than one level of the protocol
stack. QUIC and RTCP both have rules describing when certain signals are to be
sent. This document does not change any of the rules described by either
protocol. Rather, it specifies a baseline for replacing some of the RTCP packet types
by mapping the contents to QUIC connection statistics, and reducing the
transmission frequency and bandwidth requirements for some RTCP packet types
that must be transmitted periodically. Future documents can extend this mapping
for other RTCP format types and can make use of new QUIC extensions that become
available over time. The mechanisms described in this section can enhance the
statistics provided by RTCP and reduce the bandwidth overhead required by
certain RTCP packets. Applications using RoQ still need to adhere to the rules for
RTCP feedback given by {{!RFC3550}} and the RTP profiles in use.

Most statements about "QUIC" in {{rtcp-mapping}} are applicable to both RTP
packets encapsulated in QUIC streams and RTP packets encapsulated in DATAGRAMs.
The differences are described in {{roc-d}} and {{roc-s}}.

While RoQ places no restrictions on applications sending RTCP, this document assumes that the reason an implementer chooses to support RoQ is to obtain benefits beyond what's available when RTP uses UDP as its underlying transport layer.
Exposing relevant information from the QUIC layer to the application instead of exchanging additional RTCP packets, where applicable, will reduce processing and bandwidth overhead for RoQ senders and receivers.

{{transport-layer-feedback}} discusses what information can be exposed from the
QUIC connection layer to reduce the RTCP overhead.

## RoQ Datagrams {#roc-d}

QUIC DATAGRAMs are ACK-eliciting packets, which means that an acknowledgment is
triggered when a DATAGRAM frame is received. Thus, a sender can assume that an
RTP packet arrived at the receiver or was lost in transit, using the QUIC
acknowledgments of QUIC DATAGRAM frames. In the following, an RTP packet is
regarded as acknowledged when the QUIC DATAGRAM frame that carried the RTP
packet was acknowledged.

## RoQ Streams {#roc-s}

For RTP packets that are sent over QUIC streams, an RTP packet is considered
acknowledged after all STREAM frames that carried parts of the RTP packet were
acknowledged.

When QUIC streams are used, the implementer needs to be aware that the direct
mapping proposed below might not reflect the real characteristics of the network.
RTP packet loss can seem lower than actual packet loss due to QUIC's automatic
retransmissions. Similarly, timing information can be incorrect due to
retransmissions or transmission delays introduced by the QUIC stack.

## Multihop Topologies {#multi-hop}

In some topologies, RoQ might only be used on some of the links between multiple
session participants. Other links might be using RTP over UDP, or over some other
supported RTP encapsulation protocol, and some participants might be using RTP
implementations that don't support RoQ at all. These participants will not be
able to infer feedback from QUIC, and they might receive less RTCP feedback than
expected. This situation can arise when participants using RoQ are not aware that
other participants are not using RoQ and minimize their use of RTCP,
assuming their RoQ peer will be able to infer statistics from QUIC. There are
two ways to solve this problem:

* If the middlebox translating between RoQ and RTP
over other RTP transport protocols such as UDP or TCP provides Back-to-Back RTP
sessions as described in {{Section 3.2.2 of !RFC7667}}, this middlebox can add
RTCP packets for the participants not using RoQ by using the statistics the
middlebox gets from QUIC and the mappings described in the following sections.
* If the middlebox does not provide Back-to-Back RTP sessions, participants can
use additional signaling to let the RoQ participants know what RTCP is
required.

## Feedback Mappings {#transport-layer-feedback}

This section explains how some of the RTCP packet types that are used to signal
reception statistics can be replaced by equivalent statistics that are already
collected by QUIC. The following list explains how this mapping can be achieved
for the individual fields of different RTCP packet types.

The list of RTCP packets in this section is not exhaustive, and similar considerations would apply when exchanging any other type of RTCP control packets using RoQ.

A more thorough analysis including the information that cannot be mapped from
QUIC can be found in {{rtcp-analysis}}: RTCP Control Packet Types (in
{{control-packets}}), Generic RTP Feedback (RTPFB) (in {{generic-feedback}}),
Payload-specific RTP Feedback (PSFB) (in {{payload-specific-feedback}}),
Extended Reports (in {{extended-reports}}), and RTP Header Extensions (in
{{rtp-header-extensions}}).

### Negative Acknowledgments ("NACK") {#NACK-mappings}

Generic *Negative Acknowledgments* (`PT=205`, `FMT=1`, `Name=Generic NACK`,
{{!RFC4585}}) contain information about RTP packets which the receiver
considered lost. {{Section 6.2.1. of !RFC4585}} recommends using this feature
only if the underlying protocol cannot provide similar feedback. QUIC does not
provide negative acknowledgments but can detect lost packets based on the Gap
numbers contained in QUIC ACK frames ({{Section 6 of !RFC9002}}).

### ECN Feedback ("ECN") {#ECN-mappings}

*ECN Feedback* (`PT=205`, `FMT=8`, `Name=RTCP-ECN-FB`, {{!RFC6679}}) packets report the count of observed ECN-CE marks.
{{!RFC6679}} defines two RTCP reports, one packet type (with `PT=205` and `FMT=8`), and a new report block for the extended reports.
QUIC supports ECN reporting through acknowledgments.
If the QUIC connection supports ECN, using QUIC acknowledgments to report ECN counts, rather than RTCP ECN feedback reports, reduces bandwidth and processing demands on the RTCP implementation.

### Goodbye Packets ("BYE") {#BYE-mapping}

RTP session participants can use *Goodbye* RTCP packets (`PT=203`, `Name=BYE`,
{{!RFC3550}}), to indicate that a source is no longer active. If the participant
is also going to close the QUIC connection, the *BYE* packet can be replaced by
a QUIC CONNECTION_CLOSE frame. In this case, the reason for leaving can be
transmitted in QUIC's CONNECTION_CLOSE *Reason Phrase*. However, if the
participant wishes to use this QUIC connection for any other multiplexed
traffic, the participant has to use the BYE packet because the QUIC
CONNECTION_CLOSE would close the entire QUIC connection for all other QUIC
streams and DATAGRAMs.

# RoQ-QUIC and RoQ-RTP API Considerations {#api-considerations}

The mapping described in the previous sections relies on the QUIC implementation passing some information to the RoQ implementation.
Although RoQ will function without this information, some
optimizations regarding rate adaptation and RTCP mapping require certain
functionalities to be exposed to the application.

Each item in the following list can be considered individually. Any exposed
information or function can be used by RoQ regardless of whether the other items
are available. Thus, RoQ does not depend on the availability of all of the
listed features but can apply different optimizations depending on the
functionality exposed by the QUIC implementation.

* *initial_max_data transport*: If the QUIC receiver has indicated a willingness to accept
  0-RTT packets with early data, this is the maximum size that the QUIC sender can use,
  as described in {{ed-considerations}}.
* *Maximum Datagram Size*: The maximum DATAGRAM size that the QUIC connection
  can transmit on the network path to the QUIC receiver. If a RoQ sender using
  DATAGRAMs does not know the maximum DATAGRAM size for the path to the RoQ
  receiver, there are only two choices - either use heuristics to limit the size
  of RoQ messages, or be prepared to lose RoQ messages that were too large to be
  carried through the network path and delivered to the RoQ receiver.
* *Datagram Acknowledgment and Loss*: {{Section 5.2 of !RFC9221}} allows QUIC
  implementations to notify the application that a DATAGRAM was
  acknowledged or that it believes a DATAGRAM was lost. Given the DATAGRAM
  acknowledgments and losses, the application can deduce which RTP packets
  arrived at the receiver and which were lost (see also {{roc-d}}).
* *Stream States*: The stream states include which parts of the data sent on a
  stream were successfully delivered and which are still outstanding to be sent
  or retransmitted. If an application keeps track of the RTP packets sent on a
  stream, their respective sizes, and in which order they were transmitted, it
  can infer which RTP packets were acknowledged according to the definition in
  {{roc-s}}.
* *Arrival timestamps*: If the QUIC connection uses a timestamp extension like
  {{?I-D.draft-smith-quic-receive-ts}} or {{?I-D.draft-huitema-quic-ts}}, the
  arrival timestamps or one-way delays can support the application as described
  in {{rtcp-mapping}} and {{congestion-control}}.
* *Bandwidth Estimation*: If a bandwidth estimate is available in the QUIC
  implementation, exposing it avoids the overhead of executing an additional
  bandwidth estimation algorithm in the application.
* *ECN*: If ECN marks are available, they can support the bandwidth estimation
  of the application.
* *RTT*: The RTT estimations as described in {{Section 5 of !RFC9002}}.

One goal for the RoQ protocol is to shield RTP applications from the details of QUIC encapsulation, so the RTP application doesn't need much information about QUIC from RoQ, but some information will be valuable. For example, it could be desirable that the RoQ implementation provides an indication of connection migration to the RTP application.

# Discussion

This section contains topics that are worth mentioning, but don't fit well into other sections of the document.

## Impact of Connection Migration

RTP sessions are characterized by a continuous flow of packets in either or
both directions.  A connection migration might lead to pausing media
transmission until reachability of the peer under the new address is validated.
This might lead to short breaks in media delivery in the order of RTT and, if
RTCP is used for RTT measurements, might cause spikes in observed delays.
Application layer congestion control mechanisms (and packet repair schemes
such as retransmissions) need to be prepared to cope with such spikes. As noted in {{api-considerations}}, it could be desirable that the RoQ implementation provides an indication of connection migration to the RTP application, to assist in coping.

## 0-RTT and Early Data considerations {#ed-considerations}

RoQ applications, like any other RTP applications, want to establish a media path quickly, reducing clipping at the beginning of a conversation. For repeated connections between endpoints that have previously carried out a full TLS handshake procedure, the initiator of a QUIC connection can
use 0-RTT packets with "early data" to include application data in a packet that is used to establish a connection.

As 0-RTT packets are subject to replay attacks, RoQ applications MUST carefully specify which data types and operations
are allowed.

{{Section 9.2 of !RFC9001}} says

> Application protocols MUST either prohibit the use of extensions that carry application semantics in 0-RTT or provide replay mitigation strategies.

For the purposes of this discussion, RoQ is an application protocol that allows the use of 0-RTT.

RoQ application developers ought to take the considerations described in {{rej-ed}} and {{replay-ed}} into account when deciding whether to use 0-RTT with early data for an application.

### Effect of 0-RTT Rejection for RoQ using Early Data {#rej-ed}

If the goal for using early data is to reduce clipping, a QUIC endpoint is relying on the other QUIC endpoint to accept the 0-RTT packet carrying early data containing media.

A QUIC endpoint indicates its willingness to accept a 0-RTT packet containing early data by sending the TLS early_data extension in the NewSessionTicket message with the max_early_data_size parameter set to the sentinel value 0xffffffff.
The amount of data that a QUIC client can send in QUIC 0-RTT is controlled by the initial_max_data transport parameter supplied by the QUIC server.
This is described in more detail in {{Section 4.6.1 of !RFC9001}}.

If a QUIC endpoint is not willing to accept a 0-RTT packet containing early data, but receives one anyway, the QUIC endpoint rejects the 0-RTT packet by sending EncryptedExtensions without an early_data extension. This is described in more detail, in {{Section 4.6.2 of !RFC9001}}.
This necessarily means that a QUIC endpoint attempting to convey RoQ media is now subject to at least one additional RTT delay, as it must send a QUIC Initial packet and perform a full QUIC handshake before it can send RoQ media.

### Effect of 0-RTT Replay Attacks for RoQ using Early Data {#replay-ed}

Including "early data" in the packet payload in any QUIC 0-RTT packet exposes the application to an additional risk, of accepting "early data" from a 0-RTT packet that has been replayed.

While it is true that

- RTP typically carries ephemeral media contents that is rendered and possibly recorded but otherwise causes no side effects,

- the amount of data that can be carried as packet payload in a 0-RTT packet is rather limited, and

- RTP implementations are likely to discard any replayed media packets as duplicates,

it is still the responsibility of the RoQ application to determine whether the benefits of using 0-RTT with early data outweigh the risks.

Since the QUIC connection will often be created in the context
of an existing signaling relationship (e.g., using WebRTC or SIP), a careful RoQ implementer can exchange specific 0-RTT
keying material to prevent replays across sessions.

## Coalescing RTP packets in a single QUIC packet {#coalescing-packets}

Applications have some control over how the QUIC stack maps application data to
QUIC frames, but applications cannot control how the QUIC stack maps STREAM and
DATAGRAM frames to QUIC packets {{Section 13 of ?RFC9000}} and {{Section 5 of
?RFC9308}}.

* When RTP payloads are carried over QUIC streams, the RTP payload is treated as
  an ordered byte stream that will be carried in QUIC STREAM frames, with no
  effort to match application data boundaries.
* When RTP payloads are carried over DATAGRAMs, each RTP payload data unit
  is mapped into a DATAGRAM frame, but
* QUIC implementations can include multiple STREAM frames from different streams
  and one or more DATAGRAM frames into a single QUIC packet, and can include
  other QUIC frames as well.

QUIC stacks are allowed to wait for a short period of time if the queued QUIC
packet is shorter than the Path MTU, in order to optimize for bandwidth
utilization instead of latency, while real-time applications usually prefer to
optimize for latency rather than bandwidth utilization. This waiting interval is
under the QUIC implementation's control, and could be based on knowledge about
application sending behavior or heuristics to determine whether and for how long
to wait.

When there are a lot of small DATAGRAM frames (e.g., an audio stream) and a lot
of large DATAGRAM frames (e.g., a video stream), it could be a good idea to make sure the
audio frames can be included in a QUIC packet that also carries video frames
(i.e., the video frames don't fill the whole QUIC packet). Otherwise, the QUIC
stack might have to send additional small packets only carrying single audio
frames, which would waste some bandwidth.

Application designers are advised to take these considerations into account when
selecting and configuring a QUIC stack for use with RoQ.

# Directions for Future Work {#futures}

This document describes RoQ in sufficient detail that an implementer can build a RoQ application, but we recognize that additional work is likely, after we have sufficient experience with RoQ to guide that work ({{futures-impl-deploy}}) and as new QUIC extensions become available ({{futures-new-ext}}).

## Future Work Resulting from Implementation and Deployment Experience {#futures-impl-deploy}

Possible directions would include

* More guidance on transport for RTCP (for example, when to use QUIC streams vs. DATAGRAMs) including guidance on prioritization between streams and DATAGRAMs for the performance of RTCP.

* More guidance on the use of real-time-friendly congestion control algorithms (for example, Copa {{Copa}}, L4S {{?RFC9330}}, etc.).

* More guidance for congestion control and rate adaptation for multiple RoQ flows (whether streams or datagrams).

* Possible guidance for connection sharing between real-time and non-real-time flows, including considerations for congestion control and rate adaptation, scheduling, prioritization, and which ALPNs to use.

* Investigation of the effects of delaying or dropping DATAGRAMs due to congestion before they can be transmitted by the QUIC stack.

* Implementation of translating middleboxes for translating between RoQ and RTP over UDP. As described in {{topologies}}, RoQ can be used to connect to some RTP middleboxes using some topologies, and these middleboxes might be connecting RoQ endpoints and non-RoQ endpoints, so will need to translate between RoQ and RTP over UDP.

For these reasons, publication of this document as a stable reference for implementers to test with, and report results, seems useful.

## Future Work Resulting from New QUIC Extensions {#futures-new-ext}

In addition, as noted in {{new-quic}}, one of the motivations for using QUIC as a transport for RTP is to exploit new QUIC extensions as they become available. We noted several specific proposed QUIC extensions in {{optional-extensions}}, but these proposals are all solving relevant problems, and those problems are worth solving for the QUIC protocol, whether the listed proposals are used in the solution or not.

* Guidance for using RoQ with QUIC connection migration and over multiple paths. QUIC connection migration was already defined in {{!RFC9000}}, and the Multipath Extension for QUIC {{?I-D.draft-ietf-quic-multipath}} has been adopted and is relatively mature, so this is likely to be the first new QUIC extension we address.

* Guidance for using RoQ with QUIC NAT traversal solutions. This could use Interactive Connectivity Establishment (ICE) {{?RFC8445}} or other NAT traversal solutions.

* Guidance for improved jitter calculations to use with congestion control and rate adaptation.

* Guidance for other aspects of QUIC performance optimization relying on extensions.

Other QUIC extensions, not yet proposed, might also be useful with RoQ.

# Implementation Status

> **RFC Editor's note:** Please remove this section prior to publication of a
> final version of this document.

This section records the status of known implementations of the protocol defined
by this specification at the time of posting of this Internet-Draft, and is
based on a proposal described in {{?RFC7942}}. The description of
implementations in this section is intended to assist the IETF in its decision
processes in progressing drafts to RFCs. Please note that the listing of any
individual implementation here does not imply endorsement by the IETF.
Furthermore, no effort has been spent to verify the information presented here
that was supplied by IETF contributors. This is not intended as, and must not be
construed to be, a catalog of available implementations or their features.
Readers are advised to note that other implementations may exist.

According to {{?RFC7942}}, "this will allow reviewers and working groups to
assign due consideration to documents that have the benefit of running code,
which may serve as evidence of valuable experimentation and feedback that have
made the implemented protocols more mature. It is up to the individual working
groups to use this information as they see fit".

## mengelbart/roq

Ogranization:
: Technical University of Munich

Implementation:
: {{roq}}

Description:
: *roq* is a library implementing the basic encapsulation described in
{{encapsulation}}. The library uses the Go programming language and supports the
{{quic-go}} QUIC implementation.

Level of Maturity:
: prototype

Coverage: : The library supports sending and receiving RTP and RTCP packets
using QUIC streams and QUIC DATAGRAMs, and supports multiplexing using flow
identifiers. Applications using the library are responsible for appropriate
signaling, setting up QUIC connections, and managing RTP sessions. Applications
choose whether to send RTP and RTCP packets over streams or DATAGRAMs, and
applications also have control over the QUIC and RTP congestion controllers in
use since they control the QUIC connection setup and can thus configure the QUIC
stack they use to their preferences.

Version Compatibility:
: The library implements {{?I-D.draft-ietf-avtcore-rtp-over-quic-10}}.

Licensing:
: MIT License

Implementation Experience:
: The implementer reports they have no experience with the topics discussed in
{{futures}}. RoQ relies on out-of-band signaling for connection establishment,
and since there is currently no specification for SDP for RoQ, applications
using the library have to statically configure connection information to allow
testing

Contact Information:
: Mathis Engelbart (mathis.engelbart@gmail.com)

Last Updated:
: 25 May 2024

## bbc/gst-roq

Ogranization:
: BBC Research and Development

Implementation:
: RTP-over-QUIC elements for GStreamer {{gst-roq}}

Description:
: *gst-quic-transport* provides a set of GStreamer plugins implementing QUIC
transport. *gst-roq* provides a set of GStreamer plugins implementing RoQ.

Level of Maturity:
: research

Coverage:
: The plugins support sending and receiving RTP and RTCP packets using QUIC
streams and QUIC DATAGRAMs, and supports multiplexing using flow identifiers.
GStreamer pipelines that use the RoQ plugins found in the *gst-roq* repository
can make use of the plugins found in the *gst-quic-transport* repository to set
up QUIC connections. RTP sessions can be managed by existing GStreamer plugins
available in the standard GStreamer release. GStreamer pipeline applications
choose whether to send RTP and RTCP packets over streams or DATAGRAMs.

Version Compatibility:
: The library implements {{?I-D.draft-ietf-avtcore-rtp-over-quic-05}}.

Licensing:
: GNU Lesser General Public License v2.1

Implementation Experience:
: The implementer reports they have no experience with the topics discussed in
{{futures}}. Both in-band and out-of-band signalling for RoQ media sessions is
in active development via an implementation of {{?I-D.draft-hurst-sip-quic-00}},
which re-uses the GStreamer plugins described above.

Contact Information:
: Sam Hurst (sam.hurst@bbc.co.uk)

Last Updated:
: 05 June 2024

## mengelbart/rtp-over-quic

Ogranization:
: Technical University of Munich

Implementation:
: RTP over QUIC {{RTP-over-QUIC}}

Description:
: *RTP over QUIC* is a experimental implementation of the mapping described in
an earlier version of this document.

Level of Maturity:
: research

Coverage:
: The application implements the RoQ DATAGRAMs mapping and implements SCReAM
congestion control at the application layer. It can optionally disable the
built-in QUIC congestion control (NewReno). The endpoints only use RTCP for
congestion control feedback, which can optionally be disabled and replaced by
the QUIC connection statistics as described in {{transport-layer-feedback}}.
Experimental results of the implementation can be found in {{RoQ-Mininet}}.

Version Compatibility:
: {{?I-D.draft-ietf-avtcore-rtp-over-quic-00}}

Licensing:
: MIT

Implementation Experience:
: See {{RoQ-Mininet}}

Contact Information:
: Mathis Engelbart (mathis.engelbart@gmail.com)

Last Updated:
: 25 May 2024

# Security Considerations {#sec-considerations}

RoQ is subject to the security considerations of RTP described in
{{Section 9 of !RFC3550}} and the security considerations of any RTP profile in
use.

The security considerations for the QUIC protocol and DATAGRAM extension
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

This document registers a new ALPN protocol ID (in {{iana-alpn}}) and creates a
new registry that manages the assignment of error code points in RoQ (in
{{iana-error-codes}}).

## Registration of a RoQ Identification String {#iana-alpn}

This document creates a new registration for the identification of RoQ
in the "TLS Application-Layer Protocol Negotiation (ALPN) Protocol IDs" registry
{{?RFC7301}}.

The "roq" string identifies RoQ:

  Protocol:
  : RTP over QUIC (RoQ)

  Identification Sequence:
  : 0x72 0x6F 0x71 ("roq")

  Specification:
  : This document

## RoQ Error Codes Registry {#iana-error-codes}

This document establishes a registry for RoQ error codes. The "RTP over QUIC
(RoQ) Error Codes" registry manages a 62-bit space and is listed under the
"Real-Time Transport Protocol (RTP) Parameters" heading.

The new error codes registry created in this document operates under the QUIC
registration policy documented in {{Section 22.1 of !RFC9000}}. This registry
includes the common set of fields listed in {{Section 22.1.1 of !RFC9000}}.

Permanent registrations in this registry are assigned using the Specification
Required policy ({{!RFC8126}}), except for values between 0x00 and 0x3f (in
hexadecimal; inclusive), which are assigned using Standards Action or IESG
Approval as defined in {{Sections 4.9 and 4.10 of !RFC8126}}.

Registrations for error codes are required to include a description of the error
code. An expert reviewer is advised to examine new registrations for possible
duplication or interaction with existing error codes.

In addition to common fields as described in Section {{Section 22.1 of
!RFC9000}}, this registry includes two additional fields. Permanent
registrations in this registry MUST include the following fields:

Name:
: A name for the error code.

Description:
: A brief description of the error code semantics, which can be a summary if a
specification reference is provided.

The initial allocations in this registry are all assigned permanent status and
list a change controller of the IETF and a contact of the AVTCORE working group
(avt@ietf.org).

The entries in {{tab-error-codes}} are registered by this document.

| Value  | Name                          | Description                            | Specification      |
| ------ | ----------------------------- | -------------------------------------- | ------------------ |
| 0x00   | ROQ\_NO\_ERROR                | No Error                               | {{error-handling}} |
| 0x01   | ROQ\_GENERAL\_ERROR           | General error                          | {{error-handling}} |
| 0x02   | ROQ\_INTERNAL\_ERROR          | Internal Error                         | {{error-handling}} |
| 0x03   | ROQ\_PACKET\_ERROR            | Invalid payload format                 | {{error-handling}} |
| 0x04   | ROQ\_STREAM\_CREATION\_ERROR  | Invalid stream type                    | {{error-handling}} |
| 0x05   | ROQ\_FRAME\_CANCELLED         | Frame cancelled                        | {{error-handling}} |
| 0x06   | ROQ\_UNKNOWN\_FLOW\_ID        | Unknown Flow ID                        | {{error-handling}} |
| 0x07   | ROQ\_EXPECTATION\_UNMET       | Externally signaled requirement unmet | {{error-handling}} |
{: #tab-error-codes title="Initial RoQ Error Codes"}

--- back

# List of optional QUIC Extensions {#optional-extensions}

The following is a list of QUIC protocol extensions that could be beneficial for
RoQ, but are not required by RoQ.

* *An Unreliable Datagram Extension to QUIC* {{?RFC9221}}. Without support for
  unreliable DATAGRAMs, RoQ cannot use the encapsulation specified in
  {{quic-datagrams}}, but can still use QUIC streams as specified in
  {{quic-streams}}.
* A version of QUIC receive timestamps can be helpful for improved jitter
  calculations and congestion control. If the QUIC connection uses a timestamp
  extension such as *Quic Timestamps For Measuring One-Way Delays* {{?I-D.draft-huitema-quic-ts}} or
  *QUIC Extension for Reporting Packet Receive Timestamps* {{?I-D.draft-smith-quic-receive-ts}},
  the arrival timestamps or one-way delays could be exposed to
  the application for improved bandwidth estimation or RTCP mappings as
  described in {{rtcp-mapping}} and {{rtcp-analysis}}.
* *QUIC Acknowledgment Frequency* {{?I-D.draft-ietf-quic-ack-frequency}} can
  be used by a sender to optimize the acknowledgment behavior of the receiver,
  e.g., to optimize congestion control.
* *Signaling That a QUIC Receiver Has Enough Stream Data*
  {{?I-D.draft-thomson-quic-enough}} and *Reliable QUIC Stream Resets*
  {{?I-D.draft-ietf-quic-reliable-stream-reset}} would allow RoQ senders and
  receivers to use versions of CLOSE\_STREAM and STOP\_SENDING that contain
  offsets. The offset could be used to reliably retransmit all frames up to a
  certain frame that ought to be cancelled before resuming transmission of further
  frames on new QUIC streams.

# Considered RTCP Packet Types and RTP Header Extensions {#rtcp-analysis}

This section lists all the RTCP packet types and RTP header extensions that were
considered in the analysis described in {{rtcp-mapping}}.

Each subsection in {{rtcp-analysis}} corresponds to an IANA registry, and includes a reference pointing to that registry.

Several but not all of these control packets and their attributes can be mapped
from QUIC, as described in {{transport-layer-feedback}}. *Mappable from QUIC*
has one of four values: *yes*, *partly*, *QUIC extension needed*, and *no*.
*Partly* is used for packet types for which some fields can be mapped from QUIC,
but not all. *QUIC extension needed* describes packet types which could be
mapped with help from one or more QUIC extensions.

Examples of how certain packet types could be mapped with the help of QUIC
extensions follow in {{rtcp-quic-ext-examples}}.

## RTCP Control Packet Types {#control-packets}

The IANA registry for this section is {{IANA-RTCP-PT}}.

| Name | Shortcut | PT | Defining Document | Mappable from QUIC | Comments |
| ---- | -------- | -- | ----------------- | ---------------- | -------- |
| SMPTE time-code mapping | SMPTETC | 194 | {{?RFC5484}}  | no | |
| Extended inter-arrival jitter report | IJ | 195 | {{?RFC5450}} | no | Would require send-timestamps, which are not provided by any QUIC extension today |
| Sender Reports | SR | 200 | {{?RFC3550}}  | QUIC extension needed / partly | see {{al-repair}} and {{RR-mappings}} |
| Receiver Reports | RR | 201 | {{?RFC3550}} | QUIC extension needed | see {{RR-mappings}} |
| Source description | SDES | 202 | {{?RFC3550}}  | no | |
| Goodbye | BYE | 203 | {{?RFC3550}}  | partly | see {{BYE-mapping}} |
| Application-defined | APP | 204 | {{?RFC3550}}  | no | |
| Generic RTP Feedback | RTPFB | 205 | {{?RFC4585}} | partly | see {{generic-feedback}} |
| Payload-specific | PSFB | 206 | {{?RFC4585}}  | partly | see {{payload-specific-feedback}} |
| extended report | XR | 207 | {{?RFC3611}} | partly | see {{extended-reports}} |
| AVB RTCP packet | AVB | 208 | {{IEEE-1733-2011}} | no | |
| Receiver Summary Information | RSI | 209 | {{?RFC5760}} | no | |
| Port Mapping | TOKEN | 210 | {{?RFC6284}}  | no | |
| IDMS Settings | IDMS | 211 | {{?RFC7272}}  | no | |
| Reporting Group Reporting Sources | RGRS | 212 | {{?RFC8861}} | no | |
| Splicing Notification Message | SNM | 213 | {{?RFC8286}} | no | |

## RTCP XR Block Type {#extended-reports}

The IANA registry for this section is {{IANA-RTCP-XR-BT}}.

| Name | Document | Mappable from QUIC  | Comments |
| ---- | -------- | ---------------- | -------- |
| Loss RLE Report Block | {{?RFC3611}} | yes | If only used for acknowledgment, could be replaced by QUIC acknowledgments, see {{roc-d}} and {{roc-s}} |
| Duplicate RLE Report Block | {{?RFC3611}}  | no | |
| Packet Receipt Times Report Block | {{?RFC3611}}  | QUIC extension needed / partly | QUIC could provide packet receive timestamps when using a timestamp extension that reports timestamp for every received packet, such as {{?I-D.draft-smith-quic-receive-ts}}. However, QUIC does not provide feedback in RTP timestamp format. |
| Receiver Reference Time Report Block | {{?RFC3611}}  | QUIC extension needed | Used together with DLRR Report Blocks to calculate RTTs of non-senders. RTT measurements can natively be provided by QUIC. |
| DLRR Report Block | {{?RFC3611}}  | QUIC extension needed | Used together with Receiver Reference Time Report Blocks to calculate RTTs of non-senders. RTT can natively be provided by QUIC. |
| Statistics Summary Report Block | {{?RFC3611}}  | QUIC extension needed / partly | Packet loss and jitter can be inferred from QUIC acknowledgments, if a timestamp extension is used (see {{?I-D.draft-smith-quic-receive-ts}} or {{?I-D.draft-huitema-quic-ts}}). The remaining fields cannot be mapped to QUIC. |
| VoIP Metrics Report Block | {{?RFC3611}}  | no | as in other reports above, only loss and RTT available |
| RTCP XR | {{?RFC5093}} | no | |
| Texas Instruments Extended VoIP Quality Block | | | |
| Post-repair Loss RLE Report Block | {{?RFC5725}} | no | |
| Multicast Acquisition Report Block | {{?RFC6332}}  | no | |
| IDMS Report Block | {{?RFC7272}} | no | |
| ECN Summary Report | {{?RFC6679}} | partly | see {{ECN-mappings}} |
| Measurement Information Block | {{?RFC6776}} | no | |
| Packet Delay Variation Metrics Block | {{?RFC6798}} | no | QUIC timestamps can be used to achieve the same goal |
| Delay Metrics Block | {{?RFC6843}} | no | QUIC has RTT and can provide timestamps for one-way delay, but QUIC timestamps cannot provide end-to-end statistics when QUIC is only used on one segment of the path. |
| Burst/Gap Loss Summary Statistics Block | {{?RFC7004}} | no | |
| Burst/Gap Discard Summary Statistics Block | {{?RFC7004}}  | no | |
| Frame Impairment Statistics Summary | {{?RFC7004}}   | no | |
| Burst/Gap Loss Metrics Block | {{?RFC6958}} | | no |
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
{: #tab-xr-blocks title="Extended Report Blocks"}

## FMT Values for RTP Feedback (RTPFB) Payload Types {#generic-feedback}

The IANA registry for this section is {{IANA-RTCP-FMT-RTPFB-PT}}.

| Name     | Long Name | Document | Mappable from QUIC  | Comments |
| -------- | --------- | -------- | ---------------- | -------- |
| Generic NACK | Generic negative acknowledgement | {{?RFC4585}}   | partly | see {{NACK-mappings}} |
| TMMBR | Temporary Maximum Media Stream Bit Rate Request | {{?RFC5104}}  | no | |
| TMMBN | Temporary Maximum Media Stream Bit Rate Notification | {{?RFC5104}}   | no | |
| RTCP-SR-REQ | RTCP Rapid Resynchronisation Request | {{?RFC6051}}  | no | |
| RAMS | Rapid Acquisition of Multicast Sessions | {{?RFC6285}} | no | |
| TLLEI | Transport-Layer Third-Party Loss Early Indication | {{?RFC6642}}  | no | There is no way to tell a QUIC implementation "don't ask for retransmission". |
| RTCP-ECN-FB | RTCP ECN Feedback | {{?RFC6679}} | partly | see {{ECN-mappings}} |
| PAUSE-RESUME | Media Pause/Resume | {{?RFC7728}} | no | |
| DBI | Delay Budget Information (DBI) | {{3GPP-TS-26.114}} | |
| CCFB | RTP Congestion Control Feedback | {{?RFC8888}} | QUIC extension needed | see {{CCFB-mappings}} |

## FMT Values for Payload-Specific Feedback (PSFB) Payload Types {#payload-specific-feedback}

The IANA registry for this section is {{IANA-RTCP-FMT-PSFB-PT}}.

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
| VP | Viewport (VP) | {{3GPP-TS-26.114}} |
| AFB | Application Layer Feedback | {{?RFC4585}} |
| TSRR | Temporal-Spatial Resolution Request | {{?I-D.draft-ietf-avtcore-rtcp-green-metadata}} |
| TSRN | Temporal-Spatial Resolution Notification | {{?I-D.draft-ietf-avtcore-rtcp-green-metadata}} |

## RTP Header extensions {#rtp-header-extensions}

Like the payload-specific feedback packets, QUIC cannot directly replace the
control information in the following header extensions. RoQ does not place
restrictions on sending any RTP header extensions. However, some extensions,
such as Transmission Time offsets {{?RFC5450}} are used to improve network
jitter calculation, which can be done in QUIC if a timestamp extension is used.

### RTP Compact Header Extensions

The IANA registry for this section is {{IANA-RTP-CHE}}.

| Extension URI | Description | Reference | Mappable from QUIC |
| ------------- | ----------- | --------- | ---- |
| urn:ietf:params:rtp-hdrext:toffset | Transmission Time offsets | {{?RFC5450}} | no |
| urn:ietf:params:rtp-hdrext:ssrc-audio-level | Audio Level | {{?RFC6464}} | no |
| urn:ietf:params:rtp-hdrext:splicing-interval | Splicing Interval | {{?RFC8286}} | no |
| urn:ietf:params:rtp-hdrext:smpte-tc | SMPTE time-code mapping | {{?RFC5484}} | no |
| urn:ietf:params:rtp-hdrext:sdes | Reserved as base URN for RTCP SDES items that are also defined as RTP compact header extensions. | {{?RFC7941}} | no |
| urn:ietf:params:rtp-hdrext:ntp-64 | Synchronisation metadata: 64-bit timestamp format | {{?RFC6051}} | no |
| urn:ietf:params:rtp-hdrext:ntp-56 | Synchronisation metadata: 56-bit timestamp format | {{?RFC6051}} | no |
| urn:ietf:params:rtp-hdrext:encrypt | Encrypted extension header element | {{?RFC6904}} | no |
| urn:ietf:params:rtp-hdrext:csrc-audio-level | Mixer-to-client audio level indicators | {{?RFC6465}} | no |
| urn:3gpp:video-orientation:6 | Higher granularity (6-bit) coordination of video orientation (CVO) feature, see clause 6.2.3 | {{3GPP-TS-26.114}} | probably not(?) |
| urn:3gpp:video-orientation | Coordination of video orientation (CVO) feature, see clause 6.2.3 | {{3GPP-TS-26.114}} | probably not(?) |
| urn:3gpp:roi-sent | Signalling of the arbitrary region-of-interest (ROI) information for the sent video, see clause 6.2.3.4 | {{3GPP-TS-26.114}} | probably not(?) |
| urn:3gpp:predefined-roi-sent | Signalling of the predefined region-of-interest (ROI) information for the sent video, see clause 6.2.3.4 | {{3GPP-TS-26.114}} | probably not(?) |

### RTP SDES Compact Header Extensions

The IANA registry for this section is {{IANA-RTP-SDES-CHE}}.

| Extension URI | Description | Reference | Mappable from QUIC |
| ------------- | ----------- | --------- | ---- |
| urn:ietf:params:rtp-hdrext:sdes:cname | Source Description: Canonical End-Point Identifier (SDES CNAME) | {{?RFC7941}} | no |
| urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id | RTP Stream Identifier | {{?RFC8852}} | no |
| urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id | RTP Repaired Stream Identifier | {{?RFC8852}} | no |
| urn:ietf:params:rtp-hdrext:sdes:CaptId | CLUE CaptId | {{?RFC8849}} | no |
| urn:ietf:params:rtp-hdrext:sdes:mid | Media identification | {{?RFC9143}} | no |

## Examples {#rtcp-quic-ext-examples}

### Mapping QUIC Feedback to RTCP Receiver Reports ("RR") {#RR-mappings}

Considerations for mapping QUIC feedback into *Receiver Reports* (`PT=201`,
`Name=RR`, {{!RFC3550}}) are:

* *Fraction lost*: When RTP packets are carried in DATAGRAMs, the fraction of lost packets can be directly inferred from QUIC's acknowledgments.
The calculation includes all packets up to the acknowledged RTP packet with the highest RTP sequence number.
* *Cumulative lost*: Similar to the fraction of lost packets, the cumulative
  loss can be inferred from QUIC's acknowledgments, including all packets up
  to the latest acknowledged packet.
* *Highest Sequence Number received*: In RTCP, this field is a 32-bit field
  that contains the highest sequence number a receiver received in an RTP
  packet and the count of sequence number cycles the receiver has observed. A
  sender sends RTP packets in QUIC packets and receives acknowledgments for
  the QUIC packets. By keeping a mapping from a QUIC packet to the RTP packets
  encapsulated in that QUIC packet, the sender can infer the highest sequence
  number and number of cycles seen by the receiver from QUIC acknowledgments.
* *Interarrival jitter*: If QUIC acknowledgments carry timestamps as described
  in {{?I-D.draft-smith-quic-receive-ts}}, senders can infer the interarrival
  jitter from the arrival timestamps in QUIC acknowledgments.
* *Last SR*: Similar to lost packets, the NTP timestamp of the last received
  sender report can be inferred from QUIC acknowledgments.
* *Delay since last SR*: This field is not required when the receiver reports
  are entirely replaced by QUIC feedback.

### Congestion Control Feedback ("CCFB") {#CCFB-mappings}

RTP *Congestion Control Feedback* (`PT=205`, `FMT=11`, `Name=CCFB`,
{{!RFC8888}}) contains acknowledgments, arrival timestamps, and ECN
notifications for each received packet. Acknowledgments and ECNs can be inferred
from QUIC as described above. Arrival timestamps can be added through extended
acknowledgment frames as described in {{?I-D.draft-smith-quic-receive-ts}} or
{{?I-D.draft-huitema-quic-ts}}.

### Extended Report ("XR") {#XR-mappings}

*Extended Reports* (`PT=207`, `Name=XR`, {{!RFC3611}}) offer an extensible
framework for a variety of different control messages. Some of the statistics
that are defined as extended report blocks can be derived from QUIC, too. Other
report blocks need to be evaluated individually to determine whether the
contained information can be transmitted using QUIC instead. {{tab-xr-blocks}}
in {{extended-reports}} lists considerations for mapping QUIC feedback to some
of the *Extended Reports*.

### Application Layer Repair and other Control Messages {#al-repair}

While {{RR-mappings}} presented some RTCP packets that can be replaced by QUIC
features, QUIC cannot replace all of the defined RTCP packet types. This mostly
affects RTCP packet types, which carry control information that is to be
interpreted by the RTP application layer rather than the underlying transport
protocol itself.

* *Sender Reports* (`PT=200`, `Name=SR`, {{!RFC3550}}) are similar to *Receiver
  Reports*, as described in {{RR-mappings}}. They are sent by media senders and
  additionally contain an NTP and an RTP timestamp and the number of packets and
  octets transmitted by the sender. The timestamps can be used by a receiver to
  synchronize media streams. QUIC cannot provide similar control information since it
  does not know about RTP timestamps. A QUIC receiver cannot calculate the
  packet or octet counts since it does not know about lost DATAGRAMs. Thus,
  sender reports are necessary in RoQ to synchronize media streams at the receiver.

In addition to carrying transmission statistics, RTCP packets can contain
application layer control information that cannot directly be mapped to QUIC.
Examples of this information might include:

* *Source Description* (`PT=202`, `Name=SDES`) and *Application* (`PT=204`,
  `Name=APP`) packet types from {{!RFC3550}}, or
* many of the payload-specific feedback messages (`PT=206`) defined in
  {{!RFC4585}}, used to control the codec behavior of the sender.

Since QUIC does not provide any kind of application layer control messaging,
QUIC feedback cannot be mapped into these RTCP packet types. If the RTP
application needs this information, the RTCP packet types are used in the same
way as they would be used over any other transport protocol.

# Acknowledgments
{:numbered="false"}

Early versions of this document were similar in spirit to
{{?I-D.draft-hurst-quic-rtp-tunnelling}}, although many details differ. The
authors would like to thank Sam Hurst for providing his thoughts about how QUIC
could be used to carry RTP.

The guidance in {{quic-streams}} about configuring the number of parallel unidirectional QUIC streams is based on {{Section 6.2 of ?RFC9114}}, with obvious substitutions for RTP.

The authors would like to thank Bernard Aboba, David Schinazi, Lucas Pardue, Sam Hurst, Sergio Garcia Murillo,  and Vidhi Goel for their valuable comments and suggestions contributing to this document.
