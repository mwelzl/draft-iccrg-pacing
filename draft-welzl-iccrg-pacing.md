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
    street: 25111 Country Club Blvd, Suite 295
    city: North Olmsted, OH 44070
    country: United States of America
    email: wes@mti-systems.com

normative:

informative:

  RFC9002:

  VL87:
    title: "Hashed and hierarchical timing wheels: data structures for the efficient implementation of a timer facility"
    author:
      -
        ins: G. Varghese
      -
        ins: T. Tauck
    date: 1987-11-01
    seriesinfo:
      DOI: 10.1145/37499.37504

--- abstract

Applications or congestion control mechanisms can produce bursty traffic which can cause unnecessary queuing and packet loss. To reduce the burstiness of traffic, the concept of evenly spacing out the traffic from a data sender over a round-trip time known as "Pacing" has been used in many transport protocol implementations. This document gives an overview of Pacing and how some known Pacing implementations work.


--- middle

# Introduction

Applications commonly generate either bulk data (e.g. files) or bursts of data (e.g. segments of media) that transport protocols deliver into the network based on congestion control algorithms.

RFCs describing congestion control generally refer to a congestion window (cwnd) state variable as an upper limit for either the number of unacknowledged packets or bytes that a sender is allowed to emit. This limits the sender's transmission rate at the granularity of a round-trip time (RTT). If the sender transmits the entire cwnd sized data in an instant, this can result in unnecessarily high queuing and eventually packet losses at the bottleneck. Such consequences are detrimental to users' applications in terms of both responsiveness and goodput. To solve this problem, the concept of pacing was introduced. Pacing allows to send the same cwnd sized data but spread it across a round-trip time more evenly.

Congestion control specifications always allow to send less than the cwnd, or temporarily emit packets at a lower rate. Accordingly, it is in line with these specifications to pace packets. Pacing is known to have advantages -- if some packets arrive at a bottleneck as a burst (all packets being back-to-back), loss can be more likely to happen than in a case where there are time gaps between packets (e.g., when they are spread out over the RTT). It also means that pacing is less likely to cause any sudden, ephemeral increases in queuing delay. Since keeping the queues short reduces packet losses, pacing can also yield higher goodput by reducing the time lost in loss recovery.

Because of its known advantages, pacing has become common in implementations of congestion controlled transports. It is also an integral element of the "BBR" congestion control mechanism {{!I-D.cardwell-iccrg-bbr-congestion-control}}.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Pacing: general considerations and consequences

TODO

# Implementation examples

## Linux TCP

The following description is based on Linux kernel version 6.8.9.

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

Pacing capability is expected in QUIC senders.  While standard QUIC congestion control {{RFC9002}} is based on TCP Reno, which is not defined to include pacing (but also does not prohibit it), QUIC congestion control requires either pacing or some other burst limitation ({{Section 7.7 of RFC9002}}).  BBR congestion control implementations are common in QUIC stacks, and pacing is integral to BBR, so this document focuses on it.

Pacing in QUIC stacks commonly involves:

1. Access to lower-level (e.g. OS and hardware) capabilities needed for effective pacing.

2. Managing additional timers related to pacing, along with those already needed for retransmission, and other events.

3. Details of the actual pacing algorithm (e.g. granularity of bursts allowed, etc.).

Examples of different approaches to dealing with these challenges in ways that work on multiple operating systems and hardware platforms can be found in open source QUIC stacks, such as Google "quiche" and Meta "mvfst", that provide examples for some of the concepts discussed below.

Unlike TCP implementations that typically run within the operating system kernel, QUIC implementations more typically run in user space and are thus faced with more challenges regarding timing and coupling with the underlying protocol stack and hardware needed to achieve pacing.  For instance, if an application trying to do pacing is running on a highly loaded system, it may often "wake up late" and miss the times that it intends to pace packets.

When a large amount of data needs to be sent, pacing naively could result in an excessive number of timers to be managed and adjusted along with all of the other timers that the QUIC stack and rest of the application require.  The Hashed Hierarchical Timing Wheel {{VL87}} provides one approach for such cases, but implementations may also simply schedule the next send event based on the current pacing rate, and then schedule subsequent events as needed, rather than adjusting timers for them.  In any case, typically a pacing algorithm should allow for some amount of burstiness, in order to efficiently use the hardware as well as to be responsive for bursty (but low overall rate) applications, and to avoid excessive timer management.

Pacing can be done based on different approaches such as a token-based or tokenless algorithm.  For instance, a tokenless algorithm might compute a regular interval time and batch size (number of packets) to be released every interval and achieve the pacing rate.  This allows specific future transmissions to be scheduled.  In contrast, a token-based algorithm accumulates tokens to permit transmission based on the pacing rate, using a "leaky bucket" to control bursts.  In this case the size of bursts may be more granular, depending on how much time is elapsed between evaluations.

The additional notion of "burst tokens" (or other burst allowance) may be present in order to rapidly transmit data if coming out of a quiescent period (e.g. when a flow has been application-limited without data to send).  A number of burst tokens, representing packets that can be sent unpaced, is initialized to some value (e.g. 10) when a flow starts or becomes quiescent.  If burst tokens are available, outgoing packets are sent immeidately, without pacing, up to the limit permitted by the congestion window, and the burst tokens are depleted by each packet sent.  The number of burst tokens is reduced to zero on congestion events.  When coming out of quiescence, it is set to the minimum of the initial burst size, or the amount of packets that the congestion window (in bytes) represents.

There may be additional "lumpy tokens" that further allow unpaced packets after the burst tokens have been consumed, and the congestion window does not limit sending.  The amount of lumpy tokens that might be present is determined using heuristics, generally limiting to a small number of packets (e.g. 1 or 2).

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
