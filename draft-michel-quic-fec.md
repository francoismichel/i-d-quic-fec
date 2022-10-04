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
  RFC9265:

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

Works has already been done to consider the use of
Forward Erasure Correction for the QUIC protocol to ensure timely
data delivery for delay-sensitive applications {{QUIC-FEC}} {{FlEC}}
{{I-D.swett-nwcrg-coding-for-quic}}.
This documents lists the required additions to the QUIC protocol to
extend its loss recovery mechanism and make it able to recover from
packet losses prior to loss detection.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

## FEC-related definitions

Source symbol: piece of information exchanged by two endpoints.

Repair symbol: redundant information constructed from the combination
or several source symbols.

Erasure: loss of one or more entire symbols

Erasure correction code: algorithm combining one or several source
symbols to produce repair symbols and reconstructing source symbols
from a given set of source and repair symbols.

Forward Erasure Correction: process of recovering erased symbols prior
their erasure has been identified.

FEC scheme: the conjunction of an erasure correction code and its
specific protocol elements required to use this erasure correction code
within the design described in this document.

Encoder: entity producing repair symbols using an error correction code.
The encoder can be a library used by the protocol implementation or a
program running in a separate process or machine.

Decoder: entity reconstructing missing source symbols from received source
and repair symbols.
The decoder can be a library used by the protocol implementation or a
program running in a separate process or machine.


# The network channel and the coding channel

Regular QUIC endpoints exchange information over a network channel. Adding
Forward Erasure Correction to QUIC enables an enpoint to receive information
over a coding channel.  A coding channel can be seen as a communication
channel between a QUIC endpoint and a FEC decoder. The decoder often
runs on the same machine as the QUIC receiver and can even be part of the
protocol implementation itself. A source symbol is received through the coding
channel when it is recovered by the FEC decoder instead of being explicitly
received through the network. Only the data sent on the network channel are
really transmitted on the wire. {{fig-network-and-coding-channels}}
illustrates how symbols can be received from both channels.

~~~~

            Network Channel              Coding Channel

Sender                         Receiver               Decoder
  |                                |                    |
  |  PACKET(1)[SOURCE_SYMBOL(1)]   |                    |
  |------------------------------->|  SOURCE_SYMBOL(1)  |
  |                                |------------------->|
  |  PACKET(2)[SOURCE_SYMBOL(2)]   |                    |
  |--------------x                 |                    |
  |                                |                    |
  |  PACKET(3)[REPAIR_SYMBOL]      |                    |
  |------------------------------->|    REPAIR_SYMBOL   |
  |                                |------------------->|
  |                                |                    |(recomputes)
  |                                |                    |(the source)
  |                                |                    |(symbol    )
  |                                |  SOURCE_SYMBOL(2)  |
  |                                |<-------------------|
  |                                |                    |
  |                                |                    |
