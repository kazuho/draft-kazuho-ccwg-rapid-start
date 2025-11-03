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
grows the window by 3× per RTT until queue buildup is observed, so that a sender
can reach the path BDP faster than with classic 2× slow start. When congestion
is observed, Rapid Start immediately scales the window relative to the bytes
that have passed through the bottleneck and then hands over to normal recovery
and congestion avoidance.


--- middle

# Introduction

New transport connections do not know the available bandwidth or the
bandwidth–delay product (BDP) of the path, so TCP and QUIC start from an initial
window and use an exponential startup (“slow start”;
{{Section 3.1 of !RFC5681}}, {{Section 7.3.1 of !RFC9002}}) to probe for the
bottleneck. Classic slow start doubles the congestion window once per RTT. This
is safe, but on high-RTT or high-BDP paths it can still take a considerable
amount of time to reach the path BDP. It is a poor fit for short-lived
connections such as HTTP, where many connections complete while still in the
startup phase.

Rapid Start keeps this IW-based probing model but increases the congestion
window by 3× per RTT while an RTT-sized observation window shows no queueing, so
that the sender reaches the path BDP in fewer RTTs than with 2× slow start. Once
queue buildup is observed in that window, Rapid Start stops using 3× growth and
reverts to 2× growth. If actual congestion is signaled (for example, by packet
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
RTT of the most recent round trip with the connection's minimum RTT.

Let:

* `min_rtt` be the minimum RTT observed for the connection so far; and

* `rtt_floor` be the minimum RTT sample observed during the last round trip
  (i.e., within the most recent interval of length min_rtt).

If `rtt_floor` is no greater than `min(min_rtt + 4 ms, min_rtt * 1.10)`, the
sender increases the congestion window (cwnd) by 2 bytes for every byte that is
newly acknowledged, which results in a 3× growth of cwnd per round-trip time.

If `rtt_floor` is greater than this threshold, the sender SHOULD increase the
congestion window as classic slow start does; i.e., by 1 byte for every byte
that is newly acknowledged, which results in a 2× growth of cwnd per round-trip
time.

The additive term (+4 ms) and the multiplicative term (×1.10) are RECOMMENDED
defaults that provide tolerance for typical jitter while keeping Rapid Start out
of the range where early queueing-detection algorithms such as HyStart++
{{?RFC9406}} are known to trigger. Therefore, HyStart++ can be used in
conjunction with Rapid Start.


## Pacing Requirement

Rapid Start uses a more aggressive growth factor than classic slow start. When
such growth is used, sending the initial congestion window as a short burst can
make the sender observe a bottleneck overflow earlier than it would under evenly
paced transmission. To ensure that Rapid Start observes the path's queueing
behavior rather than sender-side burstiness, a sender that fully fills the
connection's congestion window for the first time MUST use the pacing described
in Careful Resume {{!CAREFUL-RESUME=I-D.ietf-tsvwg-careful-resume}}.

For connections that have no prior knowledge of the path (i.e., no previously
saved CC parameters applicable to the 4-tuple), the sender SHOULD limit the
initial jump window (`jump_cwnd`) to at most `2 * IW`. With this bound, the
required pacing rate (`pacing_rate = jump_cwnd / min_rtt`) does not exceed the
pacing rate that would be used by classic slow start with pacing, so Rapid Start
does not create a larger burst than existing paced startup.


## Congestion Handling

When Rapid Start observes the first packet loss or an explicit congestion
signal (e.g., ECN-CE), the sender enters the recovery period. The purpose of
this period is (1) to drain the queue and (2) after the more aggressive startup,
to bring the congestion window back in line with the actual BDP of the path.

When entering the recovery period, the sender scales the current congestion
window by a silence factor. This momentarily stops transmission so that the
bottleneck queue can drain by a controlled amount.

~~~pseudocode
cwnd *= silence_factor
~~~

During the recovery period, whenever new data is acknowledged, the sender
reduces the congestion window in proportion to the amount that has been newly
acknowledged:

~~~pseudocode
cwnd -= ack_factor * bytes_newly_acked
~~~

Likewise, whenever packet loss is confirmed during the recovery period, the
sender reduces the congestion window in proportion to the amount of data lost:

~~~pseudocode
cwnd -= loss_factor * bytes_newly_lost
~~~

This approach ensures that, by the end of the recovery period,  the congestion
window becomes a fraction of the full BDP (the sum of the idle BDP and the
bottleneck queue size), while keeping the silence period short enough that the
sender is likely to resume transmission before the bottleneck is fully drained,
even if the congestion window had to be reduced significantly to compensate for
the aggressive ramp-up.

The sender SHOULD NOT reduce the congestion window below

~~~pseudocode
cwnd_before_loss * (silence_factor - 1/3 * ack_factor - 2/3 * loss_factor)
~~~

because, if the losses are caused purely by tail drops at the bottleneck queue,
the loss ratio is unlikely to exceed the reciprocal of the most aggressive
growth factor.

Separately, the sender MUST NOT reduce the congestion window below the minima
specified by {{RFC5681}} or {{RFC9002}}.

The sender MAY stop reducing the congestion window once it reaches the initial
window multiplied by `beta`. By doing so, the sender retains the same
aggressiveness as classic slow start on connections whose BDP is very small.


### Deriving the Reduction Factors

The reduction factors are constants derived from `beta`, which is used in the
congestion-avoidance phase. The factors are calculated as:

~~~pseudocode
K               = 11/18
silence_factor  = beta + K * (1 - beta)
ack_factor      = K * (1 - beta)
loss_factor     = beta + K * (1 - beta)
~~~

Specifically, when `beta` is 0.5, the values are:

~~~pseudocode
silence_factor  = 29/36
ack_factor      = 11/36
loss_factor     = 29/36
~~~

When `beta` is 0.7, the values are:

~~~pseudocode
silence_factor  = 53/60
ack_factor      = 11/60
loss_factor     = 53/60
~~~

The formula guarantees the following properties:

* When the loss ratio is 2/3, the duration of the silence period is `1 - beta`
  relative to the full BDP, the same as during the congestion-avoidance phase.
* At the end of the recovery period, the congestion window becomes as large as
  the full BDP multiplied by beta, the same as at the end of the recovery period
  during the congestion avoidance phase.


# Limitations

To estimate the BDP during the first recovery period, Rapid Start depends on the
transport protocol's accurately and promptly reporting the traversal of each
sent packet, even when the packet loss ratio is high. QUIC, with its explicit
packet numbers and ACK frames capable of reporting many gaps, meets this
criterion. However, with TCP, there can be issues producing a reliable estimate.


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
