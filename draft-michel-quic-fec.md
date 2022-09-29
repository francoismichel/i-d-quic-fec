---
title: "Forward Erasure Correction for QUIC loss recovery"
abbrev: "FEC for QUIC"
category: info

docname: draft-michel-quic-fec-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Transport"
workgroup: "QUIC"
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: "QUIC"
  type: "Working Group"
  mail: "quic@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/quic/"
  github: "francoismichel/i-d-quic-fec"
  latest: "https://francoismichel.github.io/i-d-quic-fec/draft-michel-quic-fec.html"

author:
 -
    fullname: François Michel
    organization: UCLouvain
    email: "francois.michel@uclouvain.be"

normative:
  QUIC: RFC9000
  QUIC-RECOVERY: RFC9002

informative:
  QUIC-FEC: DOI.10.23919/IFIPNetworking.2019.8816838
  FlEC: DOI.10.1109/TNET.2022.3195611
  I-D.swett-nwcrg-coding-for-quic:



--- abstract

This documents lays down the QUIC protocol design considerations
needed for QUIC to apply Forward Erasure Correction on the data sent
through the network.



--- middle

# Introduction

QUIC version 1 {{QUIC}} does not retransmit neither frames nor packets
upon network losses. Instead, the lost information is sent again in
new frames carried by new packets. Retransmitting the lost information
requires the loss recovery mechanism to identify lost packets which
may take up to several hundreds of milliseconds {{QUIC-RECOVERY}}.
Depending on their delay-sensitivity, some applications running QUIC
could not afford such a waiting time to ensure a good quality of
experience to their users.

Several works have already been performed to consider the use of
Forward Erasure Correction for the QUIC protocol to ensure timely
data delivery for delay-sensitive applications {{QUIC-FEC}} {{FlEC}}
{{I-D.swett-nwcrg-coding-for-quic}}.
This documents lists the required additions to the QUIC protocol to
extend its loss recovery mechanism and make it able to recover from
packet losses prior to loss detection.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# The network channel VS the coding channel

TODO

# Sending redundancy with the QUIC protocol

Forward Erasure Correction is not the only possible mechanism to send
redundant information with QUIC: QUIC already sends acknowledgements
redundantly with cumulative acknowledgements and ACK blocks repeated
until they are acked by the peer. If an ACK frame is lost, the
acknowledged packets can be retrieved. On the other hand,
repeating the payload of STREAM or DATAGRAM frames will have a large
negative impact on bandwidth. FEC and network coding allow sending
more efficiently redundancy to protect data from losses.

## Example {-}

A simple and suboptimal FEC technique can be implemented using a XOR
operation between all the informations chunks whose retransmission
should be avoided. Let P1, P2 and P3 be three pieces of information of
the same sizesent on the wire. The repair symbol R can be defined like
the following :

    R = P1 XOR P2 XOR P3

Such that the loss of any of P1, P2 or P3 can be recovered by XORing
the two remaining pieces of information with R.

## Protocol requirements for protecting information through FEC

In this section, we list the points that must be defined by the protocol
for allowing QUIC endpoints to protect information using FEC in a more
efficient way than duplicating the information sent on the wire.

### Defining the content of source symbols

There is no need to protect every piece information sent on the wire by
QUIC. Some pieces of information are already sent redundantly (e.g. ACKs)
and some data are not delay sensitive and can be retransmitted later with
no harm (e.g. background download on a separate stream). When exchanging
packets, endpoints need to agree on which parts of the packets are part of
the protected source symbols and how to build a source symbol from what is
sent on the wire.

The versions 01 and 02 of {{I-D.swett-nwcrg-coding-for-quic}} only protect
streams payload. The idea is simple when using a single stream but becomes
complicated and requires more signaling in a multi-stream scenario. It also
cannot handle the protection of DATAGRAM frames payload.
In this document, we propose to consider whole frames as part of the source
symbols. Source symbols are thus the equivalent to QUIC packets for the
coding channel: packets carry frames through the network channel while
source symbols carry frames through the coding channel. In order to reduce
signalling between the peers, the design described in this document requires
that a single source symbol MUST NOT contain the frames of several QUIC
packets at the same time.

### Identifying the source symbols

In order to recover lost source symbols, the decoder needs to know how many
and which source symbols were lost. From the receiver viewpoint, it is not
possible to distinguish a lost packet from a packet that has never been
sent as QUIC does not enforce the sending of packets with a contiguously
increasing packet number. Furthermore a QUIC sender may not want to protect
the payload of some packets if they do not carry latency-sensitive
information. It is thus required to uniquely identify the source symbols
so that the decoder can point the lost source symbol by looking at the
received source symbols only. Source symbols are thus attributed a Symbol
ID (SID). As the QUIC packet number cannot be used to carry the SID, this
information must be transmitted using either a dedicated QUIC frame or a
dedicated header field. The following sections discuss the two alternatives.

#### Alternative 1: sending the SID inside a frame

TODO

#### Alternative 2: sending the SID using a packet header field





# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
