---
title: "Forward Erasure Correction for QUIC loss recovery"
abbrev: "FEC for QUIC"
category: exp

docname: draft-michel-quic-fec-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Transport"
workgroup: "QUIC"
keyword:
 - FEC
 - network coding
 - QUIC
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
 -
    fullname: Olivier Bonaventure
    organization: UCLouvain, WEL RI
    email: "olivier.bonaventure@uclouvain.be"

normative:
  QUICv1: RFC9000
  QUIC-RECOVERY: RFC9002
  RFC9265:
  QUIC-DATAGRAM: RFC9221

informative:
  QUIC-FEC: DOI.10.23919/IFIPNetworking.2019.8816838
  rQUIC: DOI.10.1109/GLOBECOM38437.2019.9013401
  FlEC: DOI.10.1109/TNET.2022.3195611
  I-D.swett-nwcrg-coding-for-quic:
  I-D.roca-nwcrg-rlc-fec-scheme-for-quic:
  I-D.irtf-nwcrg-tetrys:
  RFC6865:




--- abstract

This documents lays down the QUIC protocol design considerations
needed for QUIC to apply Forward Erasure Correction on the data sent
through the network.



--- middle

# Introduction

The QUIC protocol {{QUICv1}} relies on retransmissions to ensure
the reliable delivery of stream data. Retransmitting the lost
information requires the loss recovery mechanism to identify lost
packets which may take up to several hundreds of milliseconds
{{QUIC-RECOVERY}}. Depending on their delay-sensitivity, some
applications using QUIC could not afford such a waiting time to
ensure a good quality of experience to their users.

Works has already been done to consider the use of
Forward Erasure Correction (FEC) for the QUIC protocol to ensure timely
data delivery for delay-sensitive applications {{QUIC-FEC}} {{FlEC}}
{{rQUIC}} {{I-D.swett-nwcrg-coding-for-quic}}. These loss recovery
mechanisms generally maintain a coding window containing the
latency-sensitive application data and generate repair symbols
protecting this coding window from packet losses.
This document defines additions to the QUIC protocol to extend its
loss recovery mechanism with FEC capabilities and make it able to
recover from packet losses prior to their retransmission.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

## FEC-related definitions

Source symbol: piece of information exchanged by two endpoints. This document considers QUIC packets payloads as source symbols.

Repair symbol: redundant information constructed from the combination
of several source symbols.

Erasure: loss of one or more symbols

Erasure correction code: algorithm generating repair symbols and
reconstructing missing source symbols from a set of source and
repair symbols.

Forward Erasure Correction: process of recovering erased symbols
before the detection of their erasure.

FEC scheme: the conjunction of an erasure correction code and the
specific protocol elements required to use it with the design
described in this document.

Encoder: entity producing repair symbols using an erasure correction code.
The encoder can be a library used by the protocol implementation or a
program running in a separate process or machine.

Decoder: entity reconstructing missing source symbols using an erasure
correction code. The decoder can be a library used by the protocol
implementation or a program running in a separate process or machine.

Coding window: window of source symbols that are required to decode a
repair symbol.

# Network packets and coded symbols

QUIC endpoints exchange information over a network channel. Adding
Forward Erasure Correction to QUIC introduces adds a FEC encoder and a
FEC decoder to the QUIC endpoint. The encoder and decoder exchange source
and repair symbols that are carried through QUIC frames inside QUIC
packets.
{{fig-packets-and-symbols}} illustrates how a FEC-enabled QUIC
endpoint behaves.

~~~~
  +---------------------------------------------------------+
  |                       Application                       |
  +---------------------------------------------------------+
       | Application data                            ^
       | to send              Received and recovered |
       v                            application data |
    +------+                                      +------+
  __| Send |______________________________________| Recv |_______
 |  | API  |                                      | API  |       |
 |  +------+                                      +------+       |
 |     |                                             ^           |
 |     v                                             |           |
 | +---------+                QUIC              +---------+      |
 | |   FEC   |                                  |   FEC   |      |
 | | Encoder |                                  | Decoder |      |
 | +---------+                                  +---------+      |
 |      | Source and                                 ^           |
 |      | repair symbols             Received source |           |
 |      | to send                         and repair |           |
 |      v                                    symbols |           |
 |   +--------+                                +----------+      |
 |   |  QUIC  |                                |   QUIC   |      |
 |   | Sender |                                | Receiver |      |
 |   +--------+                                +----------+      |
 |_______|___________________________________________^___________|
         | QUIC packets                              |
         | to send                          Received |
         v                              QUIC packets |
      +-------------------------------------------------+
      |                     Network                     |
      +-------------------------------------------------+

