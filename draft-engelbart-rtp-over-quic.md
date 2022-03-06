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
    email: mathis.engelbart@gmail.com

normative:
  draft-ietf-quic-datagram-10:
    title: "An Unreliable Datagram Extension to QUIC"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-datagram-10
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

  draft-huitema-quic-ts-05:
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

  draft-schinazi-quic-h3-datagram-05:
    title: "Using QUIC Datagrams with HTTP/3"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-schinazi-quic-h3-datagram-05
    author:
      -
        ins: D. Schinazi
        name: David Schinazi
        org: Google LLC
        role: editor

  draft-smith-quic-receive-ts-00:
    title: "QUIC Extension for Reporting Packet Receive Timestamps"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-smith-quic-receive-ts-00
    author:
      -
        ins: C. Smith
        name: Connor Smith
        org: Magic Leap, Inc.
        role: editor
      -
        ins: I. Swett
        name: Ian Swett
        org: Google LLC
        role: editor

  draft-ietf-quic-ack-frequency-01:
    title: "QUIC Acknowledgement Frequency"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-ack-frequency-01
    author:
      -
        ins: J. Iyengar
        name: Jana Iyengar
        org: Fastly
        role: editor
      -
        ins: I. Swett
        name: Ian Swett
        org: Google
        role: editor

informative:
  draft-hurst-quic-rtp-tunnelling-01:
    title: "QRT: QUIC RTP Tunnelling"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-hurst-quic-rtp-tunnelling-01
    author:
      -
        ins: S. Hurst
        name: Sam Hurst
        org: BBC Research & Development
        role: editor

  draft-dawkins-avtcore-sdp-rtp-quic:
    title: "SDP Offer/Answer for RTP using QUIC as Transport"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-dawkins-avtcore-sdp-rtp-quic-00
    author:
      -
        ins: 
        name: Spencer Dawkins
        org: Tencent America LLC
        role: editor

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
the media exchange and occasionally TCP (and possibly TLS) as
a fallback.  With the advent of QUIC and, most notably, its unreliable
DATAGRAM extension, another secure transport protocol becomes
available.  QUIC and its DATAGRAMs combine desirable properties for
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
Protocol (SDP) {{!RFC8866}}.

The scope of this document is limited to unicast RTP/RTCP.

Note that this draft is similar in spirit to but differs in numerous ways from
{{draft-hurst-quic-rtp-tunnelling-01}}.


# Terminology and Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

The following terms are used:

Congestion Controller:
: QUIC specifies a congestion controller in {{Section 7 of !RFC9002}}, but the
specific requirements for interactive real-time media lead to the development of
dedicated congestion control algorithms. In this document, the term congestion
controller refers to these algorithms dedicated to real-time applications.

Datagram:
: Datagrams exist in UDP as well as in QUICs unreliable datagram extension. If not explicitly noted
differently, the term datagram in this document refers to a QUIC Datagram as defined in
{{draft-ietf-quic-datagram-10}}.

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

# Protocol Overview

This document introduces a mapping of the Real-time Transport Protocol (RTP) to
the QUIC transport protocol. QUIC supports two transport methods: reliable
streams and unreliable datagrams {{!RFC9000}}, {{draft-ietf-quic-datagram-10}}.
RTP over QUIC uses unreliable QUIC datagrams to transport real-time data, and
thus, the QUIC implementation MUST support QUICs unreliable datagram extension.
Since datagram frames cannot be fragmented, the QUIC implementation MUST also
provide a way to query the maximum datagram size so that an application can
create RTP packets that always fit into a QUIC datagram frame.

{{!RFC3550}} specifies that RTP sessions need to be transmitted on different
transport addresses to allow multiplexing between them. RTP over QUIC uses a
different approach to leverage the advantages of QUIC connections without
managing a separate QUIC connection per RTP session. QUIC does not provide
demultiplexing between different flows on datagrams but suggests that an
application implement a demultiplexing mechanism if required. An example of such
a mechanism are flow identifiers prepended to each datagram frame as described
in {{draft-schinazi-quic-h3-datagram-05}}. RTP over QUIC uses a flow identifier
to replace the network address and port number to multiplex many RTP sessions
over the same QUIC connection.

