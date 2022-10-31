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
to implement congestion control.

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
describes different options for implementing congestion control for RTP over
QUIC. The mapping specified in this document relies on the WebTransport Protocol
Framework {{!I-D.draft-ietf-webtrans-overview-04}} to implement multiplexing of
RTP/RTCP sessions and other protocols in a single QUIC connection.

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
: Datagrams exist in UDP as well as in QUICs unreliable datagram extension. If not explicitly noted
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

RTP over QUIC mostly defines an application usage of QUIC
{{?I-D.draft-ietf-quic-applicability}}. The specification relies on the
WebTransport Protocol Framework {{!I-D.draft-ietf-webtrans-overview-04}} to
provide a multiplexing layer on top of QUIC.

Additionally, the specification can benefit from QUIC extesions such as QUIC
datagrams {{!RFC9221}} as described below. Moreover, this document describes how
a QUIC implementation and its API can be extended to improve efficiency of the
protocol operation.

On top of WebTransport, this document defines an encapsulation of RTP and RTCP
packets.

The scope of this document is limited to carrying RTP over QUIC. It does not attempt
to enhance QUIC for real-time media or define a replacement or evolution of RTP.
Such new media transport protocols may be covered elsewhere, e.g., in the MOQ WG.

Protocols for negotiating connection setup and the associated parameters are defined
separately, e.g., in {{?I-D.draft-dawkins-avtcore-sdp-rtp-quic}}.

# Protocol Overview

RTP over QUIC relies on the WebTransport protocol framework
{{!I-D.draft-ietf-webtrans-overview-04}} to implement multiplexing of many RTP
sessions and potentially also other protocols over the same QUIC connection.
This next section gives an overview over the WebTransport protocol framework and
how it is used to map RTP to QUIC connections. The following section describes
limitations of RTP over QUIC regarding different RTP topologies.

## WebTransport Overview

The WebTransport Protocol Framework defines a set of abstract features that can
be provided by any concrete transport implementation. Next to session
establishment and termination, WebTransport provides abstractions for sending
and receiving bytes on one or more uni- or bidirectional streams as well as
sending and receiving datagrams.

Multiple WebTransport sessions can be multiplexed over the same transport
protocol. Relying on WebTransport as abstraction from QUIC allows RTP over
QUIC to multiplex multiple RTP sessions and possibly other protocol sessions
(e.g., for data transfers or signaling) into the same transport connection.

WebTransport protocols are required to limit the rate at which clients can send
data. The implications of such rate limiting for real-time applications using
RTP are discussed in detail in {{congestion-control}}.

Although there might be WebTransport protocol instances that do not use QUIC as
the underlying transport protocol, this document is written with specfically
QUIC in mind. The general mapping allows the usage of other transport protocols,
but care should be taken if other protocols are employed due to their varying
support for features such as unreliable delivery. Some transport protocols may
emulate certain features such as unreliable datagrams or independent streams.
For an application they might look the same compared to a QUIC connection, but
in reality, the implementation may still retransmit datagrams or suffer from
head-of-line blocking between streams.

## Supported RTP Topologies {#topologies}

This specification only supports some of the RTP topologies described in
{{?RFC7667}}. Most notably, due to WebTransport offering only unicast transports
at the time of writing, it cannot be used as a multicast protocol in any of the
multicast topologies (e.g., *Topo-ASM*, *Topo-SSM*, *Topo-SSM-RAMS*).

RTP supports different types of translators and mixers. Whenever a middlebox
such as a translator or a mixer needs to access the content of RTP/RTCP-packets,
the transport connection has to be terminated at that middlebox.

Using RTP over streams (see {{streams}}) can support much larger RTP packet
sizes than other transport protocols such as UDP can, which can lead to problems
with transport translators which translate from RTP over QUIC to RTP over such a
transport protocol. A similar problem can occur if a translator needs to
translate from RTP over UDP to RTP over WebTransport datagrams, where the MTU of
a WebTransport datagram may be smaller than the MTU of a UDP datagram. In both
cases, the translator may need to rewrite the RTP packets to fit into the
smaller MTU of the other protocol. Such a translator may need codec-specific
knowledge to packetize the payload of the incoming RTP packets in smaller RTP
packets.

# Encapsulation {#encapsulation}

WebTransports supports two transport modes: reliable streams and unreliable
datagrams. This document specifies mappings of RTP to both modes. {{streams}}
and {{datagrams}} explain the specifics of mapping of RTP to streams and
datagrams respectively. Senders MAY combine both modes by sending some RTP/RTCP
packets over the same or different streams as described in {{streams}} and
others in datagrams as described in {{datagrams}}.

