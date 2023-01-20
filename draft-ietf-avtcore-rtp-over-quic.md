---
title: "RTP over QUIC"
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

--- abstract

This document specifies a minimal mapping for encapsulating RTP and RTCP packets
within QUIC. It also discusses how to leverage state from the QUIC
implementation in the endpoints to reduce the exchange of RTCP packets and how
to implement  congestion control and rate adaptation.

--- middle

# Introduction

The Real-time Transport Protocol (RTP) {{!RFC3550}} is generally used to carry
real-time media for conversational media sessions, such as video conferences,
across the Internet.  Since RTP requires real-time delivery and is tolerant to
packet losses, the default underlying transport protocol has been UDP, recently
with DTLS on top to secure the media exchange and occasionally TCP (and possibly
TLS) as a fallback.  With the advent of QUIC {{!RFC9000}} and, most notably, its
unreliable DATAGRAM extension {{!RFC9221}}, another secure transport protocol
becomes available.  QUIC and its DATAGRAMs combine desirable properties for
real-time traffic (e.g., no unnecessary retransmissions, avoiding head-of-line
blocking) with a secure end-to-end transport that is also expected to work well
through NATs and firewalls.

Moreover, with QUIC's multiplexing capabilities, reliable and unreliable
transport connections as, e.g., needed for WebRTC, can be established with only
a single port used at either end of the connection.  This document defines a
mapping of how to carry RTP over QUIC. The focus is on RTP and RTCP packet
mapping and on reducing the amount of RTCP traffic by leveraging state
information readily available within a QUIC endpoint. This document also
describes different options for implementing congestion control and rate adaptation for RTP over
QUIC.

The scope of this document is limited to unicast RTP/RTCP.

This document does not cover signaling for session setup. Signaling for RTP over
QUIC can be defined in separate documents such as
{{?I-D.draft-dawkins-avtcore-sdp-rtp-quic}} does for SDP.

Note that this draft is similar in spirit to but differs in numerous ways from
{{?I-D.draft-hurst-quic-rtp-tunnelling}}.

# Terminology and Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

The following terms are used:

Datagram:
: Datagrams exist in UDP as well as in QUIC's unreliable datagram extension. If not explicitly noted
differently, the term datagram in this document refers to a QUIC Datagram as defined in
{{!RFC9221}}.

Endpoint:
: A QUIC server or client that participates in an RTP over QUIC session.

Frame:
: A QUIC frame as defined in {{!RFC9000}}.

Media Encoder:
: An entity that is used by an application to produce a stream of encoded media, which can be
packetized in RTP packets to be transmitted over QUIC.

Receiver:
: An endpoint that receives media in RTP packets and may send or receive RTCP packets.

Sender:
: An endpoint that sends media in RTP packets and may send or receive RTCP packets.

Packet diagrams in this document use the format defined in {{Section 1.3 of RFC9000}} to
illustrate the order and size of fields.

# Scope

RTP over QUIC mostly defines an application usage of QUIC {{?I-D.draft-ietf-quic-applicability}}.
As a baseline, the specification does not expect more than a standard QUIC implementation
as defined in {{!RFC8999}}, {{!RFC9000}}, {{!RFC9001}}, and {{!RFC9002}}.
Nevertheless, the specification can benefit from QUIC extesions such as QUIC datagrams
{{!RFC9221}} as described below.
Moreover, this document describes how a QUIC implementation and its API can be
extended to improve efficiency of the protocol operation.

On top of QUIC, this document defines an encapsulation of RTP and RTCP packets.

The scope of this document is limited to carrying RTP over QUIC. It does not attempt
to enhance QUIC for real-time media or define a replacement or evolution of RTP.
Work to map other media transport protocols is under way elsewhere in the IETF.

Protocols for negotiating connection setup and the associated parameters are defined
separately, e.g., in {{?I-D.draft-dawkins-avtcore-sdp-rtp-quic}}.

# Protocol Overview

This document introduces a mapping of the Real-time Transport Protocol (RTP) to
the QUIC transport protocol. RTP over QUIC allows the use of QUIC streams and
QUIC datagrams to transport real-time data, and thus, the QUIC
implementation MUST support QUIC's datagram extension, if RTP packets
should be sent over QUIC datagrams. Since datagram frames cannot be fragmented,
the QUIC implementation MUST also provide a way to query the maximum datagram
size so that an application can create RTP packets that always fit into a QUIC
datagram frame.

