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


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