## Streams {#streams}

WebTransport protocols provide client- and server-initiated uni- and
bidirectional streams. RTP/RTCP packets MUST be sent in unidirectional streams,
because there is no synchronous relationship between sent and received RTP/RTCP
packets.

Senders can send RTP/RTCP packets over streams using an encapsulation similar to
the one defined in {{!RFC4571}}, but instead of using a 16-bit length field,
each packet is prefixed with a variable length integer as defined in {{Section
16 of !RFC9000}}. Using a variable length integer as the length field allows to
compose larger packets, which avoids RTP packet fragmentation, while keeping the
size of the length field to a minimum.

The encapsulation format for RTP over Streams is shown in
{{fig-stream}} and {{fig-stream-payload}}.

~~~
Stream {
  Payload (..) ...,
}
~~~
{: #fig-stream title="RTP over Streams Payload Format."}

~~~
Payload {
  Length(i),
  RTP/RTCP Packet(..),
}
~~~
{: #fig-stream-payload title="Content of a payload sent on a stream"}

Length:

: A variable length integer ({{Section 16 of !RFC9000}}) describing the length
of the following RTP/RTCP packets in bytes.

RTP/RTCP Packet:

: The RTP/RTCP packet to transmit.

Senders MAY open new streams for new RTP packets to avoid head-of-line blocking
or they MAY send multiple RTP packets on the same stream. In both cases, each
packet MUST be prepended with the packets length.

Senders MAY abort the send side of a stream at any time. Aborting a stream
allows a sender to cancel retransmissions of data that is no longer needed,
e.g., a media frame that has passed its playout deadline. A sender should keep
in mind, that aborting a stream cancels (re-)transmissions of all data on that
stream and thus should only be used when all of the outstanding data can safely
be discarded.

A translators that translates between two endpoints, which are both connected
via a WebTransport protocol, MUST forward abortion of streams received from one
end to the other end, unless it is forwarding the RTP packets on datagrams.

> **Editor's Note:** It might be desired to also allow the receiver to abort a
> stream. However, this might lead to unintended packet loss, because the
> receiver does not know which and how many packets follow on the same stream.
> If this feature is required, a solution could be to require senders to open
> new streams for each application data unit, as described in a previous version
> of this document. Another solution might involve some form of additional
> signaling between sender and receiver.

Large RTP packets sent on a stream can be fragmented in smaller frames at the
transport layer. The frames of a single stream are transmitted reliably and in
order, such that a receiving application can read a complete packet from the
stream. No retransmission has to be implemented by the application, since frames
that are lost in transit are retransmitted by the underlying transport protocol.

> **TODO:** The following paragraph assumes that an application has some say
> about flow control, if that is not the case in WebTransport protocols, we
> should remove it or make it less a requirement of the application.

Opening new streams for new packets implicitly limits the amount of packets
which are concurrently in transit. The number of packets which have to be
transmitted concurrently depends on a number of factors such as the number of
RTP sessions within a QUIC connection, the rate at which new application data is
produced and the maximum acceptable transmission delay of a given packet.
Receivers are responsible for providing senders with enough credit to open new
streams for new packets at any time.

## Datagrams {#datagrams}

RTP packets can be sent in unreliable datagrams. Unreliable datagrams preserve
frame boundaries, thus a single RTP packet can be mapped to a single datagram,
without the need for an additional framing. WebTransport protocols are required
to expose the maximum datagram size the application can send. Applications MUST
NOT try to send RTP packets exceeding the maximum datagram size limit.

The encapsulation format for Datagrams is described in
{{fig-dgram-payload}}.

~~~
Datagram {
  RTP/RTCP Packet (..),
}
~~~
{: #fig-dgram-payload title="Datagram Payload Format"}

RTP/RTCP Packet:

: The RTP/RTCP packet to transmit.

Since datagrams are not retransmitted on loss (see also
{{transport-layer-feedback}} for loss signaling), if an application wishes to
retransmit lost RTP packets, the retransmission has to be implemented by the
application.

# RTCP {#rtcp}

The RTP Control Protocol (RTCP) allows RTP senders and receivers to exchange
control information to monitor connection statistics and to identify and
synchronize streams. Many of the statistics contained in RTCP packets may
overlap with the connection statistics collected by a transport protocol. QUIC
is one example of a protocol that collects a number of metrics that overlap with
the connection statistics derived from RTCP. To avoid using up bandwidth for
duplicated control information, the information SHOULD only be sent at one
protocol layer.

In general, applications using a WebTransport protocol MAY send RTCP without
any restrictions. This document specifies a baseline for replacing some of the
RTCP packet types by mapping the contents to connection statistics of underlying
the transport. Future documents can extend this mapping for other RTCP format
types. It is RECOMMENDED to expose relevant information from the transport layer
to the application instead of exchanging addtional RTCP packets, where
applicable.

This section discusses what information can be exposed from the transport
layer to reduce the RTCP overhead and which type of RTCP messages cannot be
replaced by similar feedback from the transport layer. The list of RTCP packets
in this section is not exhaustive and similar considerations SHOULD be taken
into account before exchanging any other type of RTCP packets.

## Transport Layer Feedback {#transport-layer-feedback}

This section explains how some of the RTCP packet types which are used to signal
reception statistics can be replaced by equivalent statistics that may already
be collected at the transport layer. The following list explains how this
mapping can be achieved for the individual fields of different RTCP packet
types.

When reliable Streams are used, the application should be aware that the direct
mapping proposed below may not reflect the real characteristics of the network.
RTP packet loss can seem lower than actual packet loss due to automatic
retransmissions. Similarly, timing information might be incorrect due to
retransmissions.

Acknowledgments play in important part in the analysis below. Due to the
reliable nature of streams, it is likely that there will be some form of
acknowledgment available for RTP packets that are sent over reliable streams.
For RTP packets that are sent over QUIC streams, an RTP packet can be considered
acknowledged, when all frames which carried fragments of the RTP packet were
acknowledged.

As datagrams are unreliable, there is no need for acknowledgments. However, some
transport protocols might still use acknowledgments for unreliable datagrams. In
QUIC, Datagrams are ack-eliciting packets, which means, that an acknowledgment is
triggered when a datagram frame is received. Thus, a sender can assume that an
RTP packet arrived at the receiver or was lost in transit, using QUICs loss
detection.

Some of the transport layer feedback that can be implemented in RTCP contains
information that is not included in every WebTransport protocol by default, but
can be added via extensions. One important example are arrival timestamps, which
are not part of QUIC's default acknowledgment frames, but can be added using
{{!I-D.draft-smith-quic-receive-ts}} or {{!I-D.draft-huitema-quic-ts}}. Another
extension, that can improve the precision of QUIC's feedback is
{{!I-D.draft-ietf-quic-ack-frequency}}, which allows a sender to control the
delay of acknowledgments sent by the receiver.

* *Receiver Reports* (`PT=201`, `Name=RR`, {{!RFC3550}})
  * *Fraction lost*: The fraction of lost packets can be directly infered from
    acknowledgments. The calculation SHOULD include all packets up to the
    acknowledged RTP packet with the highest RTP sequence number. Later packets
    SHOULD be ignored, since they may still be in flight, unless other packets
    that were sent on the same connection and after the packets that contained
    the RTP packet, were already acknowledged.
  * *Cumulative lost*: Similar to the fraction of lost packets, the cumulative
    loss can be infered from acknowledgments including all packets up to the
    latest acknowledged packet.
  * *Highest Sequence Number received*: The highest sequence number received is
    the highest sequence number of all RTP packets that were acknowledged.
  * Interarrival jitter: If acknowledgments carry timestamps as described in one
    of the extensions referenced above, senders can infer the interarrival
    jitter from the arrival timestamps.
  * Last SR: Similar to RTP arrival times, the arrival time of RTCP Sender
    Reports can be inferred from acknowledgments, if they include timestamps.
  * Delay since last SR: This field is not required when the receiver reports
    are entirely replaced by transport layer feedback.
* *Negative Acknowledgments* (`PT=205`, `FMT=1`, `Name=Generic NACK`,
  {{!RFC4585}})
  * The generic negative acknowledgment packet contains information about
    packets which the receiver considered lost. {{Section 6.2.1. of !RFC4585}}
    recommends to use this feature only, if the underlying protocol cannot
    provide similar feedback. Acknowledgments and loss detection of an
    underlying transport protocol can be used as an alternative to RTCP negative
    acknowledgments.
* *ECN Feedback* (`PT=205`, `FMT=8`, `Name=RTCP-ECN-FB`, {{!RFC6679}})
  * ECN feedback packets report the count of observed ECN-CE marks. {{!RFC6679}}
    defines two RTCP reports, one packet type (with `PT=205` and `FMT=8`) and a
    new report block for the extended reports which are listed below. If the
    transport protocol supports ECN, the reporting of ECN counts SHOULD be done
    at the transport layer instead of using RTCP.
* *Congestion Control Feedback* (`PT=205`, `FMT=11`, `Name=CCFB`, {{!RFC8888}})
  * RTP Congestion Control Feedback contains acknowledgments, arrival timestamps
    and ECN notifications for each received packet. Acknowledgments and ECNs can
    be infered from a transport protocol as described above. Arrival timestamps
    can be added to QUIC through extended acknowledgment frames as described in
    {{!I-D.draft-smith-quic-receive-ts}} or {{!I-D.draft-huitema-quic-ts}}.
* *Extended Reports* (`PT=207`, `Name=XR`, {{!RFC3611}})
  * Extended Reports offer an extensible framework for a variety of different
    control messages. Some of the standard report blocks which can be
    implemented in extended reports such as loss RLE or ECNs can be implemented
    in WebTransport protocols, too. For other report blocks, it SHOULD be
    evaluated individually, if the contained information can be transmitted
    using the transport protocol instead of RTCP.

## Application Layer Repair and other Control Messages

While the previous section presented a list of RTCP packets that can be replaced
by transport layer features, many RTCP packets cannot be mapped to transport
layer features.

*Sender Reports* (`PT=200`, `Name=SR`, {{!RFC3550}}) are similar to *Receiver
Reports*. They are sent by media senders and additionally contain a NTP and a
RTP timestamp and the number of packets and octets transmitted by the sender.
The timestamps can be used by a receiver to synchronize streams. Transport
protocols typically cannot provide a similar control information, since they do
not have any knowledge of RTP timestamps. A receiver can also not calculate the
packet or octet counts, since it does not know about lost datagrams. Thus,
sender reports are required to synchronize streams at the receiver. The sender
reports SHOULD not contain any receiver report blocks, if the information can be
infered from the transport as explained in the previous section.

Next to carrying transmission statistics, RTCP packets can contain application
layer control information, that cannot directly be mapped to the transport
layer. This includes for example the *Source Description* (`PT=202`,
`Name=SDES`), *Bye* (`PT=203`, `Name=BYE`) and *Application* (`PT=204`,
`Name=APP`) packet types from {{!RFC3550}} or many of the payload specific
feedback messages (`PT=206`) defined in {{!RFC4585}}, which can for example be
used to control the codec behavior of the sender. Since WebTransport does not
provide any kind of application layer control messaging, these RTCP packet types
SHOULD be used in the same way as they would be used over any other transport
protocol.

# Congestion Control {#congestion-control}

Like any other application on the internet, RTP needs to perform congestion
control to avoid overloading the network.

RTP does not specify a congestion controller, but provides feedback formats for
congestion control (e.g. {{?RFC8888}}) as well as different congestion control
algorithms in separate RFCs (e.g. SCReAM {{?RFC8298}} and NADA {{?RFC8698}}).
The congestion control algorithms for RTP are specifically tailored for
real-time transmissions at low latencies. The available congestion control
algorithms for RTP expose a `target_bitrate` that can be used to dynamically
reconfigure media codecs to produce media at a rate that can be sent in
real-time under the observed network conditions.

WebTransport protocols are required to limit the rate at which a clients sends
data. Senders are free to choose how to accomplish this requirement.

This section defines two architectures for congestion control and bandwidth
estimation, but it does not mandate any specific congestion control algorithm to
use. The section also discusses congestion control implications of using shared
or multiple separate QUIC connections to send and receive multiple independent
data streams.

It is assumed that the congestion controller in use provides a pacing mechanism
to determine when a packet can be sent to avoid bursts. The currently proposed
congestion control algorithms for real-time communications provide such pacing
mechanisms. The use of congestion controllers which don't provide a pacing
mechanism is out of scope of this document.

## Congestion Control at the Transport layer {#cc-transport-layer}

If congestion control is to be applied at the transport layer, it is RECOMMENDED
that the transport Implementation uses a congestion controller that keeps
queueing delays short to keep the transmission latency for RTP and RTCP packets
as low as possible.

Many low latency congestion control algorithms depend on detailed arrival time
feedback to estimate the current one-way delay between sender and receiver. QUIC
does not provide arrival timestamps in its acknowledgments. The QUIC
implementations of the sender and receiver can use an extension to add this
information to QUICs acknowledgment frames, e.g.
{{!I-D.draft-smith-quic-receive-ts}} or {{!I-D.draft-huitema-quic-ts}}.

To adapt media codec configurations to the current bandwidth capacity, the
application also needs a mechanism to query the currently available bandwidth
estimation. The employed congestion controller SHOULD expose such an estimation
to the application. If a current bandwidth estimation is not available from the
congestion controller, the sender can either implement an alternative bandwidth
estimation at the application layer as described in {{cc-application-layer}} or
a receiver can feedback the observed bandwidth through RTCP, e.g., using
{{?I-D.draft-alvestrand-rmcat-remb}}.

## Congestion Control at the Application Layer {#cc-application-layer}

If an application cannot access a bandwidth estimation from the transport layer,
or the transport implementation does not support a delay-based, low-latency
congestion control algorithm, it can alternatively implement a bandwidth
estimation algorithm at the application layer. Calculating a bandwidth
estimation at the application layer can be done using the same bandwidth
estimation algorithms as described in {{congestion-control}} (NADA, SCReAM). The
bandwidth estimation algorithm typically needs some feedback on the transmission
performance. This feedback can be collected following the guidelines in
{{rtcp}}.

Some transport implementations might offer an configuration option to disable
the congestion controller completely. It is RECOMMENDED that an application
disables the transport congestion controller, if all of the following conditions
are met:

1. the application implements full congestion control (rather than just
   computing a bandwidth estimation),
2. the application is using a congestion controller that satisfies the
   requirements of {{Section 7 of !RFC9002}},
3. the connection is only used to send real-time media which is subject to the
   application layer congestion control.

Disabling the additional congestion controllers helps to avoid any interference
between the different congestion controllers.

## Shared QUIC connections

> **TODO:** This following paragraphs are probably not needed if we rely on
> WebTransport as multiplexing layer. However, it may be helpful to keep this
> section for considerations/expectations to the multiplexing layer regarding
> bandwidth sharing. E.g., if two WebTransport sessions share a QUIC connection,
> it would be good if a session using RTP could rely on a certain share of the
> bandwidth.

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

# API Considerations {#api-considerations}

The mapping described in the previous sections poses some interface requirements
on the transport implementation. Although a basic mapping should work without
any of these requirements most of the optimizations regarding congestion control
and RTCP mapping require certain functionalities to be exposed to the
application. The following two sections contain a list of information
({{transport-api-read}}) and functions that a transport implementation SHOULD
expose to an application ({{transport-api-write}}), to enable the application to
implement different optimizations.

Each item in the following list can be considered individually. Any exposed
information or function can be used by the application regardless of whether the
other items are available. Thus, the application does not depend on the
availability of all of the listed features to use RTP, but can apply different
optimizations depending on the functionality exposed by the transport
implementation.

## Information to be exported from the Transport {#transport-api-read}

This section provides a list of items that an application might want to export
from an underlying transport implementation.

* *Maximum Datagram Size*: The maximum datagram size that the application can
  transmit over the underlying transport connection.
* *Datagram Acknowledgment and Loss*: {{Section 5.2 of !RFC9221}} allows QUIC
  implementations to notify the application that a QUIC Datagram was
  acknowledged or that it believes a datagram was lost. The exposed information
  SHOULD include enough information to allow the application to maintain a
  mapping between the datagram that was acknowledged/lost and the RTP packet
  that was sent in that datagram.
* *Stream States*: The transport implementation SHOULD expose to a sender, how
  much of the data that was sent on a stream was successfully delivered and how
  much data is still outstanding to be sent or retransmitted.
* *Arrival timestamps*: If the transport connection uses timestamps like
  {{I-D.draft-smith-quic-receive-ts}} or {{I-D.draft-huitema-quic-ts}}, the
  arrival timestamps or one-way delays SHOULD be exposed to the application.
* *Bandwidth Estimation*: If congestion control is done at the transport layer,
  the transport implementation SHOULD expose an estimation of the currently
  available bandwidth to the application. Exposing the bandwidth estimation
  avoids the implementation of an additional bandwidth estimation algorithm in
  the application.
* *ECN*: If ECN marks are available, they SHOULD be exposed to the application.

## Functions to be exposed by the Transport {#transport-api-write}

This sections lists functions that a transport implementation SHOULD expose to
an application to implement different features of the mapping described in the
previous sections of this document.

* *Cancel Streams*: To allow an application to cancel (re)transmission of
  packets that are no longer needed, the transport implementation SHOULD expose
  a way to cancel the corresponding transport streams.
* *Configure Congestion Controller*: If congestion control is to be implemented
  at the transport layer as described in {{cc-transport-layer}}, the
  implementation SHOULD expose an API to allow the application to configure the
  specifics of the congestion controller.
* *Disable Congestion Controller*: If congestion control is to be implemented at
  the application layer as described in {{cc-application-layer}}, and the
  application layer is trusted to apply adequate congestion control as described
  in {{Section 7 of !RFC9002}} and {{Section 3.1 of !RFC8085}}, it is
  RECOMMENDEDto allow the application to disable transport layer congestion
  control entirely.

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

This document has no IANA considerations.

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

The authors would like to thank Spencer Dawkins, Lucas Pardue and David Schinazi
for their valuable comments and suggestions contributing to this document.