{{!RFC3550}} specifies that RTP sessions need to be transmitted on different
transport addresses to allow multiplexing between them. RTP over QUIC uses a
different approach to leverage the advantages of QUIC connections without
managing a separate QUIC connection per RTP session. QUIC does not provide
demultiplexing between different flows on datagrams but suggests that an
application implement a demultiplexing mechanism if required. An example of such
a mechanism are flow identifiers prepended to each datagram frame as described
in {{Section 2.1 of ?I-D.draft-ietf-masque-h3-datagram}}. RTP over QUIC uses a
flow identifier to replace the network address and port number to multiplex many
RTP sessions over the same QUIC connection.

A rate adaptation algorithm can be plugged in to adapt the media bitrate to the
available bandwidth. This document does not mandate any specific rate adaptation
algorithm. Some examples include Network-Assisted Dynamic Adaptation (NADA)
{{!RFC8698}} and Self-Clocked Rate Adaptation for Multimedia (SCReAM)
{{!RFC8298}}. These rate adaptation algorithms require some feedback about
the network's performance to calculate target bitrates. Traditionally this
feedback is generated at the receiver and sent back to the sender via RTCP.
Since QUIC also collects some metrics about the network's performance, these
metrics can be used to generate the required feedback at the sender-side and
provide it to the rate adaptation algorithm to avoid the additional overhead of the
RTCP stream.

## Supported RTP Topologies {#topologies}

RTP over QUIC only supports some of the RTP topologies described in
{{?RFC7667}}. Most notably, due to QUIC being a purely unicast protocol at the
time of writing, RTP over QUIC cannot be used as a transport protocol in any of
the multicast topologies (e.g., *Topo-ASM*, *Topo-SSM*, *Topo-SSM-RAMS*).

RTP supports different types of translators and mixers. Whenever a middlebox
such as a translator or a mixer needs to access the content of RTP/RTCP-packets,
the QUIC connection has to be terminated at that middlebox.

Using RTP over QUIC streams (see {{quic-streams}}) can support much larger RTP
packet sizes than other transport protocols such as UDP can, which can lead to
problems with transport translators which translate from RTP over QUIC to RTP
over a different transport protocol. A similar problem can occur if a translator
needs to translate from RTP over UDP to RTP over QUIC datagrams, where the MTU
of a QUIC datagram may be smaller than the MTU of a UDP datagram. In both cases,
the translator may need to rewrite the RTP packets to fit into the smaller MTU
of the other protocol. Such a translator may need codec-specific knowledge to
packetize the payload of the incoming RTP packets in smaller RTP packets.

# Connection Establishment and ALPN {#alpn}