~~~~
{: #fig-packets-and-symbols title="Exchanging source and
repair symbols over a QUIC connection"}

The application submits new data using the stream or datagram abstraction
provided by the QUIC Send API (left part of {{fig-packets-and-symbols}}).
The FEC Encoder encodes the
application data into one or several source symbols and generates repair
symbols protecting these when needed. These symbols are then packed into
network packets by the QUIC Sender.
When repair symbols must be sent, the QUIC Sender packs them inside
dedicated QUIC frames discussed in {{sec-repair-frame}}.
On the receiving path (right part of {{fig-packets-and-symbols}}),
the QUIC Receiver consumes network packets and unpacks the symbols they
contain. It provides the received symbols
to the FEC Decoder that then recovers the lost source symbols when
possible. It finally passes the application data present in the newly received or recovered source symbols to the application using the QUIC
Recv API.


# FEC and the loss recovery mechanism

The FEC mechanism described in this document is an enhancement of the
classical QUIC loss recovery mechanism {{QUIC-RECOVERY}}. It does
not replace it by any means. A QUIC endpoint MAY ignore every received
repair symbol and MAY not perform any symbol recovery at all. The FEC
mechanism is only intended to allow a receiver recovering faster from
packet losses on the network if it values the timeliness of
data delivery.


# Protocol requirements for protecting information through FEC

In this section, we list the points that must be defined by the protocol
for allowing QUIC endpoints to protect information using FEC.


## Defining the FEC-protected parts of a QUIC payload

There is no need to protect every piece information sent on the wire by
QUIC. Some pieces of information are already sent redundantly (e.g. ACKs)
and some data are not delay sensitive and can be retransmitted later with
no harm (e.g. background download on a separate stream). Endpoints need
to agree on which parts of the packets are part of the protected source
symbols and how to compose a source symbol from what is sent on the wire.

Versions 01 and 02 of {{I-D.swett-nwcrg-coding-for-quic}} only protect
streams payload. The idea is simple when using a single stream but becomes
complicated and requires more signaling in a multi-stream scenario. It also
cannot protect DATAGRAM frames {{QUIC-DATAGRAM}}.
In this document, we propose to consider whole frames as part of the source
symbols.
In order to reduce signalling between the peers, a single source symbol
MUST NOT contain the frames of several QUIC packets at the same time.


## Identifying the source symbols from QUIC packets

Upon reception of a QUIC packet, a receiver needs to identify the source
symbols contained in the packet to forward to the FEC decoder.
The decoder also needs to know which source symbols were lost. Since QUIC
does not enforce sending contiguously increasing packet numbers, it is not
possible for a receiver to distinguish a lost packet from a packet that
has never been sent. Furthermore, a QUIC sender may not want to protect
the payload of some packets if they do not carry latency-sensitive
information. Source symbols are thus attributed a Symbol ID (SID). The
SID of the first source symbol MUST be zero and the SIDs are contiguously
increasing. When a FEC decoder notes gaps in the received SIDs,
the missing SIDs correspond to lost source symbols that can be
recovered using FEC.

As the QUIC packet number cannot be used to carry the SID, it must
be transmitted using either a dedicated QUIC frame or a dedicated
header field. The second solution being incompatible with {{QUICv1}},
it is not discussed in this document. The source
symbol payload can either be put inside a dedicated frame
({{sec-source-symbol-frame}}) or infered when handling a specific frame
({{sec-sid-frame}}). Both alternatives are presented in the next
sections. Lead to the exact same packet wire format and outcome.


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

The frame explicitly represents a source symbol. The FEC Protected Payload
field is analogous to the payload of a QUIC packet: it contains a
sequence of frames that are protected by FEC. The SOURCE_SYMBOL frame
is idempotent and explicit: it exactly describes the frames inside the
source symbol. The main drawback is that existing QUIC implementations
are not used to write frames inside other frames which may increase
the implementation cost of the approach. An example of the use of the
SOURCE_SYMBOL frame is shown in {{fig-source-symbol-frame-example}}.


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

The two SOURCE_SYMBOL frames contain the source symbols with
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
MUST NOT contain more than one SID frame. A packet whose payload is
FEC-protected MUST contain a SID frame whose SID field is the SID of
the related source symbol. This alternative has one drawback: the SID frame
is not idempotent since it is related to its containing packet.
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

The source symbols carried by the packets in this example are the same as for
{{fig-source-symbol-frame-example}} and the outcome and wire
format are the same.

### Choosing an alternative

One of these two design alternatives must be chosen to complete the design
of this document. As authors, we tend to prefer Alternative 1 due to the
idempotent character of the introduced frame.

## Sending the repair symbols
{: #sec-repair-frame}

The repair symbols and the metadata attached to them are transferred using
the REPAIR frame shown in {{fig-fec-repair-frame}}.

~~~~
REPAIR {
  FEC Scheme Specific Repair Payload (1..),
}
~~~~
{: #fig-fec-repair-frame title="REPAIR frame format"}

The payload of the REPAIR frame is specific to the underlying
FEC scheme. In addition to the repair symbol itself, it may
contain any metadata needed by the erasure-correcting code (e.g. identifying
the source symbols protected by the repair symbol carried by the frame).
Depending on the FEC scheme, the REPAIR frame MAY contain
only a part of a repair symbol.


## Announcing the coding window size
{: #sec-fec-window-frame}

The receiver needs to store the received symbols in order to recover the
lost source symbols. The FEC_WINDOW frame is sent by the receiver
to announce the number of symbols  that can be stored simultaneously by
the receiver at a given point of time. The format of the FEC_WINDOW frame
is described in {{fig-fec-window-frame}}.

~~~~
FEC_WINDOW {
  FEC Window Epoch (i),
  FEC Window Size (i),
}
~~~~
{: #fig-fec-window-frame title="FEC_WINDOW frame format"}

The FEC Window Epoch field is a unique identifier for the announced window.
The first epoch is set to 0. Each time a new FEC_WINDOW frame is sent,
the FEC Window Epoch field is increased by exactly one.

The Window Size field indicates the number of symbols that can be stored
simultaneously by the receiver. The Window Size value overrides the
window sizes received for smaller window epochs.


## Announcing the recovered symbols
{: #sec-symbol-ack-frame}

The FEC receiver MAY advertise the recovered source symbols to avoid the sender retransmitting already recovered data. This can be done using the SYMBOL_ACK frame as shown in {{fig-symbol-ack-frame}}.

~~~~
SYMBOL_ACK {
  Largest Acknowledged (i),
  ACK Range Count (i),
  First ACK Range (i),
  ACK Range (..) ...,
}
~~~~
{: #fig-symbol-ack-frame title="SYMBOL_ACK frame format"}

The frame has a similar format as the ACK frame, announcing the reception
of SIDs instead of packet numbers. In addition to symbols
recovered by FEC, this frame MAY also announce symbols received regularly
through the network to avoid gaps in the ACK ranges and reduce the frame
size. There is no obligation for a FEC receiver to send SYMBOL_ACK frames.
The FEC receiver MAY decide to only advertize a subset of the received
source symbols. The SYMBOL_ACK frame MUST NOT be used to infer any congestion
state on the network (see {{sec-coding-and-congestion}}).

## Example

To illustrate the different mechanisms and frames introduced here,
let us take an example where an application sends the bytes
"ABCDEF" over a single QUIC stream. {{fig-example-fec-mechanisms}}
illustrates this example. In this example, the QUIC Sender
sends the stream data into two separate regular STREAM frames.
Following the Alternative 1 proposed in {{sec-source-symbol-frame}},
these two STREAM frames are then placed into two SOURCE_SYMBOL
frames sent in the QUIC packets 1 and 2 and are protected
by a REPAIR_SYMBOL frame sent in packet 3. PKT(1)
containing the bytes "ABC" is lost. Upon reception of PKT(2),
the bytes "DEF" must be stored by the receiver. The receiver then
has to wait for receiving the first part of the stream before
delivering "DEF" to the application. Once PKT(3) is received,
the repair symbol it contains can be used to recompute the
first source symbol containing the bytes "ABC" without having
to wait for a retransmission. The receiver can tehen deliver
"ABCDEF" to the application. The packets received through the
network are acknowledged using a regular ACK frame and the
recovered source symbol is acknowledged using the SYMBOL_ACK
frame defined in this document.

~~~~
        QUIC Sender                               QUIC Receiver
            |                                           |
  App sends |                                           |
   "ABCDEF" |  PKT(1)[SOURCE_SYMBOL(1, STREAM{"ABC"})]  |
 ---------->|---------------------x                     |
            |                                           |
            |  PKT(2)[SOURCE_SYMBOL(2, STREAM{"DEF"})]  |
            |------------------------------------------>| Store "DEF"
            |                                           |
            |  PKT(3)[REPAIR_SYMBOL]                    |
            |------------------------------------------>| (Recompute )
            |                                           | (the source)
            |                                           | (symbol    )
            |                                           |
            |        PKT(2)[ACK[2, 3], SYMBOL_ACK([1])] |
            |<------------------------------------------| Deliver
(Empty rtx) |                                           | "ABCDEF"
(    queue) |                                           | to the App
            |                                           |---------->
            |                                           |
            |                                           |
            |                                           |
~~~~
{: #fig-example-fec-mechanisms title="Recovering lost stream data using FEC"}

# Coding and congestion control
{: #sec-coding-and-congestion}

The fact of successfully recovering symbols SHOULD NOT be
used to infer any congestion state on the network.
More specifically, the recovery a of a symbol through FEC decoding
SHOULD NOT be used to hide the network loss event of the corresponding
packet to the congestion control, applying Recommandation 1 of {{RFC9265}}.


# Negociating the FEC extension using transport parameters

This section defines the new transport parameters used to negociate and
parametrize the FEC extension described in this document.


## enable_fec
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


## decoder_fec_scheme
{: #sec-decoder-fec-scheme-tp}

Each QUIC endpoint uses the decoder_fec_scheme transport parameter to
define the FEC scheme used to decode the received repair symbols. The
QUIC sender MUST use the specified FEC scheme to generate repair symbols.
The decoder_fec_scheme parameter is an integer value representing the
identifier of the desired FEC scheme.
For instance, a FEC scheme using Reed Solomon could be identified by the
ID 0x0 and a FEC scheme using LDPC could be identified by 0x1.

This document does not specify nor identify any FEC scheme yet.
Several FEC schemes have been proposed
{{I-D.roca-nwcrg-rlc-fec-scheme-for-quic}} {{RFC6865}}
{{I-D.irtf-nwcrg-tetrys}}. The next version of this
document will detail how some of these schemes can be directly integrated
in QUIC.

Future
versions of this document will provide ways to format FEC scheme-specific
payload for REPAIR frames.
When the decoder_fec_scheme parameter is not advertized by the peer,
the QUIC sender MUST NOT send any repair symbol.


## initial_coding_window
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

An attacker could try to cause a DoS of a receiver by selectively sending source
and repair symbols to trigger intensive erasure correction operations on the
receiver. A QUIC receiver is never forced to perform any erasure correction
and may ignore any received repair symbol if it has doubts in its capabilities
to decode it in a reasonable amount of time.


# IANA Considerations

*Disclaimer: the IDs defined in this section are present for experimental*
*purposes only. They are not requested codepoints and are subject to change*
*in the next versions of this document.*

This document defines three new transport parameters and five new frames. The
SID and SOURCE_SYMBOL frames serve the same purpose. One of them will be
removed in next versions of this document. The values present in the tables
are used for experiments.


## New transport parameters

Parameter ID | Parameter name | Specification
---------|---------------------------------------
0x238ffeceXX | enable_fec             | {{sec-enable-fec-tp}}
0x238ffecd   | decoder_fec_scheme     | {{sec-decoder-fec-scheme-tp}}
0x238ffecc    | initial_coding_window  | {{sec-initial-coding-window-tp}}
{: #iana-transport-parameters title="New transport parameters"}

The XX in 0x238ffeceXX are to be replaced by the version of this document
that is implemented by the QUIC endpoint (e.g. the parameter ID for the
version 00 of this document is 0x238ffece00).


## New frames

Frame ID | Frame name | Specification
---------|---------------------------------------
0x32a80fec    | REPAIR          | {{sec-repair-frame}}
0x32a80fec55  | SOURCE_SYMBOL   | {{sec-source-symbol-frame}}
0x32a80fec1d  | SID             | {{sec-sid-frame}}
0x32a80fecac  | SYMBOL_ACK      | {{sec-symbol-ack-frame}}
0x32a80fecc0 | FEC_WINDOW      | {{sec-fec-window-frame}}
{: #iana-frames title="New frames"}

# Acknowledgments
{:numbered="false"}

Maxime Piraux and all the authors of
{{I-D.swett-nwcrg-coding-for-quic}}.