A congestion controller can be plugged in to adapt the media bitrate to the
available bandwidth. This document does not mandate any congestion control
algorithm. Some examples include Network-Assisted Dynamic Adaptation (NADA)
{{!RFC8698}} and Self-Clocked Rate Adaptation for Multimedia (SCReAM)
{{!RFC8298}}. These congestion control algorithms require some feedback about
the network's performance to calculate target bitrates. Traditionally this
feedback is generated at the receiver and sent back to the sender via RTCP.
Since QUIC also collects some metrics about the network's performance, these
metrics can be used to generate the required feedback at the sender-side and
provide it to the congestion controller to avoid the additional overhead of the
RTCP stream.

# Packet Format {#packet-format}

All RTP and RTCP packets MUST be sent in QUIC datagram frames with the following format:

~~~
Datagram Payload {
  Flow Identifier (i),
  RTP/RTCP Packet (..)
}
~~~
{: #fig-datagram-payload title="Datagram Payload Format"}

Flow Identifier:

: Flow identifier to demultiplex different data flows on the same QUIC
connection.

RTP/RTCP Packet:

: The RTP/RTCP packet to transmit.

For multiplexing RTP sessions on the same QUIC connection, each RTP/RTCP packet
is prefixed with a flow identifier. This flow identifier serves as a replacement
for using different transport addresses per session. A flow identifier is a QUIC
variable-length integer which must be unique per stream.

RTP and RTCP packets of a single RTP session MAY be sent using the same flow
identifier (following the procedures defined in {{!RFC5761}}, or they MAY be
sent using different flow identifiers. The respective mode of operation MUST be
indicated using the appropriate signaling, e.g., when using SDP as discussed in
{{sdp}}.

RTP and RTCP packets of different RTP sessions MUST be sent using different flow
identifiers.

Differentiating RTP/RTCP datagrams of different RTP sessions from non-RTP/RTCP
datagrams is the responsibility of the application by means of appropriate use
of flow identifiers and the corresponding signaling.

Senders SHOULD consider the header overhead associated with QUIC datagrams and
ensure that the RTP/RTCP packets, including their payloads, QUIC, and IP
headers, will fit into path MTU.

# Congestion Control {#congestion-control}

RTP over QUIC needs to employ congestion control to avoid overloading the
network. RTP and QUIC both offer different congestion control mechanisms. QUIC
specifies a congestion control algorithm similar to TCP NewReno, but allows
senders to choose a different algorithm, as long as the algorithm conforms to
the guidelines specified in {{Section 3 of !RFC8085}}. RTP does not specify a
congestion controller, but provides feedback formats for congestion control
(e.g. {{!RFC8888}}) as well as different congestion control algorithms in
separate RFCs (e.g. {{!RFC8298}} and {{!RFC8698}}). The congestion control
algorithms for RTP are specifically tailored for real-time transmissions at low
latencies. RTP congestion control mostly works delay-based, using the growing
one-way delay as a congestion signal. The available congestion control
algorithms for RTP also expose a `target_bitrate` that can be used to
dynamically reconfigure media encoders to produce media at a rate that can be
sent in real-time under the given network conditions.

This section defines two options for congestion control for RTP over QUIC, but
it does not mandate which congestion control algorithms to use. The congestion
control algorithm MUST expose a `target_bitrate` to which the encoder should be
configured to fully utilize the available bandwidth. Furthermore, it is assumed
that the congestion controller provides a pacing mechanism to determine when a
packet can be sent to avoid bursts. The currently proposed congestion control
algorithms for real-time communications provide such a pacing mechanism. The use
of congestion controllers which don't provide a pacing mechanism is out of scope
of this document.

Additionally, the section defines how the connection statistics obtained from
QUIC can be used to reduce RTCP feedback overhead.

## RTCP and QUIC Connection Statistics {#rtcp-and-quic-stats}

Since QUIC provides generic congestion signals which allow the implementation of
different congestion control algorithms, senders are not dependent on RTCP
feedback for congestion control. However, there are some restrictions, and the
QUIC implementation MUST fulfill some requirements to use these signals for
congestion control instead of RTCP feedback.

To estimate the currently available bandwidth, real-time congestion control
algorithms keep track of the sent packets and typically require a list of
successfully delivered packets together with the timestamps at which they were
received by a receiver. The bandwidth estimation can then be used to decide
whether the media encoder can be configured to produce output at a higher or
lower rate.

A congestion controller used for RTP over QUIC should be able to compute an
adequate bandwidth estimation using the following inputs:

* `t_current`: A current timestamp
* `pkt_departure`: The departure time for each RTP packet sent to the receiver.
* `pkt_arrival`: The arrival time for each RTP packet that was successfully
  delivered to the receiver.
* The RTT estimations calculated by QUIC as described in {{Section 5 of RFC9002}}:
  * `latest_rtt`: The latest RTT sample generated by QUIC.
  * `min_rtt`: The miminum RTT observed by QUIC over a period of time
  * `smoothed_rtt`: An exponentially-weighted moving average of the observed RTT
    values
  * `rtt_var`: The mean deviation in the observed RTT values
* `ecn`: Optionally ECN marks may be used, if supported by the network and
  exposed by the QUIC implementation.

The only value of these inputs not currently available in QUIC is the
`pkt_arrival`. The exact arrival times of QUIC Datagrams can be obtained by
using the QUIC extension described in {{draft-smith-quic-receive-ts-00}} or
{{draft-huitema-quic-ts-05}}.

QUIC allows acknowledgments to be sent with some delay, which could cause
problems for delay-based congestion control algorithms. Sender and receiver can
use {{draft-ietf-quic-ack-frequency-01}} to avoid feedback inaccuracies caused
by delayed acknowledgments.

If the QUIC extensions described in
{{draft-smith-quic-receive-ts-00}}/{{draft-huitema-quic-ts-05}} and
{{draft-ietf-quic-ack-frequency-01}} are not supported by sender and receiver,
it is RECOMMENDED to use RTCP feedback reports instead of thee QUIC connection
statistics for congestion control.

## RTP Congestion Control at the QUIC layer {#cc-quic-layer}

The first option implements congestion control at the QUIC layer by replacing
the standard QUIC congestion control with one of the congestion control
algorithms for RTP.

> **Editor's note:** How can a QUIC connection be shared with non-RTP streams,
> when SCReAM/NADA/GCC is used as congestion controller? Can these algorithms be
> adapted to allow different streams including non-real-time streams?

> **Editor's note:** If this option is chosen, but the required QUIC
> statistics/extensions are not available and the sender has to use RTCP
> feedback for congestion control, the feedback needs to be fed back to the QUIC
> implementation.

## RTP Congestion Control at the Application Layer {#cc-application-layer}

The second option implements real-time congestion control at the application
layer. This gives an application more control over the congestion controller and
the congestion control feedback to use. It is RECOMMENDED to disable QUIC's
congestion control when this option is used to avoid interferences between the
congestion controllers at different layers.

> **Editor's note:** Maybe this option should be removed, as it has some issues:
> 1. It cannot be used in situations where the application is untrusted, such as
>    in Webtransport, where the browser implements QUIC but cannot trust a JS
>    application using it to do the right thing.
> 2. It is unclear how non-real-time data sharing the same connection can be
>    congestion controlled.
> 3. If QUIC connection statistics should be used instead of RTCP, these have to
>    be exposed to the application.

## Bandwidth Allocation {#bandwidth-allocation}

When the QUIC connection is shared between multiple data streams, a share of the
available bandwidth should be allocated to each stream. An implementation MUST
ensure that a real-time flow is always allowed to send data unless it has
exhausted its allocated bandwidth share. This is especially important when the
connection is shared with non-real-time flows.

> **Editor's note:** This section may need to explain the problem that occurs
> when non-real-time data fills up the congestion window when a real-time flow
> does not fully use its assigned bandwidth share.

## Media Rate Control {#media-rate-control}

Independent from which option is chosen to implement congestion control, the
sender likely needs to reconfigure the media encoder in reaction to changing
network conditions. Common real-time congestion control algorithms expose a
`target_bitrate` for this purpose. An RTP over QUIC implementation can either
expose the most recent `target_bitrate` produced by the congestion controller to
the application or accept a callback from the application, which updates the
encoder bitrate whenever the congestion controller updates the `target_bitrate`.

# SDP Signalling {#sdp}

> **Editor's note:** See also {{draft-dawkins-avtcore-sdp-rtp-quic}}.

QUIC is a connection-based protocol that supports connectionless transmissions of DATAGRAM frames
within an established connection.  As noted above, demultiplexing DATAGRAMS intended for different
purposes is up to the application using QUIC.

There are several necessary steps to carry out jointly between the
communicating peers to enable RTP over QUIC:

0. The protocol identifier for the m= lines MUST be "QUIC/RTP", combined as per {{!RFC8866}}
   with the respective audiovisual profile: for example, "QUIC/RTP/AVP".

1. The peers need to decide whether to establish a new QUIC connection or whether to re-use an
   existing one.  In case of establishing a new connection, the initiator and the responder
   (client and server) need to be determined.  Signaling for this step MUST follow {{!RFC8122}}
   on SDP attributes for connection-oriented media for the a=setup, a=connection, and
   a=fingerprint attributes.  They MUST use the appropriate protocol identification as per 1.

2. The peers must provide a means for identifying RTP sessions carried in QUIC DATAGRAMS.
   To enable using a common transport connection for one, two, or more media sessions in
   the first place, the BUNDLE grouping framework MUST be used {{!RFC8843}}.  All media sections
   belonging to a bundle group, except the first one, MUST set the port in the m= line to zero
   and MUST include the a=bundle-only attribute.

   For disambiguating different RTP session, a reference needs to be provided for each m= line to
   allow associating this specific media session with a flow identifier.  This could be
   achieved following different approaches:

   * Simply reusing the a=extmap attribute {{!RFC8285}} and relying on RTP header extensions
     for demultiplexing different media packets carried in QUIC DATAGRAM frames.

   * Defining a variant or different flavor of the a=extmap attribute {{!RFC8285}} that binds
     media sessions to flow identifiers used in QUIC DATAGRAMS.

   > **Editor's note:** It is likely preferable to use multiplexing using QUIC DATAGRAM flow
   identifiers because this multiplexing mechanisms will also work across RTP and non-RTP media
   streams.

   In either case, the corresponding identifiers MUST be treated independently for each
   direction of transmission, so that an endpoint MAY choose its own identifies and only
   uses SDP to inform its peer which RTP sessions use which identifiers.

   To this end, SDP MUST be used to indicate the respective flow identifiers for RTP and RTCP
   of the different RTP sessions (for which we borrow inspiration from {{!RFC3605}}).

3. The peers MUST agree, for each RTP session, whether or not to apply RTP/RTCP multiplexing.
   If multiplexing RTP and RTCP shall take place on the same flow identifier, this MUST be
   indicated using the attribute a=rtcp-mux.

A sample session setup offer (liberally borrowed and extended from {{!RFC8843}} and {{!RFC8122}}
could look as follows:

~~~
v=0
o=alice 2890844526 2890844526 IN IP6 2001:db8::3
s=
c=IN IP6 2001:db8::3
t=0 0
a=group:BUNDLE abc xyz

m=audio 10000 QUIC/RTP/AVP 0 8 97
a=setup:actpass
a=connection:new
a=fingerprint:SHA-256 \
 12:DF:3E:5D:49:6B:19:E5:7C:AB:4A:AD:B9:B1:3F:82:18:3B:54:02:12:DF: \
 3E:5D:49:6B:19:E5:7C:AB:4A:AD
b=AS:200
a=mid:abc
a=rtcp-mux
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:97 iLBC/8000
a=extmap:1 urn:ietf:params:<tbd>

m=video 0 QUIC/RTP/AVP 31 32
b=AS:1000
a=bundle-only
a=mid:bar
a=rtcp-mux
a=rtpmap:31 H261/90000
a=rtpmap:32 MPV/90000
a=extmap:2 urn:ietf:params:<tbd>
~~~
{: #sdp-example title="SDP Offer"}

Signaling details to be worked out.


# Used RTP/RTCP packet types

Any RTP packet can be sent over QUIC and no RTCP packets are used by default. Since QUIC already
includes some features which are usually implemented by certain RTCP messages, RTP over QUIC
implementations should not need to implement the following RTCP messages:

* PT=205, FMT=1, Name=Generic NACK: Provides Negative Acknowledgments {{!RFC4585}}. Acknowledgment
  and loss notifications are already provided by the QUIC connection.
* PT=205, FMT=8, Name=RTCP-ECN-FB: Provides RTCP ECN Feedback {{!RFC6679}}. If supported, ECN
  may directly be exposed by the used QUIC implementation.
* PT=205, FMT=11, Name=CCFB: RTP Congestion Control Feedback which contains receive marks,
  timestamps and ECN notifications for each received packet {{!RFC8888}}. This can be inferred from
  QUIC as described in {{rtcp-and-quic-stats}}.
* PT=210, FMT=all, Name=Token, {{!RFC6284}} specifies a way to dynamically assign ports for RTP
  receivers. Since QUIC connections manage ports on their own, this is not required for RTP over
  QUIC.

# Discussion

## Impact of Connection Migration

# Security Considerations

TBD

# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
