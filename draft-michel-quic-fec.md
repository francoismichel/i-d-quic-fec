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


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
