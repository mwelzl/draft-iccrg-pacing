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

## More likely to saturate a bottleneck {#losstypes}

We can distinguish between two reasons for packet losses that are due to congestion at a bottleneck with a DropTail (FIFO) queue:

1. A flight of N packets arrives. The amount of data in this flight exceeds the amount of data that can be transmitted by the bottleneck during the flight's arrival plus the queue length, i.e. some data do not fit into the queue.
2. The bottleneck is fully saturated. The queue is full, and packets drain from it more slowly than new packets arrive.

The second type of loss matches the typical expectation of a congestion control algorithm: the cwnd value when loss happens is indicative of the bottleneck being fully saturated. When the first type of loss happens, however, a sender's cwnd can be much smaller than the Bandwidth*Delay Product (BDP) of the path (the amount of data that can be in flight, ignoring the queue). In the absence of other traffic, the probability for the first type of loss to happen depends on the queue length and the ratio between the departure and the arrival rate during the flight's arrival. By introducing time gaps between the packets of a burst, this ratio is increased, i.e. the difference between the departure and the arrival rate becomes smaller, and the second type of loss is more likely.

For example, consider a network path with a bottleneck capacity of 50 Mbit/s, a queue length of 15000 bytes (or 10 packets of size 1500 bytes) and an RTT of 30 ms. Assume that all packets emitted by the sender have a size of 1500 bytes. Then, the BDP equals 125 packets. The bottleneck of this network path is fully saturated when a (BDP + queue length) amount of bytes are in flight: 135 packets.

In this network, the first type of loss can happen as follows: say, N=40 packets arrive at this bottleneck at a rate of 100 Mbit/s. In an otherwise empty network and assuming an initial window of 10 packets and no delayed ACKs, this occurs in the third round of slow start without pacing, provided that the capacities of all links before the bottleneck are at least 100 Mbit/s.
In this case, an overshoot will occur: packets are forwarded with half their arrival rate, i.e. less than 20 packets can be forwarded during the burst's arrival. The remaining 20 (or more) packets cannot fit into the 10-packet queue. A cwnd of 40 packets is much smaller than the (BDP + queue) limit of 135 packets, and the bottleneck is not fully saturated.

Let us now assume that the flight of 40 packets is instead paced, such that the arrival rate only mildly exceeds the departure rate -- e.g., they arrive at a rate of 60 Mbit/s. When the last packet of this flight arrives at the bottleneck, 5/6 * 39 = 32.5 packets should already have been transferred. Since only complete packets are sent, 32 packets have really been transferred, and the remaining 40-32 = 8 packets fit in the queue. No loss occurs.

This example explains how pacing can enable a rate increase to last longer than without pacing. This makes it more likely that a bottleneck is saturated, such that cwnd reflects the BDP plus the queue length (loss type 2).

### Backing off after the increase

The two loss types explained in {{losstypes}} require a different back-off factor to allow the queue to drain and congestion to dissipate. Specifically, in the single-sender single-bottleneck example above, when a slow start overshoot occurs as loss type 2, a back-off function such as: ssthresh = cwnd * beta with beta >= 0.5 is guaranteed to cause a second loss after the end of loss recovery. This is because, when cwnd exceeds a fully saturated bottleneck (i.e., cwnd > BDP + queue length), cwnd will have grown further by another (BDP + queue length) by the time the sender learns about the loss. In this case, beta = 0.5 will cause ssthresh to exceed (BDP + queue length) again.

Since pacing makes loss type 2 more likely, beta < 0.5 may be a better choice after slow start overshoot when pacing is used.

### Able to work with smaller queues

The probability of loss type 1 in {{losstypes}} is indirectly proportional to the queue length. Pacing therefore enables a rate increase to continue with a smaller queue at the bottleneck than in the case without pacing.


## Getting good RTT estimates

Since pacing algorithms generally attempt to spread out packets evenly across an RTT, it is important to have a good RTT estimate. Especially in the beginning of a transfer, when sending the initial window, the only RTT estimate available may be from the SYN-SYN/ACK handshake. Being based on only one sample, this is a very unreliable estimate, and using it to pace the initial window can cause unnecessary delay. This may be the reason why the Linux implementation does not pace the first 10 packets (see {{linux}}). As a possible improvement, the initial RTT estimate could also be based on a previous connection (temporal sharing) or on another ongoing connection (ensemble sharing) {{?RFC9040}}.


# Implementation examples

## Linux TCP {#linux}

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

(TODO)


# Security Considerations

While congestion control designs, including aspects such as pacing, could result in unwanted competing traffic, they do not directly result in new security considerations.

Transport protocols that provide authentication (including those using encryption), or are carried over protocols that provide authentication, can protect their congestion control algorithm from network attack. This is orthogonal to the congestion control rules.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