~~~~
{: #fig-network-and-coding-channels title="Receiving symbols through
the network channel"}


In this illustration, the sender sends three QUIC packets through the
network channel. Packets 1 and 2 carry one source symbol each
and the packet 3 contains carries one repair symbol protecting the
two source symbols. Packet 2 is lost due to network imperfection
preventing Source Symbol 2 from being received through the network
channel. The FEC decoder reconstructs Source Symbol 2 by combining
Source Symbol 1 and Repair Symbol. Source Symbol 2 is thus received
through the coding channel. Note that Source Symbol 2 is not received
as a packet since QUIC packets are only exchanged through the network
channel. On the other hand, source symbols can be received from both
the network and coding channels.


# FEC and the loss recovery mechanism

The FEC mechanism described in this document is additional to the
classical QUIC loss recovery mechanism {{QUIC-RECOVERY}} and does
not replace it by any means. A QUIC endpoint MAY ignore every received
repair symbol and MAY not perform any symbol recovery at all. The FEC
mechanism is only intended to allow a receiver recovering faster from
packet losses on the network channel if it values the timeliness of
data delivery.


# Protocol requirements for protecting information through FEC

In this section, we list the points that must be defined by the protocol
for allowing QUIC endpoints to protect information using FEC in a more
efficient way than duplicating the information sent on the wire.


## Defining the content of source symbols

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
symbols. Source symbols are thus the counterpart to QUIC packets for the
coding channel: packets carry frames through the network channel while
source symbols carry frames through the coding channel. In order to reduce
signalling between the peers, a single source symbol MUST NOT contain the
frames of several QUIC packets at the same time.


## Identifying the source symbols

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
a dedicated QUIC frame or a dedicated header field. The second solution being
incompatible with {{QUICv1}}, it is not discussed in this document.
The source symbols transmitted through the network channel are carried by
QUIC packets. The source symbol payload can either be put inside
a dedicated frame ({{sec-source-symbol-frame}}) or infered when handling
a specific frame ({{sec-sid-frame}}). Both alternatives lead to the exact
same packet wire format and outcome.


### Alternative 1: sending the source symbol inside a frame
{: #sec-source-symbol-frame}

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
implementation cost of the approach. An example of the use of the
SOURCE_SYMBOL frame is shown in {{fig-sid-frame-example}}.


~~~~

Sender                                                       Receiver
  |                                                             |
  | Pkt(5)[ACK[..], SOURCE_SYMBOL(0, { STREAM(4, "abc") }]      |
  |------------------------------------------------------------>|
  |                                                             |
  | Pkt(6)[STREAM(2, "xyz"),                                    |
  |        SOURCE_SYMBOL(1, { STREAM(8, "def"),                 |
  |                           DATAGRAM("msg") }]                |
  |------------------------------------------------------------>|
  |                                                             |
  |                                            Pkt(1)[ACK[..]]  |
  |<------------------------------------------------------------|
  |                                                             |

~~~~
{: #fig-source-symbol-frame-example title="SID Alternative 1"}

The two SOURCE_SYMBOL frames contain describe the source symbols with
SID 0 and 1. The first source symbol contains a frame for stream 4 while
the second one contains a frame for stream 8 and a DATAGRAM frame. The ACK
frame of packet 5 and the STREAM frame for stream 2 of packet 6 are not part
of the source symbol and are thus not protected by FEC. In this scenario,
every source symbol is correctly received and their reception can be deduced
from the acknowledgements of packets 5 and 6.


### Alternative 2: only sending the SID inside a frame
{: #sec-sid-frame}

This alternative is compatible with {{QUICv1}}. It defines a new SID frame
as shown in {{fig-sid-frame}}.

~~~~
SID {
  SID (i),
}
~~~~
{: #fig-sid-frame title="SID frame format"}

A QUIC packet carrying an SID frame means that the frames following the SID
frames in the packet payload are part of a FEC source symbol. The SID of
this symbol is represented by the only field of the SID frame. A packet
cannot contain more than one SID frame. A packet whose payload is
FEC-protected MUST contain a SID frame whose SID field is the SID of
the related source symbol. This alternative has one drawback: the SID frame
is related to its containing packet. The SID frame is thus not idempotent.
An example of the use of the SID frame is shown in {{fig-sid-frame-example}}.


~~~~

Sender                                       Receiver
  |                                            |
  | Pkt(5)[ACK[..], SID(0), STREAM(4, "abc")]  |
  |------------------------------------------->|
  |                                            |
  | Pkt(6)[STREAM(2, "xyz"),                   |
  |        SID(1), STREAM(8, "def"),           |
  |        DATAGRAM("msg")]                    |
  |------------------------------------------->|
  |                                            |
  |                            Pkt(1)[ACK[..]] |
  |<-------------------------------------------|
  |                                            |

~~~~
{: #fig-sid-frame-example title="SID Alternative 2"}

The source symbols carried by these packets are the same as for
{{fig-source-symbol-frame-example}} of Alternative 1 and the outcome and wire
format are the same.


## Sending the repair symbols
{: #sec-repair-frame}

The REPAIR symbols and the metadata attached to it are transferred using
the REPAIR frame shown in {{fig-fec-repair-frame}}.

~~~~
REPAIR {
  FEC Scheme Specific Repair Payload (1..),
}
~~~~
{: #fig-fec-repair-frame title="REPAIR frame format"}

The payload of the REPAIR frame is specific to the underlying
erasure-correcting code. In addition to the repair symbol itself, it may
contain any metadata needed by the erasure-correcting code (e.g. identifying
the source symbols protected by the repair symbol carried in the frame).
Depending on the underlying FEC scheme, the REPAIR frame MAY contain
only a part of a repair symbol.


## Announcing the coding window size
{: #sec-fec-window-frame}

The receiver needs to store the received symbols in order to recover the
lost source symbols. The FEC_WINDOW frame is sent by the symbols receiver
to announce the number of symbols  that can be stored simultaneously by
the receiver at a given point of time. The format of the FEC_WINDOW frame
is described in {{fig-fec-window-frame}}.

~~~~
FEC_WINDOW {
  Window Epoch (i),
  Window Size (i),
}
~~~~
{: #fig-fec-window-frame title="FEC_WINDOW frame format"}

The Window Epoch field is a unique identifier increasing by exactly one
for each new FEC_WINDOW frame sent by the FEC receiver.

The Window Size field indicates the number of symbols that can be stored
simultaneously by the receiver. The Window Size value overrides the
the window sizes received for smaller window epochs.


## Announcing the recovered symbols
{: #sec-symbol-ack-frame}

The FEC receiver MAY advertise the source symbols that have been received
either through the network or using FEC to avoid the sender retransmitting
the data of the recovered source symbols. This can be done using the
SYMBOL_ACK frame as shown in {{fig-symbol-ack-frame}}.

~~~~
SYMBOL_ACK {
  Largest Acknowledged (i),
  ACK Range Count (i),
  First ACK Range (i),
  ACK Range (..) ...,
}
~~~~
{: #fig-symbol-ack-frame title="SYMBOL_ACK frame format"}

The frame has a similar format to the ACK frame. In addition to symbols
recovered by FEC, this frame MAY also announce symbols received regularly
through the network to avoid gaps in the ACK ranges and reduce the frame
size. There is no obligation for a FEC receiver to send SYMBOL_ACK frames.
The FEC receiver MAY decide to only advertize a subset of the received
source symbols. The SYMBOL_ACK frame MUST NOT be used to infer any congestion
state on the network (see {{sec-coding-and-congestion}}).


# Coding channel and congestion control
{: #sec-coding-and-congestion}

The coding and network channels being unrelated, the fact of receiving symbols
through the coding channel MUST NOT be used to infer any congestion state on
the network channel. Especially, receiving a symbol through the coding channel
MUST NOT be used to hide the network loss event of the corresponding packet to
the congestion control, applying Recommandation 1 of {{RFC9265}}.


# Negociating the FEC extension using transport parameters

This section defines the new transport parameters used to negociate and
parametrize the FEC extension described in this document.


## enable_fec (0xfec)
{: #sec-enable-fec-tp}

The use of the FEC extension is negociated using the enable_fec transport
parameter defined in {{enable-fec-transport-parameter}} :

Option | Definition
---------|---------------------------------------
0x0      | don't support FEC
0x1      | supports FEC as defined in this document
{: #enable-fec-transport-parameter title="Values for enable_fec"}

When the enable_fec value is 0 or is not advertized by the peer,
the QUIC endpoint MUST NOT use any frame or mechanism described in this
document.


## decoder_fec_scheme (0xfecd)
{: #sec-decoder-fec-scheme-tp}

Each QUIC endpoint uses the decoder_fec_scheme transport parameter to
define the FEC scheme used to decode the received repair symbols. The
QUIC sender MUST use the specified FEC scheme to generate repair symbols.
The decoder_fec_scheme parameter is an integer value representing is the
identifier of the desired FEC scheme.
For instance, a FEC scheme using reed solomon could be identified by the
ID 0x0 and a FEC scheme using LDPC could be identified by 0x1.

This document does not specify nor identify any FEC scheme yet.
When the decoder_fec_scheme parameter is not advertized by the peer,
the QUIC sender MUST NOT send any repair symbol.


## initial_coding_window (0xfecc)
{: #sec-initial-coding-window-tp}

Each QUIC endpoint uses the initial_coding_window transport parameter to
define the initial coding window size it uses to store source and repair
symbols (see {{sec-fec-window-frame}}).
When the initial_coding_window parameter is not advertized by the peer,
the QUIC sender MUST consider a default value of 0 and MUST NOT send
any repair symbol.

# Security Considerations

The FEC mechanism for QUIC only runs under 0-RTT and 1-RTT encryption levels and
only operates inside the encrypted payload.

## DoS due to difficult symbols recoveries

An attacker could try to cuase a DoS of a receiver by selectively sending source
and repair symbols to trigger intensive erasure correction operations on the
receiver. A QUIC receiver is never forced to perform any erasure correction
and may ignore any received repair symbol if it has doubts in its capabilities
to decode it in a reasonable amount of time.


# IANA Considerations

This document defines three new transport parameters and five new frames. The
SID and SOURCE_SYMBOL frames serve the same purpose. Only one will be removed
in next versions of this document.


## New transport parameters

Parameter ID | Parameter name | Specification
---------|---------------------------------------
0xfec    | enable_fec             | {{sec-enable-fec-tp}}
0xfecd   | decoder_fec_scheme     | {{sec-decoder-fec-scheme-tp}}
0xfecc   | initial_coding_window  | {{sec-initial-coding-window-tp}}
{: #iana-transport-parameters title="New transport parameters"}


## New frames

Frame ID | Frame name | Specification
---------|---------------------------------------
0xfec    | REPAIR          | {{sec-repair-frame}}
0xfec55  | SOURCE_SYMBOL   | {{sec-source-symbol-frame}}
0xfec1d  | SID             | {{sec-sid-frame}}
0xfecac  | SYMBOL_ACK      | {{sec-symbol-ack-frame}}
0xfecc0d | FEC_WINDOW      | {{sec-fec-window-frame}}
{: #iana-frames title="New frames"}

# Acknowledgments
{:numbered="false"}

Maxime Piraux, Olivier Bonaventure and all the authors of
{{I-D.swett-nwcrg-coding-for-quic}}.

