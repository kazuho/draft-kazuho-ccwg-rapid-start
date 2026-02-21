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

This document defines Rapid Start, a congestion-control startup algorithm. It
paces the transmission of the initial congestion window over a full RTT,
allowing an initial window up to 2× that of classic slow start with pacing at a
comparable pacing rate. It then grows the window by 3× per RTT until queue build
up is observed, after which it reverts to classic 2× slow start growth. When
congestion is signaled, Rapid Start gradually reduces the window in proportion
to delivered and lost bytes, converging to the appropriate window size while
avoiding bursts, before handing over to ordinary congestion avoidance.


--- middle

# Introduction

New transport connections do not know the available bandwidth or the
bandwidth-delay product (BDP) of the path, so TCP and QUIC start from an initial
window and use an exponential startup (“slow start”;
{{Section 3.1 of !RFC5681}}, {{Section 7.3.1 of !RFC9002}}) to probe for the
bottleneck, often paired with pacing to reduce sender-side burstiness. In
practice, paced slow start can still leave performance on the table:

* The sender typically starts by pacing packets for half an RTT and then
  pausing. When the bottleneck bandwidth is higher than the paced rate, the
  bottleneck can remain idle for the other half of each RTT.
* Even when the bottleneck is being utilized, utilization remains below capacity
  until queueing begins.
* When the initial window is much smaller than the path BDP, many round-trips
  are required to ramp up.

These effects are particularly detrimental to short-lived flows, which may only
have a few round-trips to send data and therefore suffer disproportionately from
underutilization during the startup.

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


## Sending in the First Round Trip

Rapid Start uses a more aggressive growth factor than classic slow start. When
such growth is used, sending the initial congestion window as a short burst can
make the sender observe a bottleneck overflow earlier than it would under evenly
paced transmission. To ensure that Rapid Start observes the path's queueing
behavior rather than sender-side burstiness, the sender SHOULD pace the packets
over a full RTT, using the current RTT estimate, when sending the first window's
worth of data.

When pacing over a full RTT, Rapid Start can use an initial window up to 2× that
of classic slow start with pacing, because spreading the transmission over a
full RTT (rather than half an RTT) yields a comparable pacing rate.

A sender using Careful Resume {{?CAREFUL-RESUME=I-D.ietf-tsvwg-careful-resume}}
satisfies these recommendations, because it requires that all packets sent in
its Unvalidated Phase be paced based on `current_rtt`, regardless of prior
knowledge.


## Increasing the Congestion Window

Similarly to Slow Start, Rapid Start increases the congestion window as packets
are acknowledged. The difference is that when the path appears not to be
building a queue, the sender uses a more aggressive startup increase.

Whether the path is "not building a queue" is determined by comparing the floor
RTT of the most recent round trip with the connection's minimum RTT.

Let:

* `min_rtt` be the minimum RTT observed for the connection so far; and

* `rtt_floor` be the minimum RTT sample observed during the most recent
   `min_rtt` interval.

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


## Congestion Handling

When Rapid Start observes the first packet loss or an explicit congestion
signal (e.g., ECN-CE), the sender enters the recovery period. The purpose of
this period is (1) to drain the queue and (2) after the more aggressive startup,
to bring the congestion window back in line with the actual BDP of the path.

When entering the recovery period, the sender scales the current congestion
window by a silence factor. This momentarily pauses transmission so that the
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
pre_recovery_cwnd * (silence_factor - 1/3 * ack_factor - 2/3 * loss_factor)
~~~

where `pre_recovery_cwnd` is the congestion window immediately before entering
the recovery period. This is because if the losses are caused purely by tail
drops at the bottleneck queue, the loss ratio is unlikely to exceed `1 - 1 / G`,
where `G` is the most aggressive growth factor.

Separately, the sender MUST NOT reduce the congestion window below the minima
specified by {{RFC5681}} or {{RFC9002}}.

The sender MAY stop reducing the congestion window once it reaches the initial
window multiplied by the window decrease factor. Doing so preserves classic slow
start's aggressiveness on connections with tiny BDPs as the sender transitions
to congestion avoidance.


### Deriving the Reduction Factors {#reduction-factors}

The reduction factors are constants derived from the multiplicative window
decrease factor (beta), which is used in the congestion avoidance phase. The
factors are calculated as:

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

When `beta` is 0.7 (i.e., that of CUBIC {{?RFC9438}}), the values are:

~~~pseudocode
silence_factor  = 53/60
ack_factor      = 11/60
loss_factor     = 53/60
~~~

The formula guarantees the following properties:

* When the loss ratio is 2/3, the duration of the silence period is `1 - beta`
  relative to the full BDP, the same as during the congestion avoidance phase.
* At the end of the recovery period, the congestion window becomes as large as
  the full BDP multiplied by beta, the same as at the end of the recovery period
  during the congestion avoidance phase.


### Interaction with ECN

{{reduction-factors}} provides the rationale for the recovery behavior in terms
of the full BDP (which, under loss-based detection, includes filling the
bottleneck queue up to the point it overflows and packets are dropped). However,
when congestion happens on an ECN-capable path, it can be reported via CE marks
without requiring packet loss. If Rapid Start enters a recovery period upon
observing a CE mark but no packets are lost, then it exits recovery with a
congestion window that is beta times its size immediately before entering
recovery.

If the growth factor in the last round-trip was 3×, the congestion window upon
entering recovery can be larger than with 2×, and therefore the congestion
window at the end of recovery (beta times the entry size) can also be larger.
This makes the next recovery period start sooner, but otherwise does not change
the flow's behavior under ECN-signaled congestion pressure.

The other concern is buffer overflow before CE feedback is observed. Under 3×
growth, the sender might build up a bottleneck queue that is twice as large as
under 2× growth. However, even in the extreme case where a network’s buffering
margin is tightly provisioned for a target maximum RTT under conventional slow
start (i.e., 2× growth), this larger queue buildup under 3× growth simply halves
the loss-free RTT range: only connections with RTTs above half of that target
maximum would be affected. In practice, networks do not generally provision
buffers that tightly; with ECN, they can signal congestion without relying on
drop, so leaving extra buffering margin typically has little downside. For these
reasons, this overflow risk is limited in practice.

On loss-based paths, a more aggressive startup increases the likelihood of
overflowing the bottleneck buffer and triggering packet drops, which delays
delivery to the application due to retransmission. In contrast, on ECN-capable
paths, congestion is typically signaled without relying on packet drops, so this
loss-induced delivery delay mode is largely avoided. As a result, the benefits
of faster growth of the congestion window are more reliable.


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

"SUSS: Improving TCP Performance by Speeding Up Slow-Start" (Mahdi Arghavani et
al.) advocates a similar approach built on top of HyStart that increases the
congestion window by up to 4× per round-trip based on ACK dispersal and RTT.
Compared to SUSS, Rapid Start, relying solely on RTT, is simpler and allows
reuse of existing mechanisms and specifications such as pacing, HyStart++, and
Careful Resume. Rapid Start also specifies how the congestion window should be
decreased upon congestion, allowing a smooth transition to congestion avoidance.
