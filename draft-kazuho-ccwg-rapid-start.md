---
title: "Rapid Startup of Congestion Control"
category: std
docname: draft-kazuho-ccwg-rapid-start-latest
wg: ccwg
ipr: trust200902
keyword: internet-draft
stand_alone: yes
pi: [toc, sortfefs, symrefs]
author:
 -
    fullname:
      :: 奥 一穂
      ascii: Kazuho Oku
    org: Fastly
    email: kazuhooku@gmail.com

normative:

informative:

...

--- abstract

This document defines Rapid Start, a congestion-control startup algorithm that
grows the window by 3× per RTT until queue buildup is observed, so a sender can
reach the path BDP faster than with classic 2× slow start. When congestion is
observed, Rapid Start immediately scales the window relative to the bytes passed
through the bottleneck and hands over to normal recovery and congestion
avoidance.


--- middle

# Introduction

New transport connections do not know the available bandwidth or the
bandwidth–delay product (BDP) of the path, so TCP and QUIC start from an initial
window and use an exponential startup (“slow start”) to probe for the
bottleneck. Classic slow start doubles the congestion window once per RTT; this
is safe but can take several RTTs to reach the path BDP on high-RTT or high-BDP
paths. This is a poor fit for short-lived connections such as HTTP, where many
connections complete while still in the startup phase.

Rapid Start keeps this IW-based probing model but increases the congestion
window by 3× per RTT while an RTT-sized observation window shows no queueing, so
that the sender reaches the path BDP in fewer RTTs than with 2× slow start. Once
queue buildup is observed in that window, Rapid Start stops using 3× growth and
reverts to 2× growth. If actual congestion is signaled (for example by packet
loss or ECN), Rapid Start does not simply apply a fixed multiplicative decrease;
instead it scales the window based on the amount of data that has passed the
bottleneck in that round and then hands control over to normal recovery and
congestion avoidance.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Algorithm

This section describes the algorithm used by Rapid Start.

## Rapid Start Phase

When the path appears not to be building a queue, the sender uses a more
aggressive startup increase than classic slow start.

Whether the path is "not building a queue" is determined by comparing the floor
RTT of the most recent round-trip with the connection's minimum RTT.

Let:

* `min_rtt` be the minimum RTT observed for the connection so far; and

* `rtt_floor` be the minimum RTT sample observed during the last round-trip
  (i.e., within the most recent interval of length min_rtt).

If `rtt_floor` is no greater than `min(min_rtt + 4ms, min_rtt * 1.10)`, the
sender increases the congestion window (cwnd) by 2 bytes for every byte that is
newly acknowledged, which results in a 3× growth of cwnd per round-trip time.

If `rtt_floor` is greater than the threshold, the sender SHOULD increase the
congestion window as classic slow start does; i.e., by 1 byte for every byte
that is newly acknowledged, which results in a 2× growth of cwnd per round-trip
time.

The additive term (+4 ms) and the multiplicative term (×1.10) are RECOMMENDED
defaults that provide tolerance for typical jitter while keeping Rapid Start out
of the range where early queueing detection algorithms such as HyStart++ are
known to trigger. Therefore, HyStart++ can be used in conjunction with Rapid
Start.


## Pacing Requirement

Rapid Start uses a more aggressive growth factor than classic slow start. When
such growth is used, sending the initial congestion window as a short burst can
make the sender observe a bottleneck overflow earlier than it would under evenly
paced transmission. To ensure that Rapid Start observes the path's queueing
behavior rather than sender-side burstiness, a sender that fully fills the
connection's congestion window for the first time MUST use the pacing described
in Careful Resume ({{!CAREFUL-RESUME=I-D.ietf-tsvwg-careful-resume}}).

For connections that have no prior knowledge of the path (i.e., no previously
saved CC parameters applicable to the 4-tuple), the sender SHOULD limit the
initial jump window (`jump_cwnd`) to at most `2 * IW`. With this bound, the
required pacing rate (`pacing_rate = jump_cwnd / min_rtt`) does not exceed the
pacing rate that would be used by classic slow start with pacing, so Rapid Start
does not create a larger burst than existing paced startup.


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