QUIC requires the use of ALPN {{!RFC7301}} tokens during connection setup. RTP
over QUIC uses "rtp-mux-quic" as ALPN token in the TLS handshake (see also
{{iana-considerations}}.

Note that the use of a given RTP profile is not reflected in the ALPN token even
though it could be considered part of the application usage.  This is simply
because different RTP sessions, which may use different RTP profiles, may be
carried within the same QUIC connection.

> **Editor's note:** "rtp-mux-quic" indicates that RTP and other protocols may
> be multiplexed on the same QUIC connection using a flow identifier as
> described in {{encapsulation}}. Applications are responsible for negotiation
> of protocols in use by appropriate use of a signaling protocol such as SDP.

> **Editor's note:** This implies that applications cannot use RTP over QUIC as
> specified in this document over WebTransport.

## Draft version identification

> **RFC Editor's note:** Please remove this section prior to publication of a
> final version of this document.

RTP over QUIC uses the token "rtp-mux-quic" to identify itself in ALPN.

Only implementations of the final, published RFC can identify themselves as
"rtp-mux-quic". Until such an RFC exists, implementations MUST NOT identify
themselves using this string.

Implementations of draft versions of the protocol MUST add the string "-" and
the corresponding draft number to the identifier.  For example,
draft-ietf-avtcore-rtp-over-quic-04 is identified using the string
"rtp-mux-quic-04".

Non-compatible experiments that are based on these draft versions MUST append
the string "-" and an experiment name to the identifier.

# Encapsulation

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

RTP over QUIC uses flow identifiers to multiplex different RTP, RTCP, and
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
flow identifier with any RTP/RTCP or non-RTP stream, it MAY drop the data
stream.

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

To send RTP/RTCP packets over QUIC streams, a sender MUST open a new
unidirectional QUIC stream. Streams are unidirectional because there is no
synchronous relationship between sent and received RTP/RTCP packets. A sender
MAY open new QUIC streams for different packets using the same flow identifier,
for example, to avoid head-of-line blocking.

{{fig-stream-payload}} shows the encapsulation format for RTP over QUIC Streams.

~~~
Payload {
  Flow Identifier (i),
  RTP/RTCP Payload(..) ...,
}
~~~
{: #fig-stream-payload title="RTP over QUIC Streams Payload Format"}

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

: A QUIC variable length integer {{Section 16 of !RFC9000}} describing the
length of the following RTP/RTCP packets in bytes.

RTP/RTCP Packet:

: The RTP/RTCP packet to transmit.

If a sender knows that a packet, which was not yet successfully and completely
transmitted, is no longer needed, the sender MAY close the stream by sending a
RESET\_STREAM frame.

A translator that translates between two endpoints, both connected via QUIC,
MUST forward RESET\_STREAM frames received from one end to the other unless it
forwards the RTP packets on QUIC datagrams.

> **Editor's Note:** It might be desired to also allow the receiver to request
> cancellation of a stream by sending STOP\_SENDING frame. However, this might
> lead to unintended packet loss because the receiver does not know which and
> how many packets follow on the same stream. If this feature is required, a
> solution could be to require senders to open new streams for each application
> data unit, as described in a previous version of this document.

Large RTP packets sent on a stream will be fragmented into smaller QUIC frames.
The QUIC frames are transmitted reliably and in order such that a receiving
application can read a complete RTP packet from the stream as long as the stream
is not closed with a RESET\_STREAM frame. No retransmission has to be
implemented by the application since QUIC frames lost in transit are
retransmitted by QUIC.

Opening new streams for new packets MAY implicitly limit the number of packets
concurrently in transit because the QUIC receiver provides an upper bound of
parallel streams, which it can update using QUIC MAX\_STREAMS frames. The number
of packets that have to be transmitted concurrently depends on several factors,
such as the number of RTP streams within a QUIC connection, the bitrate of the
media streams, and the maximum acceptable transmission delay of a given packet.
Receivers are responsible for providing senders with enough credit to open new
streams for new packets at any time.

## QUIC Datagrams {#quic-datagrams}

Senders can also transmit RTP packets in QUIC datagrams. QUIC datagrams are an
extension to QUIC described in {{!RFC9221}}. QUIC datagrams preserve frame
boundaries. Thus, a single RTP packet can be mapped to a single QUIC datagram
without additional framing. Senders SHOULD consider the header overhead
associated with QUIC datagrams and ensure that the RTP/RTCP packets, including
their payloads, flow identifier, QUIC, and IP headers, will fit into path MTU.

{{fig-dgram-payload}} shows the encapsulation format for RTP over QUIC
Datagrams.

~~~
Payload {
  Flow Identifier (i),
  RTP/RTCP Packet (..),
}
~~~
{: #fig-dgram-payload title="RTP over QUIC Datagram Payload Format"}

Flow Identifier:

: Flow identifier to demultiplex different data flows on the same QUIC
connection.

RTP/RTCP Packet:

: The RTP/RTCP packet to transmit.

Since QUIC datagrams are not retransmitted on loss (see also
{{transport-layer-feedback}} for loss signaling), if an application wishes to
retransmit lost RTP packets, the retransmission has to be implemented by the
application. RTP retransmissions can be done in the same RTP session or a
separate RTP session {{!RFC4588}} and the flow identifier MUST be set to the
flow identifier of the RTP session in which the retransmission happens.

# RTCP {#rtcp}

The RTP Control Protocol (RTCP) allows RTP senders and receivers to exchange
control information to monitor connection statistics and to identify and
synchronize streams. Many of the statistics contained in RTCP packets overlap
with the connection statistics collected by a QUIC connection. To avoid using up
bandwidth for duplicated control information, the information SHOULD only be
sent at one protocol layer. QUIC relies on certain control frames to be sent.

In general, applications MAY send RTCP without any restrictions. This document
specifies a baseline for replacing some of the RTCP packet types by mapping the
contents to QUIC connection statistics. Future documents can extend this mapping
for other RTCP format types. It is RECOMMENDED to expose relevant information
from the QUIC layer to the application instead of exchanging additional RTCP
packets, where applicable.

This section discusses what information can be exposed from the QUIC connection
layer to reduce the RTCP overhead and which type of RTCP messages cannot be
replaced by similar feedback from the transport layer. The list of RTCP packets
in this section is not exhaustive and similar considerations SHOULD be taken
into account before exchanging any other type of RTCP control packets.

## Transport Layer Feedback {#transport-layer-feedback}

This section explains how some of the RTCP packet types which are used to signal
reception statistics can be replaced by equivalent statistics that are already
collected by QUIC. The following list explains how this mapping can be achieved
for the individual fields of different RTCP packet types.

QUIC Datagrams are ack-eliciting packets, which means, that an acknowledgment is
triggered when a datagram frame is received. Thus, a sender can assume that an
RTP packet arrived at the receiver or was lost in transit, using the QUIC
acknowledgments of QUIC Datagram frames. In the following, an RTP packet is
regarded as acknowledged, when the QUIC Datagram frame that carried the RTP
packet, was acknowledged. For RTP packets that are sent over QUIC streams, an
RTP packet can be considered acknowledged, when all frames which carried
fragments of the RTP packet were acknowledged.

When QUIC Streams are used, the application should be aware that the direct
mapping proposed below may not reflect the real characteristics of the network.
RTP packet loss can seem lower than actual packet loss due to QUIC's automatic
retransmissions. Similarly, timing information might be incorrect due to
retransmissions.

Some of the transport layer feedback that can be implemented in RTCP contains
information that is not included in QUIC by default, but can be added via QUIC
extensions. One important example are arrival timestamps, which are not part of
QUIC's default acknowledgment frames, but can be added using
{{!I-D.draft-smith-quic-receive-ts}} or {{!I-D.draft-huitema-quic-ts}}. Another
extension, that can improve the precision of the feedback from QUIC is
{{!I-D.draft-ietf-quic-ack-frequency}}, which allows a sender to control the
delay of acknowledgments sent by the receiver.

The list of RTCP Receiver Reports that could be replaced by feedback from QUIC follows:

* *Receiver Reports* (`PT=201`, `Name=RR`, {{!RFC3550}})
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
* *Negative Acknowledgments* (`PT=205`, `FMT=1`, `Name=Generic NACK`,
  {{!RFC4585}})
  * The generic negative acknowledgment packet contains information about
    packets which the receiver considered lost. {{Section 6.2.1. of !RFC4585}}
    recommends to use this feature only, if the underlying protocol cannot
    provide similar feedback. QUIC does not provide negative acknowledgments,
    but can detect lost packets based on the Gap numbers contained in QUIC ACK frames.
* *ECN Feedback* (`PT=205`, `FMT=8`, `Name=RTCP-ECN-FB`, {{!RFC6679}})
  * ECN feedback packets report the count of observed ECN-CE marks. {{!RFC6679}}
    defines two RTCP reports, one packet type (with `PT=205` and `FMT=8`) and a
    new report block for the extended reports which are listed below. QUIC
    supports ECN reporting through acknowledgments. If the connection supports
    ECN, the reporting of ECN counts SHOULD be done using QUIC acknowledgments,
    rather than RTCP ECN feedback reports.
* *Congestion Control Feedback* (`PT=205`, `FMT=11`, `Name=CCFB`, {{!RFC8888}})
  * RTP Congestion Control Feedback contains acknowledgments, arrival timestamps
    and ECN notifications for each received packet. Acknowledgments and ECNs can
    be inferred from QUIC as described above. Arrival timestamps can be added
    through extended acknowledgment frames as described in
    {{!I-D.draft-smith-quic-receive-ts}} or {{!I-D.draft-huitema-quic-ts}}.
* *Extended Reports* (`PT=207`, `Name=XR`, {{!RFC3611}})
  * Extended Reports offer an extensible framework for a variety of different
    control messages. Some of the standard report blocks which can be
    implemented in extended reports such as loss RLE or ECNs can be implemented
    in QUIC, too. For other report blocks, it SHOULD be evaluated individually,
    if the contained information can be transmitted using QUIC instead.

## Application Layer Repair and other Control Messages

While the previous section presented some RTCP packet that can be replaced by
QUIC features, QUIC cannot replace all of the available RTCP packet types. This
mostly affects RTCP packet types which carry control information that is to be
interpreted by the application layer instead of the transport itself.

*Sender Reports* (`PT=200`, `Name=SR`, {{!RFC3550}}) are similar to *Receiver
Reports*. They are sent by media senders and additionally contain a NTP and a
RTP timestamp and the number of packets and octets transmitted by the sender.
The timestamps can be used by a receiver to synchronize streams. QUIC cannot
provide a similar control information, since it does not know about RTP
timestamps. Nor can a QUIC receiver calculate the packet or octet counts,
since it does not know about lost datagrams. Thus, sender reports are required
in RTP over QUIC to synchronize streams at the receiver. The sender reports
SHOULD not contain any receiver report blocks, as the information can be inferred
from the QUIC transport as explained in the previous section.

Next to carrying transmission statistics, RTCP packets can contain application
layer control information, that cannot directly be mapped to QUIC. This includes
for example the *Source Description* (`PT=202`, `Name=SDES`), *Bye* (`PT=203`,
`Name=BYE`) and *Application* (`PT=204`, `Name=APP`) packet types from
{{!RFC3550}} or many of the payload specific feedback messages (`PT=206`)
defined in {{!RFC4585}}, which can for example be used to control the codec
behavior of the sender. Since QUIC does not provide any kind of application
layer control messaging, these RTCP packet types SHOULD be used in the same way
as they would be used over any other transport protocol.

# Congestion Control and Rate Adaptaion {#congestion-control}

Like any other application on the internet, RTP over QUIC needs to perform
congestion control to avoid overloading the network.

QUIC is a congestion controlled transport protocol. Senders are required to
employ some form of congestion control. The default congestion control specified
for QUIC in {{!RFC9002}} is similar to TCP NewReno {{?RFC6582}}, but senders are free to choose
any congestion control algorithm as long as they follow the guidelines specified
in {{Section 3 of ?RFC8085}}.

RTP itself does not specify a congestion control algorithm, but {{!RFC8888}} defines an RTCP
feedback message intended to enable rate adaptation for interactive real-time traffic using RTP,
and successful rate adaptation will accomoplish congestion control as well.
Various rate adaptation algorithms for real-time media are defined in separate RFCs
(e.g. SCReAM {{!RFC8298}} and NADA {{!RFC8698}}).
The rate adaptation algorithms for RTP are specifically tailored for
real-time transmissions at low latencies. The available rate adaptation
algorithms for RTP expose a `target_bitrate` that can be used to dynamically
reconfigure media codecs to produce media at a rate that can be sent in
real-time under the observed network conditions.

This section defines two architectures for congestion control and bandwidth
estimation for RTP over QUIC, but it does not mandate any specific rate adaptation algorithm
to use. The section also discusses congestion control
implications of using shared or multiple separate QUIC connections to send and
receive multiple independent data streams.

It is assumed that the congestion controller in use provides a pacing mechanism
to determine when a packet can be sent to avoid bursts. The currently proposed
congestion control algorithms for real-time communications provide such pacing
mechanisms. The use of congestion controllers which don't provide a pacing
mechanism is out of scope of this document.

## Congestion Control at the QUIC layer {#cc-quic-layer}

If congestion control is to be applied at the transport layer, it is RECOMMENDED
that the QUIC Implementation uses a congestion controller that keeps queueing
delays short to keep the transmission latency for RTP and RTCP packets as low
as possible.

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

## Congestion Control at the Application Layer {#cc-application-layer}

If an application cannot access a bandwidth estimation from the QUIC layer, or
the QUIC implementation does not support a delay-based, low-latency congestion
control algorithm, the application can alternatively implement a bandwidth estimation
algorithm at the application layer. Calculating a bandwidth estimation at the
application layer can be done using the same bandwidth estimation algorithms as
described in {{congestion-control}} (NADA, SCReAM). The bandwidth estimation
algorithm typically needs some feedback on the transmission performance. This
feedback can be collected following the guidelines in {{rtcp}}.

If the application implements full congestion control rather than just a
bandwidth estimation at the application layer using a congestion controller that
satisfies the requirements of {{Section 7 of !RFC9002}}, and the connection is
only used to send real-time media which is subject to the application layer
congestion control, it is RECOMMENDED to disable any other congestion control
that is possibly running at the QUIC layer. Disabling the additional congestion
controllers helps to avoid any interference between the different congestion
controllers.

## Shared QUIC connections

Two endpoints may want to establish channels to exchange more than one type of data simultaneously.
The channels can be intended to carry real-time RTP data or other non-real-time data.
This can be realized in different ways.  A straightforward solution is to establish multiple QUIC
connections, one for each channel.  Or all real-time channels are mapped to one QUIC
connection, while a separate QUIC connection is created for the non-real-time channels.
In both cases, the congestion controllers can be chosen to match the demands of the respective
channels and the different QUIC connections will compete for the same resources in the network.
No local prioritization of data across the different (types of) channels would be necessary.

Alternatively, (all or a subset of) real-time and non-real-time channels are multiplexed onto a single,
shared QUIC connection, which can be done by using the flow identifier described
in {{encapsulation}}.  Applications multiplexing multiple streams in one
connection SHOULD implement some form of stream prioritization or bandwidth
allocation.

# API Considerations

The mapping described in the previous sections poses some interface requirements
on the QUIC implementation. Although a basic mapping should work without any of
these requirements most of the optimizations regarding rate adaptation and
RTCP mapping require certain functionalities to be exposed to the application.
The following to sections contain a list of information that is required by an
application to implement different optimizations ({{quic-api-read}}) and
functions that a QUIC implementation SHOULD expose to an application
({{quic-api-write}}).

Each item in the following list can be considered individually. Any exposed
information or function can be used by RTP over QUIC regardless of whether the
other items are available. Thus, RTP over QUIC does not depend on the
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
* *Disable Congestion Controller*: If congestion control is to be implemented at
  the application layer as described in {{cc-application-layer}}, and the
  application layer is trusted to apply adequate congestion control as described
  in {{Section 7 of !RFC9002}} and {{Section 3.1 of !RFC8085}}, it is
  RECOMMENDED to allow the application to disable QUIC layer congestion control
  entirely.

# Discussion

## Flow Identifier

{{!RFC9221}} suggests to use flow identifiers to multiplex different streams on
QUIC Datagrams, which is implemented in {{encapsulation}}, but it is unclear how
applications can combine RTP over QUIC with other data streams using the same
QUIC connections. If the non-RTP data streams use the same flow identifies, too
and the application can make sure, that flow identifiers are unique, there
should be no problem. Flow identifiers could be problematic, if different
specifications for RTP and non-RTP data streams over QUIC mandate different
incompatible flow identifiers.

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
are allowed.  0-RTT data may be beneficial for use with RTP over QUIC to reduce the
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

# Security Considerations

RTP over QUIC is subject to the security considerations of RTP described in
{{Section 9 of !RFC3550}} and the security considerations of any RTP profile in
use.

The security considerations for the QUIC protocol and datagram extension
described in {{Section 21 of !RFC9000}}, {{Section 9 of !RFC9001}}, {{Section 8
of !RFC9002}} and {{Section 6 of !RFC9221}} also apply to RTP over QUIC.

# IANA Considerations {#iana-considerations}

## Registration of a RTP over QUIC Identification String

This document creates a new registration for the identification of RTP over QUIC
in the "TLS Application-Layer Protocol Negotiation (ALPN) Protocol IDs" registry
{{?RFC7301}}.

The "rtp-mux-quic" string identifies RTP over QUIC:

  Protocol:
  : RTP over QUIC

  Identification Sequence:
  : 0x72 0x74 0x70 0x2D 0x6F 0x76 0x65 0x72 0x2D 0x71 0x75 0x69 0x63 ("rtp-mux-quic")

  Specification:
  : This document

--- back

# Experimental Results

An experimental implementation of the mapping described in this document can be
found on [Github](https://github.com/mengelbart/rtp-over-quic). The application
implements the RTP over QUIC Datagrams mapping and implements SCReAM congestion
control at the application layer. It can optionally disable the builtin QUIC
congestion control (NewReno). The endpoints only use RTCP for congestion control
feedback, which can optionally be disabled and replaced by the QUIC connection
statistics as described in {{transport-layer-feedback}}.

Experimental results of the implementation can be found on
[Github](https://github.com/mengelbart/rtp-over-quic-mininet), too.


# Acknowledgments
{:numbered="false"}

The authors would like to thank David Schinazi, Lucas Pardue, Spencer Dawkins, and Vidhi Goel
for their valuable comments and suggestions contributing to this document.
