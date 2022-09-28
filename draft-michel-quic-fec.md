---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "Forward Erasure Correction for QUIC loss recovery"
# abbrev: "TODO - Abbreviation"
category: info

docname: draft-todo-yourname-protocol-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: AREA
workgroup: WG Working Group
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: WG
  type: Working Group
  mail: WG@example.com
  arch: https://example.com/WG
  github: USER/REPO
  latest: https://example.com/LATEST

author:
 -
    fullname: Your Name Here
    organization: Your Organization Here
    email: your.email@example.com

normative:
  QUIC: RFC9000
  QUIC-RECOVERY: RFC9002

informative:
  QUIC-FEC: DOI.10.23919/IFIPNetworking.2019.8816838
  FlEC: DOI.10.1109/TNET.2022.3195611
  draft-swett-nwcrg-coding-for-quic-04: 



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
{{draft-swett-nwcrg-coding-for-quic-04}}.
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
