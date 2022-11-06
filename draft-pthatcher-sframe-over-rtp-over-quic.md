---
title: "SFrame over RTP over QUIC"
docname: draft-pthatcher-sframe-over-rtp-over-quic-latest
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
    ins: P. Thatcher
    name: Peter Thatcher
    organization: Microsoft
    email: pthatcher@microsoft.com

--- abstract

This document specifies how to send SFrame {{?I-D.draft-ietf-sframe-enc-00}} over RTP over QUIC {{?I-D.draft-ietf-avtcore-rtp-over-quic-01}}, including how to translate SFrame from RTP over QUIC to RTP over UDP.

--- middle

# Introduction

RTP over QUIC {{?I-D.draft-ietf-avtcore-rtp-over-quic-01}} describes how to send and received real-time media over QUIC.

SFrame {{?I-D.draft-ietf-sframe-enc-00}} describes how to encrypt a media frame such that media servers (such as SFUs) have access to certain metadata, but not to the media.

This document describes how to send SFrame over RTP over QUIC while maintaining support for the encryption
properties supported by SFrame and the RTP topologies supported by RTP over QUIC.

# Terminology and Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.


# How to combine SFrame and RTP over QUIC

To send SFrame {{?I-D.draft-ietf-sframe-enc-00}} over RTP over QUIC {{?I-D.draft-ietf-avtcore-rtp-over-quic-01}} using reliable QUIC streams ({{?I-D.draft-ietf-avtcore-rtp-over-quic-01}} Section 6.1), one SFrame is sent over one QUIC stream.  When sending with unreliable datagrams ({{?I-D.draft-ietf-avtcore-rtp-over-quic-01}} Section 6.2), the SFrame payload must be packetized before sending as described in {{?I-D.draft-ietf-sframe-enc-00}} Section 4.

# Supported RTP Topologies

{{?I-D.draft-ietf-avtcore-rtp-over-quic-01}} Section 4.1 describes the RTP topologies
supported RTP over QUIC and the use of RTP translators.  Notably, translation from from larger RTP packets (such as those sent over QUIC streams) to smaller RTP packets (such as those sent over UDP)
is supported, and thus is also supported when sending SFrame over RTP over QUIC.  In this case, according to Section 4.1, "Such a translator may need codec-specific knowledge to packetize the payload of the incoming RTP packets in smaller RTP packets." 

Because SFrame encrypts the media contained in it, the only "codec-specific knowedge" a translator can get to when using SFrame is that of SFrame itself.  In effect, SFrame is the codec, and the transator must know
how to packetize SFrame.

# SFrame Packetization

{{?I-D.draft-ietf-sframe-enc-00}} refers to a "generic RTP packetizer", but the format for that packetization is not defined.  So for purposes of translating from SFrame over RTP over QUIC to SFrame over RTP over UDP, we define an SFrame packetization here.

When translating from one large RTP packet to many small RTP packets (such as a QUIC stream containing an SFrame to many RTP packets over UDP), the following format MAY be used:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  large sequence number        |   chunk index                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                  large payload chunk                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~


When using this format, the translator MUST construct small RTP packets such that they may be translated back into a large RTP packet using the following steps:

1. The large RTP packet's payload is reconstructed by concatenating the large payload chunks of all the small RTP packets that share a large sequence number in the order of the chunk indexes beginning from index 0 until the smaller RTP packet marked with the marker bit (indicating "last chunk").

2. The large RTP header is reconstructed by copying the RTP header of the small RTP packet with chunk index 0 and replacing the sequence number with the large sequence number.

To be explicit, a translator using this format to translate from a large RTP packet to small RTP packets MUST:

1.  Set the large sequence number in all of the small RTP packets to the sequence number of the large RTP packet.

2.  Set the chunk index of the first small RTP packet to 0.

3.  Set the chunk index to consectutive values for consecutive large payload chunks.

4.  Unset (set to 0) the marker bit for all but the last small RTP packet.

5.  Set the marker bit for the last small RTP packet.

## Tradeoffs

It may be possible to save 1 byte by deriving the chunk index from the smaller RTP packet's sequence number and a bit somewhere that means "this is the first chunk", but that would come at the cost of additional 
complexity and reduced flexibility.  This format simplifies packet reconstruction when chunks arrive out of order, and allows for more flexibility in translation. For example, if a translator wanted to translate from smaller packets to larger packets but received chunk indexes 1 and 2 before chunk index 0, it could begin forwarding 1 and 2 in one QUIC stream while waiting for index 0.  It wouldn't have to wait for the first chunk of a frame before forwarding subsequent chunks.


# Security Considerations

This document is subject to the security considerations of SFrame {{?I-D.draft-ietf-sframe-enc-00}} and RTP over QUIC {{?I-D.draft-ietf-avtcore-rtp-over-quic-01}}.

