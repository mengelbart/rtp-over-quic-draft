---
title: "RTP over QUIC"
docname: draft-engelbart-rtp-over-quic-latest
category: info
date: {DATE}

ipr: trust200902
area: General
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

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
    email: mathis.engelbart@tum.de

normative:
  QUIC-TRANSPORT:
    title: "QUIC: A UDP-Based Multiplexed and Secure Transport"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-transport-34
    author:
      -
        ins: J. Iyengar
        name: Jana Iyengar
        org: Google
        role: editor
      -
        ins: M. Thomson
        name: Martin Thomson
        org: Mozilla
        role: editor

  QUIC-RECOVERY:
    title: "QUIC Loss Detection and Congestion Control"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-recovery-34
    author:
      -
        ins: J. Iyengar
        name: Jana Iyengar
        org: Google
        role: editor
      -
        ins: I. Swett
        name: Ian Swett
        org: Google
        role: editor

  QUIC-DATAGRAM:
    title: "An Unreliable Datagram Extension to QUIC"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-datagram-02
    author:
      -
        ins: T. Pauly
        name: Tommy Pauly
        org: Apple Inc.
        role: editor
      -
        ins: E. Kinnear
        name: Eric Kinnear
        org: Apple Inc.
        role: editor
      -
        ins: D. Schinazi
        name: David Schinazi
        org: Google LLC
        role: editor

  QUIC-TS:
    title: "Quic Timestamps For Measuring One-Way Delays"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-huitema-quic-ts-05
    author:
      -
        ins: C. Huitema
        name: Christian Huitema
        org: Private Octopus Inc.
        role: editor
informative:



--- abstract

This document specifies a minimal mapping for encapsulating RTP and
RTCP packets within QUIC.  It also discusses how to leverage state
from the QUIC implementation in the endpoints to reduce the exchange
of RTCP packets.

--- middle

# Introduction

The Real-time Transport Protocol (RTP) {{!RFC3550}} is generally used
to carry real-time media for conversational media sessions, such as
video conferences, across the Internet.  Since RTP requires real-time
delivery and is tolerant to packet losses, the default underlying
transport protocol has been UDP, recently with DTLS on top to secure
the media exchange and occasionallly TCP (and possibly TLS) as
fallback.  With the advent of QUIC and, most notably, its unreliable
DATAGRAM extension, another secure transport protocol becomes
avaialble.  QUIC and its DATAGRAMs combine desirable properties for
real-time traffic (e.g., no unnecessary retransmissions, avoiding
head-of-line blocking) with a secure end-to-end transport that is
also expected to work well through NATs and firewalls.

Moreover, with QUIC's multiplexing capabilities, reliable and
unreliable transport connections as, e.g., needed for WebRTC, can be
established with only a single port used at either end of the
connection.  This document defines a mapping of how to carry RTP over
QUIC.  The focus is on RTP and RTCP packet mapping and on reducing the
amount of RTCP traffic by leveraging state information readily
available within a QUIC endpoint.  This document also briefly touches
upon how to signal media over QUIC using the Session Description
Protocol (SDP) {{!RFC4566}}.

The scope of this document is limited to unicast RTP/RTCP.


# Terminology and Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Protocol Overview

This document introduces a mapping of the Real-time Transport Protocol (RTP) to the QUIC transport
protocol. QUIC supports two transport methods: reliable streams and unreliable datagrams
{{QUIC-TRANSPORT}}, {{QUIC-DATAGRAM}}. RTP over QUIC uses the unreliable datagrams to transport
real-time data.

{{!RFC3550}} specifies that RTP sessions need to be transmitted on different transport addresses to
allow multiplexing between them. RTP over QUIC uses a different approach, in order to leverage the
advantages of QUIC connections without managing a separate QUIC connection per RTP session. QUIC
does not provide multiplexing between different flows on datagrams, but recommends the use of
variable-length integers to prepend each datagram with a unique flow identifier for multiplexing
flows. RTP over QUIC uses a flow identifier as a replacement for network address and port number, to
multiplex many RTP sessions over the same QUIC connection.

