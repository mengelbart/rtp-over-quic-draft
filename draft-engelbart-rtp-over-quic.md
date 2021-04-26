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

informative:



--- abstract

TODO Abstract

--- middle

# Introduction

TODO Introduction

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
Network-Assisted Dynamic Adaptation (NADA) {{!RFC8698}} and Self- Clocked Rate Adaptation for
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


## QUIC Interface

## Congestion Controller Interface {#cc-interface}


## Codec Interface {#encoder-interface}


# Packet Format {#packet-format}


## Encoding: Binary representation

## "How to transport / encapsulate"

# Protocol Operation

This section describes how senders and receivers can exchange RTP packets using QUIC. While the
receiver side is very simple, the sender side has to keep track of sent packets and corresponding
acknowledgement to implement congestion control.

RTP/RTCP packets that are submitted to an RTP over QUIC implementation are buffered in a queue. The
congestion controller defines the rate at which the next packet is dequeued and sent over the QUIC
connection. Before a packet is sent, it is prefixed with the flow identifier described in
{{packet-format}} and encapsulated in a QUIC datagram.

The implementation has to keep track of sent packets in order to build the feedback for a congestion
controller described in {{cc-interface}}. Each sent packet is mapped to the datagram in which it was
sent over QUIC and when the QUIC implementation signals an acknowledgement for a specific datagram,
the packet that was sent in this datagram is marked as received. Together with the received mark, an
estimation of the delay at which the packet was received by the peer is stored. This estimation can
be calculated from the RTT exposed by QUIC.

> **TODO:** RTT may be better replaced by an explicit timestamp

The list of received packets and delays can be passed to the congestion controller at a frequency
specified by the used algorithm.

The congestion controller regularly outputs a target bitrate, which is forwarded to the encoder
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

On receiving a datagram, a RTP over QUIC implementation strips off and parses the flow identifier to
identify the stream to which the received RTP packet belongs. The remaining content of the datagram
is then passed to the RTP session which was assigned the given flow identifier.

> **TODO:** Decide what to do whith unknown flow IDs and streams that do not belong to RTP
> over QUIC in the case where an application sends other data apart from RTP in datagrams over the
> same QUIC connection. Some options are:

> * If no RTP session with the flow identifier was registered, the packet is dropped.
> * An error can be signaled to the application.
> * The content can be forwarded to the application without being associated to a RTP stream. (This
>   would probably require a way to assign flow IDs more gerenally than per RTP stream)

# Enhancements

## Timestamp Draft

## Delayed ACKs

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
