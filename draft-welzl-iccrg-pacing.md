---
title: Pacing in Transport Protocols
abbrev: Pacing in Transport Protocols
category: info

docname: draft-welzl-iccrg-pacing-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "IRTF"
workgroup: "Internet Congestion Control"
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: "Internet Congestion Control"
  type: "Research Group"
  mail: "iccrg@irtf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/iccrg"
  github: "mwelzl/draft-iccrg-pacing"
  latest: "https://mwelzl.github.io/draft-iccrg-pacing/draft-welzl-iccrg-pacing.html"

author:
  -
    ins: M. Welzl
    name: Michael Welzl
    org: University of Oslo
    street: PO Box 1080 Blindern
    city: 0316  Oslo
    country: Norway
    email: michawe@ifi.uio.no
    uri: http://welzl.at/
  -
    ins: W. Eddy
    name: Wesley Eddy
    org: MTI Systems
    street: 5929 Talbot Rd
    city: Lothian, MD 20711
    country: United States of America
    email: wes@mti-systems.com

normative:

informative:


--- abstract

Applications or congestion control mechanisms can produce bursty traffic which can cause unnecessary queuing and packet loss. To reduce the burstiness of traffic, the concept of evenly spacing out the traffic from a data sender over a round-trip time known as Pacing has been used in many transport protocol implementations. This document gives an overview of Pacing and how some known Pacing implementations work.


--- middle

# Introduction

RFCs describing congestion control generally refer to congestion window (cwnd) as an upper limit for the number of unacknowledged packets a sender is allowed to emit. This limits the sender's transmission rate at the granularity of a round-trip time (RTT). If the sender transmits the entire cwnd sized data in an instant, this can results in unnecessarily high queuing and eventually packet losses at the bottleneck. Such consequences are detrimental to users' applications in terms of both responsiveness and goodput. To solve this problem, the concept of pacing was introduced. Pacing allows to send the same cwnd sized data but spread it across a round-trip time more evenly.

Congestion control specifications always allow to send less than the cwnd, or temporarily emit packets at a lower rate. Accordingly, it is in line with these specifications to pace packets. Pacing is known to have advantages -- if some packets arrive at a bottleneck as a burst (all packets being back-to-back), loss can be more likely to happen than in a case where there are time gaps between packets (e.g., when they are spread out over the RTT). It also means that pacing is less likely to cause any sudden, ephemeral increases in queuing delay. Since keeping the queues short reduces packet losses, pacing can also yield higher goodput by reducing the time lost in loss recovery.

Because of its known advantages, pacing has become common in implementations of congestion controlled transports. It is also an integral element of the "BBR" congestion control mechanism {{!I-D.cardwell-iccrg-bbr-congestion-control}}.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Pacing: general considerations and consequences

TODO

# Implementation examples

## Linux TCP

The following description is based on Linux kernel version 6.7.3.

There are two ways to enable pacing in Linux: 1) via a socket option, 2) by configuring the FQ queue discipline. We describe case 1).

Independent of the value of the Initial Window (IW), the first 10 (hardcoded) packets are not paced. Later, 10 packets will generally be sent without pacing every 2^32 packets.

Every time an ACK arrives, a pacing rate is calculated, as: factor * MSS * cwnd / SRTT, where "factor" is a configurable value that, by default, is 2 in slow start and 1.2 in congestion avoidance. MSS is the sender maximum segment size {{?RFC5681}}, and SRTT is the smoothed round-trip time {{?RFC6298}} [TODO check: Linux calculates SRTT different from the standard, though RFC 6298 relaxes the rules, so maybe it's ok?]
The sender transmits data in line with the calculated pacing rate; this is approximated by calculating the rate per millisecond, and generally sending the resulting amount of data per millisecond as a burst, every millisecond. This amount of data can be larger when the peer is very close (this is a configurable value, per default with a minimum RTT of 3 milliseconds).

If the pacing rate is smaller than 2 packets per millisecond, these bursts will become 2 packets in size, and they will not be sent every millisecond but with a varying time delay (depending on the pacing rate).
If the pacing rate is larger than 64 Kbyte per millisecond, these bursts will be 64 Kbyte in size, and they will not be sent every millisecond but with a varying time delay (depending on the pacing rate).
Bursts can always be smaller than described above, or be "nothing", if a limiting factor such as the receiver window (rwnd) {{?RFC5681}} or the current cwnd disallows transmission.
If the previous packet was not sent when expected by the pacing logic, but more than half of a pacing gap ago (e.g., due to a cwnd limitation), the pacing gap is halved.


**TEMPORARY NOTE - TO BE REMOVED:** This description is based on the longer Linux pacing analysis text that is currently available at: [https://docs.google.com/document/d/1h5hN9isFjT76YjaCphHZdW9LCRYqV4y3GKwRKxgqEO0/edit?usp=sharing](https://docs.google.com/document/d/1h5hN9isFjT76YjaCphHZdW9LCRYqV4y3GKwRKxgqEO0/edit?usp=sharing)  - comments or corrections are very welcome!


## Apple OSes

(TODO)


## QUIC BBR implementations

(TODO)


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