A congestion controller can be plugged in, to adapt the media bitrate to the available bandwidth.
This document does not mandate any congestion control algorithm, some examples include
Network-Assisted Dynamic Adaptation (NADA) {{!RFC8698}} and Self-Clocked Rate Adaptation for
Multimedia (SCReAM) {{!RFC8298}}. These congestion control algorithms require some feedback about
the performance of the network in order to calculate target bitrates. Traditionally this feedback is
generated at the receiver and sent back to the sender via RTCP. Since QUIC also collects some
metrics about the networks performance, these metrics can be used to generate the required feedback
at the sender-side and provide it to the congestion controller, to avoid the additional overhead of
the RTCP stream.

> **TODO:** Should the congestion controller work independently from the congestion
> controller used in QUIC, because the QUIC connection can simultaneously be used for other data
> streams, that need to be congestion controlled, too?

# Local Interfaces

RTP over QUIC requires different components like QUIC implementations, congestion controllers and
media encoders to work together. The interfaces of these components have to fulfill certain
requirements which are described in this section.

## QUIC Interface

If the used QUIC implementation is not directly incorporated into the RTP over QUIC mapping
implementation, it has to fulfill the following interface requirements. The QUIC implementation MUST
support QUICs unreliable datagrams and it MUST provide a way to signal acknowledgements or losses of
datagrams to the application. Since datagram frames cannot be fragmented, the QUIC implementation
MUST provide a way to query the maximum datagram size, so that an application can create RTP packets
that always fit into a datagram.

Additionally, a QUIC implementation MUST expose the recorded RTT statistics as described in
{{Section 5 of QUIC-RECOVERY}} to the application. These statistics include the minium observed RTT
over a period of time (`min_rtt`), exponentially-weighted moving average (`smoothed_rtt`) and the mean
deviation (`rtt_var`). These values are necessary to perform congestion control as explained in
{{cc-interface}}.

{{Section 7.1 of QUIC-RECOVERY}} also specifies how QUIC treats ECN marks if ECN is supported by the
network path. If ECN counts can be exported from a QUIC implementation, these may be used to improve
congestion control, too.

## Congestion Controller Interface {#cc-interface}

There are different congestion control algorithms proposed by RMCAT to implement application layer
congestion control for real-time communications. To estimate the currently available bandwidth,
these algorithms keep track of the sent packets and typically require a list of received packets
together with the timestamps at which they were received by a receiver. The bandwidth estimation can
then be used to decide, whether the media encoder can be configured to produce output at a higher or
lower rate.

A congestion controller used for RTP over QUIC should be able to compute an adequate bandwidth
estimation using the following inputs:

* `t_current`: A current timestamp
* `pkt_status_list`: A list of packets that were acknowledged by the receiver
* `pkt_delay_list`: For each acknowledged packet, a delay between the sent- and receive-timestamps
  of the packet
* The RTT estimations calculated by QUIC as described in {{Section 5 of QUIC-RECOVERY}}:
  * `min_rtt`: The miminum RTT observed by QUIC over a period of time
  * `smoothed_rtt`: An exponentially-weighted moving average of the observed RTT values
  * `rtt_var`: The mean deviation in the observed rtt values
* `ecn`: Optionally ECN marks may be used, if supported by the network and exposed by the QUIC
  implementation.

A congestion controller MUST expose a `target_bitrate` to which the encoder should be configured to
fully utilize the available bandwidth.

It is assumed that the congestion controller provides a pacing mechanism to determine when a packet
can be send and to avoid bursts. All of the currently proposed congestion control algorithms for
real-time communications provide such a pacing mechanism. The use of congestion controllers which
don't provide a pacing mechanism is out of scope of this document.

## Codec Interface {#encoder-interface}

An application is expected to adapt the media bitrate to the observed available bandwidth by setting
the media encoder to the `target_bitrate` that is computed by the congestion controller. Thus, the
media encoder needs to offer a way to update its bitrate accordingly. An implementation can either
expose the most recent `target_bitrate` produced by the congestion controller, or directly take a
callback to update the encoder bitrate which is called whenever the congestion controller updates
its `target_bitrate`.

