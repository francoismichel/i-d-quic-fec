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
  QUICv1: RFC9000
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

QUIC version 1 {{QUICv1}} does not retransmit neither frames nor packets
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
ID (SID). The SID of first source symbol MUST be zero and increase by
exactly one for every new source symbol. As the QUIC packet number cannot
be used to carry the SID, this information must be transmitted using either
a dedicated QUIC frame or a dedicated header field. The following sections
discuss the two alternatives.

#### Alternative 1: sending the SID inside a frame

This alternative is compatible with {{QUICv1}}. It defines a new SID frame
as shown in {{fig-sid-frame}}.

~~~~
SID {
  SID (i),
}
~~~~
{: #fig-sid-frame title="SID frame format"}

A QUIC packet carrying an SID frame means that the frames of that packet
are part of a FEC source symbol. The SID of this symbol is represented by
the only field of the SID frame. A packet cannot contain more than one SID
frame. A packet whose payload is FEC-protected MUST contain a SID frame
whose SID field is the SID of the related source symbol. This alternative
has one drawback: the SID frame is related to its containing packet.
The SID frame is thus not idempotent.

#### Alternative 2: sending the SID and the protected frames inside a frame

This alternative is compatible with {{QUICv1}}. It defines a new
SOURCE_SYMBOL frame as shown in {{fig-source-symbol}}.

~~~~
SOURCE_SYMBOL {
  SID (i),
  FEC Protected Payload (..)
}
~~~~
{: #fig-source-symbol title="SOURCE_SYMBOL frame format"}

The frame explicitly represents a SOURCE_SYMBOL. The FEC Protected Payload
field is analogous to the payload of a QUIC packet: it contains a
sequence of frames that are protected by FEC. The SOURCE_SYMBOL frame
contains frames just as QUIC packets do. The advantage of this approach
is that the SOURCE_SYMBOL frame is idempotent: it is not related to
its containing packet as it describes clearly the frames inside the source
symbol. The main drawback is that existing QUIC implementations are not
used to write frames inside other frames which may increase the
implementation cost of the approach.

#### Alternative 3 (thrash ?): sending the SID using a packet header field

This alternative is not compatible with {{QUICv1}} and requires a new
protocol version. It defines a new short header field that is part
of the 1-RTT protected payload. The 1-RTT packet payload is redefined
as shown in {{fig-fec-packet-payload}}.

~~~~
Packet Payload {
  SID (i),
  Frame (8..) ...,
}
~~~~
{: #fig-fec-packet-payload title="1-RTT protected payload"}

The SID field contains the SID of the source symbol contained in the
packet.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