# Packet Format {#packet-format}

All packets MUST be sent as datagrams with the following format:

~~~
Datagram Payload {
  Flow Identifier (i),
  RTP/RTCP Packet (..)
}
~~~
{: #fig-datagram-payload title="Datagram Payload Format"}

For multiplexing streams on the same QUIC connection, each RTP packet is prefixed with a flow
identifier. This flow identifier serves as a replacement for using different transport addresses per
stream. A flow identifier is a QUIC variable length integer which must be unique per stream.

RTP and RTCP packets of a single RTP session MAY be sent using the same flow identifier (following
the procedures defined in {{!RFC5761}}, or they MAY be sent using different flow identifiers.
The respective mode of operation MUST be indicated using the appropriate signaling, e.g.,
when using SDP as discussed in {{sdp}}.

RTP and RTCP packets of different RTP sessions MUST be sent using different flow identifiers.

Differentiating RTP/RTCP datagrams of different RTP sessions from non-RTP/RTCP datagrams is
the responsibility of the application by means of appropriate use of flow identifiers and
the corresponding signaling.

RTP senders SHOULD consider the header overhead associated with QUIC datagrams and ensure that
the RTP/RTCP packets including their payloads, QUIC, and IP headers will fit into path MTU.

# Protocol Operation {#protocol-operation}

This section describes how senders and receivers can exchange RTP packets using QUIC. While the
receiver side is very simple, the sender side has to keep track of sent packets and corresponding
acknowledgements to implement congestion control.

RTP/RTCP packets that are submitted to an RTP over QUIC implementation are buffered in a queue. The
congestion controller defines the rate at which the next packet is dequeued and sent over the QUIC
connection. Before a packet is sent, it is prefixed with the flow identifier described in
{{packet-format}} and encapsulated in a QUIC datagram.

The implementation has to keep track of sent packets in order to build the feedback for a congestion
controller described in {{cc-interface}}. Each sent packet is mapped to the datagram in which it was
sent over QUIC. When the QUIC implementation signals an acknowledgement for a specific datagram, the
packet that was sent in this datagram is marked as received. Together with the received mark, an
estimation of the delay at which the packet was received by the peer can be stored. Assuming the RTT
is divided equally between the link from the sender to the receiver and the link back to the sender,
this estimation can be calculated using the RTT exposed by QUIC divided by two. This mapping can
later be used to create the `pkt_status_list` and the `pkt_delay_list` as described in
{{cc-interface}}.

In a regular interval, the `pkt_status_list` and the `pkt_delay_list` MUST be passed to the
congestion controller together with the current timestamp `t_current` and the RTT statistics
`min_rtt`, `smoothed_rtt` and `rtt_var`. If available, the feedback MAY also contain the ECN marks.

The feedback report can be passed to the congestion controller at a frequency
specified by the used algorithm.

The congestion controller regularly outputs the `target_bitrate`, which is forwarded to the encoder
using the interface described in {{encoder-interface}}.

~~~
+------------+ Media  +-------------+
|  Encoder   |------->| Application |
|            |        |             |
+------------+        +-------------+
      ^                      |
      |                      |
      | Set             RTP/ |
      | Target          RTCP |
      | bitrate              |
      |                      v        Target
      |               +-------------+ Bitrate   +-------------+
      +---------------|  RTP over   |<----------| Congestion  |
                      |    QUIC     | Feedback  | Controller  |
                      +-------------+---------->+-------------+
                           |   ^
                   Flow ID |   | Datagram
                  prefixed |   | ACK/Lost-
                  RTP/RTCP |   | notification
                   packets |   | +
                           |   | RTT
                           v   |
                      +-------------+
                      |    QUIC     |
                      |             |
                      +-------------+
~~~
{: #fig-send-flow title="RTP over QUIC send flow"}

On receiving a datagram, an RTP over QUIC implementation strips off and parses the flow identifier to
identify the stream to which the received RTP packet belongs. The remaining content of the datagram
is then passed to the RTP session which was assigned the given flow identifier.

# SDP Signalling {#sdp}

QUIC is a connection-based protocol that supports connectionless transmissions of DATAGRAM frames
within an established connection.  As noted above, demultiplexing DATAGRAMS intended for different
purposes is up to the application using QUIC.

There are three necessary steps to carry out jointly between the
communicating peers to enable RTP over QUIC:

1. The peers need to decide whether to establish a new QUIC connection or whether to re-use an
   existing one.  In case of establishing a new connection, the initiator and the responder
   (client and server) need to be determined.  Signaling for this step MUST follow {{!RFC4572}}
   on SDP attributes for connection-oriented media.

2. The peers must provide a mapping of RTP sessions to flow identifiers, which is conceptually
   similar to signaling port numbers for demultiplexing.  To this end, all media specified in
   SDP to use the same QUIC connection MUST share a common c= line to signal the IP address
   and MUST all use the same port number -- that of the QUIC connection -- in the m= line.

   Flow identifiers MUST be treated independently for each direction of transmission, so that
   an endpoint MAY choose its own flow identifies and only uses SDP to inform its peer which
   RTP sessions use which flow identifiers.

   To this end, SDP MUST be used to indicate the repsective flow identifiers for RTP and RTCP
   of the different RTP sessions (for which we borrow inspiration from {{!RFC3605}}).

   **TODO:** check {{!RFC8122}} {{!RFC8843}}

3. The peers MUST agree, for each RTP session, where or not to apply RTP/RTCP multiplexing.
   If multiplexing RTP and RTCP shall take place on the same flow identifier, this MUST be
   indicated using the attribute a=rtcp-mux.

   If RTP/RTCP multiplexing is used, the flow identifiers for the RTP and the RTCP flows
   MUST be identical.


> **TODO:** Describe how SDP can be used to setup session, e.g. assign flow IDs.

> **TODO:** How to deal with different applications using Datagrams on the same QUIC connection?

# Used RTP/RTCP packet types

Any RTP packet can be sent over QUIC and no RTCP packets are used by default. Since QUIC already
includes some features which are usually implemented by certain RTCP messages, RTP over QUIC
implementations should not need to implement the following RTCP messages:

* PT=205, FMT=1, Name=Generic NACK: Provides Negative Acknowledgements {{!RFC4585}}. Acknowledgement
  and loss notifications are already provided by the QUIC connection.
* PT=205, FMT=8, Name=RTCP-ECN-FB: Provides RTCP ECN Feedback {{!RFC6679}}. If supported, ECN
  may directly be exposed by the used QUIC implementation.
* PT=205, FMT=11, Name=CCFB: RTP Congestion Control Feedback which contains receive marks,
  timestamps and ECN notifications for each received packet {{!RFC8888}}. This can be inferred from
  QUIC as described in {{protocol-operation}}.
* PT=210, FMT=all, Name=Token, {{!RFC6284}} specifies a way to dynamically assign ports for RTP
  receivers. Since QUIC connections manage ports on their own, this is not required for RTP over
  QUIC.

# Enhancements

The RTT statistics collected by QUIC may not be very precise because it can be influenced by delayed
ACKs. An alternative the RTT is to explicitly measure a one way delay. {{QUIC-TS}} suggests an
extension for QUIC to implement one way delay measurements using a timestamp carried in a special
QUIC frame. The new frame carries the time at which a packet was sent. This timestamp can be used by
the receiver to estimate a one way delay as the difference between the time at which a packet was
received and the timestamp in the received packet. The one way delay can then be used as a
replacement for the receive time estimation derived from the RTT as described in
{{protocol-operation}} to create the `pkt_delay_list`.

> **TODO:** Even with one-way delay measurements it is still not possible to identify exact
> timestamps for individual packets, since the timestamp may be sent with an ACK that acks more than
> one earlier packet.

# Discussion

## Impact of Connection Migration

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
