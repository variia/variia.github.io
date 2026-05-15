---
title: "EC2 Network Baseline vs Burst: What AWS Actually Delivers"
date: 2026-05-14 12:00:00 +0100
author: Iván Ádám Vári
description: A controlled benchmark of EC2 network performance across six instance types, covering baseline enforcement, burst non-determinism, hard caps, and how to make bw_out_allowance_exceeded a useful monitoring signal.
keywords: aws,benchmark,ec2,network,bandwidth,baseline,burst,iperf3,ena,instance-type,m5,m5n,m6i,m6g,m8i,m8g
categories:
  - AWS
tags:
  - aws
  - cloud
  - network
  - performance
  - ec2
---

When selecting an EC2 instance type for a network-intensive workload, the AWS documentation gives you two numbers: a guaranteed baseline and an "up to X Gbps" burst ceiling. The question that documentation does not answer is: how long does burst last, and what happens when it runs out?

This post documents a controlled benchmark across six instance types — m5n, m8i, m8g, m5, m6i, m6g — designed to answer that question with real data. The motivation was an instance refresh for a busy production 8-node kafka cluster handling over 1M messages every second. We needed to decide whether the n-series guaranteed baseline was necessary
or whether a newer-generation instance but with a lower baseline was the right tradeoff.

---

## The question

AWS [publishes](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-network-bandwidth.html) two network performance numbers for most instance types:

- **Baseline**: guaranteed, available under normal conditions
- **"Up to" / burst ceiling**: available for short periods, subject to availability

For the instances tested:

| Instance | Baseline | Burst ceiling |
|----------|----------|--------------|
| m5n.xlarge | 4.1 Gbps | ~16 Gbps |
| m8i.xlarge | 1.8 Gbps | ~12.5 Gbps |
| m8g.xlarge | 1.8 Gbps | ~12.5 Gbps |
| m5.xlarge | 1.25 Gbps | ~10 Gbps |
| m6i.xlarge | 1.56 Gbps | ~12 Gbps |
| m6g.xlarge | 1.25 Gbps | ~10 Gbps |

The n-series (m5n) costs more but guarantees 4.1 Gbps and has the highest burst ceiling (~16 Gbps). The newer-generation m8i has a lower baseline (1.8 Gbps) and a lower burst ceiling (12.4 Gbps), but costs less. The practical questions: how far can you push these instances before AWS enforces limits, how reliable is burst in practice, and does the baseline guarantee actually matter for your workload?

---

## Test setup

| Role | Instance | Count |
|------|----------|-------|
| Targets under test | m5n.xlarge, m8i.xlarge, m8g.xlarge, m5.xlarge, m6i.xlarge, m6g.xlarge | 1 each |
| Load generator (client) | c5n.xlarge | 1 for each target test |

* All instances in eu-central-1, same AZ
* MTU 9001 (jumbo frames) on all instances — confirmed via `netstat -i` on both client and target. MSS 8,961 bytes (9001 − 20 IP − 20 TCP) used for all byte calculations
* iperf3 in reverse mode (`-R`) to stress **target outbound** — the direction that matters for Kafka (broker → consumers), 16 parallel streams
* Duration: 30 minutes per test, counters reset to 0 via reboot before each test
* Monitoring: `ethtool -S <iface> | grep bw_out_allowance_exceeded` sampled every 60 seconds
* TCP retransmit snapshots: `/proc/net/snmp` captured immediately before and after selected tests (Phase 2 only)

---

## Results

### Phase 1: m5n, m8i, m8g

Burst ceiling observed at test start when 16 Gbps was requested, enforced by the hypervisor without drawing too much attention to `bw_out_allowance_exceeded` values.

| Instance | Baseline | Burst ceiling | Load | Queued packets | Queued % | RetransSegs | Retransmit rate | Avg throughput |
|----------|----------|--------------|------|---------------|----------|-------------|-----------------|----------------|
| m5n | 4.1 Gbps | ~16 Gbps | 1.5 Gbps | 10,016 | 0.03% | — | — | 1.54 Gbps |
| m5n | 4.1 Gbps | ~16 Gbps | 6.0 Gbps | 220,009 | 0.15% | — | — | 6.00 Gbps |
| m5n | 4.1 Gbps | ~16 Gbps | 12.0 Gbps | 9,188,543 | 3.05% | — | — | 12.0 Gbps |
| m5n | 4.1 Gbps | ~16 Gbps | 16.0 Gbps | 95,484,189 | 23.8% | — | — | 16.0 Gbps |
| m8i | 1.8 Gbps | 12.4 Gbps | 1.5 Gbps | 5,384,336 | 14.3% | — | — | 1.54 Gbps |
| m8i | 1.8 Gbps | 12.4 Gbps | 6.0 Gbps | 25,271,141 | 16.8% | — | — | 6.00 Gbps |
| m8i | 1.8 Gbps | 12.4 Gbps | 12.0 Gbps | 47,255,050 | 15.7% | 373 | 0.00027% | 12.0 Gbps |
| m8i | 1.8 Gbps | 12.4 Gbps | 16.0 Gbps† | 73,248,713 | 23.5% | 97 | 0.00015% | 12.4 Gbps |
| m8g | 1.8 Gbps | 12.4 Gbps | 1.5 Gbps | 8,203,667 | 21.8% | — | — | 1.54 Gbps |
| m8g | 1.8 Gbps | 12.4 Gbps | 6.0 Gbps | 33,807,772 | 22.4% | — | — | 6.00 Gbps |
| m8g | 1.8 Gbps | 12.4 Gbps | 12.0 Gbps | 49,752,407 | 16.5% | — | — | 12.0 Gbps |
| m8g | 1.8 Gbps | 12.4 Gbps | 16.0 Gbps† | 71,231,425 | 22.9% | — | — | 12.4 Gbps |

> †Requested 16 Gbps, hypervisor hard-capped at 12.4 Gbps. Queued % calculated against 12.4 Gbps actual — TCP snapshots not captured for these tests.
{: .prompt-info }

All three instances sustained full requested throughput at every load level — m5n up to 16 Gbps, m8i/m8g up to their 12.4 Gbps hard cap. Baseline enforcement was **never triggered** on any Phase 1 instance across any 30-minute test.

### Phase 2: m5, m6i, m6g

> Added to test older, more densely populated instance pools where network bandwidth likely to be more utilized. Since enforcement chances are higher, TCP retransmit snapshots captured for all tests.
{: .prompt-info }

Burst ceilings at test start: m5 ~9.95 Gbps, m6i ~12.0 Gbps, m6g ~9.95 Gbps — matching AWS advertised "up to" values. All three burst to ceiling initially, then collapsed to near-baseline once burst credits exhausted — timing varied by load and host conditions (see Key Observations).

| Instance | Baseline | Burst ceiling | Load | Queued packets | Queued % | RetransSegs | Retransmit rate | Avg throughput |
|----------|----------|--------------|------|---------------|----------|-------------|-----------------|----------------|
| m5 | 1.25 Gbps | ~10 Gbps | 1.5 Gbps | 13,214,846 | 35.1% | 0 | 0% | ~1.5 Gbps |
| m5 | 1.25 Gbps | ~10 Gbps | 6.0 Gbps | 46,354,237 | 48.7% | 592 | 0.00051% | **3.79 Gbps** |
| m5 | 1.25 Gbps | ~10 Gbps | 12.0 Gbps | 39,625,737 | 100.5% | 1,502 | 0.0038% | **1.57 Gbps** |
| m6i | 1.56 Gbps | ~12 Gbps | 1.5 Gbps | 4,684,264 | 12.4% | 2 | ~0% | ~1.5 Gbps |
| m6i | 1.56 Gbps | ~12 Gbps | 6.0 Gbps | 24,728,060 | 49.5% | 729 | 0.00052% | **1.99 Gbps** |
| m6i | 1.56 Gbps | ~12 Gbps | 12.0 Gbps | 38,992,465 | 78.0% | 411 | 0.00082% | **1.99 Gbps** |
| m6g | 1.25 Gbps | ~10 Gbps | 1.5 Gbps | 14,119,475 | 37.5% | 0 | 0% | ~1.5 Gbps |
| m6g | 1.25 Gbps | ~10 Gbps | 6.0 Gbps | 45,711,546 | 113.6% | 165 | 0.00014% | **1.60 Gbps** |
| m6g | 1.25 Gbps | ~10 Gbps | 12.0 Gbps | 40,208,730 | 100.2% | 1,014 | 0.0025% | **1.60 Gbps** |

**Bold avg throughput = baseline enforcement observed.** Once enforced, pushing additional load changes nothing — m5 at both 6G and 12G settles near at ~1.25 Gbps baseline. Note that m5 6G avg throughput (3.79 Gbps) is higher than 12G (1.57 Gbps) because enforcement kicked in later at the lower load — burst exhausted at ~964s at 6G vs ~97s at 12G.

---

### Key Observations

**1. AWS delivers exactly what it advertises — both burst ceiling and baseline.**

Every instance hit its "up to X Gbps" ceiling at test start and, where sustained load exceeded burst capacity, throttled to its advertised baseline. m5/m6g settled at ~1.57-1.60 Gbps (advertised 1.25 Gbps — slight premium from residual burst credits), m6i at ~1.99 Gbps (advertised 1.56 Gbps). The numbers are honest, the documentation simply does not define how long burst lasts.


**2. Burst duration is non-deterministic.**

The same instance type under the same load triggered enforcement at different times across repeated tests — m5 at 6G enforced at ~964s in one run, held for the full test in another on the same day. m6i held 12G for 5 minutes clean in a validation run after enforcing in a 30-minute test. Attempts to model burst as a simple token bucket (fixed bucket size draining at excess rate above baseline) failed — enforcement timing at 6G vs 12G was not proportional to the excess rate. Host conditions, tenant density, and available physical NIC capacity are the likely variables, but none are observable from inside the instance. **The only plannable number is the baseline.**

**3. Baseline enforcement and the hard cap both happen at the hypervisor layer — TCP handles it cleanly.**

When m5/m6i/m6g were throttled to baseline, TCP retransmit rate stayed near-zero (max 0.0038%). TCP congestion control is designed to detect network pressure and voluntarily reduce the send rate — it adds delay between sends rather than pushing packets that will be dropped. Modern algorithms like BBR use rising RTT (caused by hypervisor queuing) as the backoff signal, backing off before packets are ever dropped. This means no retransmits, no dropped connections, no errors. The sender simply slows down and the kernel absorbs the enforcement silently.

**4. bw_out_allowance_exceeded has value when normalised to packet rate.**

> Raw counts are misleading — a freshly booted m5.xlarge showed `bw_in_allowance_exceeded` at 474 before any workload, and enforcement and free-flow states produced counts in the same order of magnitude. But when expressed as a percentage of total outbound packets, a pattern emerges from the data:
{: .prompt-info }

| State | Example | Queued % |
|-------|---------|----------|
| Free-flow, below baseline | m5 at 1.5G | 35.1% |
| Free-flow, at or near ceiling | m5n at 16G | 23.8% |
| Enforced to baseline | m5 at 12G | 100.5% |
| Enforced to baseline | m6g at 6G | 113.6% |

The queue acts as a buffer between the instance and the hypervisor — its fill rate is a direct measure of how much the hypervisor is willing to pass through at any given time. When the fill rate approaches or exceeds 100%, the hypervisor is the bottleneck. This scales correctly across instance types: a higher baseline simply means the hypervisor drains the queue faster, so the fill rate stays low under normal load. A sudden spike in the normalised rate — regardless of instance type — indicates the send rate is outpacing what the hypervisor will pass through.

Example calculation (m5 at 12G, enforced):
```
Test duration:       1800s
Avg throughput:      1.57 Gbps
Total bytes sent:    1.57 × 10⁹ × 1800 / 8  = ~353 GB
MSS:                 8,961 bytes
Total packets sent:  353 × 10⁹ / 8,961       = ~39,396,000
Queued packets:      39,625,737               (final counter value, reset to 0 via reboot before test)
Queued %:            39,625,737 / 39,396,000  = 100.5%
```

All enforced tests in this benchmark crossed 50% queued rate, all free-flow tests stayed below 50%. The [AWS documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-network-performance-ena.html) states the metric counts packets "queued or dropped" and the [AWS networking blog](https://aws.amazon.com/blogs/networking-and-content-delivery/amazon-ec2-instance-level-network-performance-metrics-uncover-new-insights/) recommends upsizing when it increments at a "noticeable rate" — our data provides the quantification AWS could not: normalised to outbound packet rate, **~50% is a threshold worth alerting on**. Whether the mechanism is packet drop or TCP congestion backoff is irrelevant — either way throughput is being constrained.

**5. m8i and m8g baseline was never enforced in any 30-minute test.**

Both sustained full throughput at every load level up to their 12.4 Gbps hard cap. Whether this reflects lower host density on newer pools, a larger burst credit bucket, or a different enforcement policy is not documented by AWS and cannot be determined from instance-level metrics. The result may not be representative of production m8 instances under sustained high load over longer periods or on a busier host.

**6. m8g and m8i are equivalent in throughput delivery.**

Both instances behaved identically at every load level — same throughput, same hard cap at 12.4 Gbps, same enforcement
behaviour. Queued packet counts varied between the two at lower load levels but the difference was not consistent across all tests and had no application-level consequence.

---

## Conclusion

Three things this benchmark established clearly:

**AWS delivers exactly what it publishes.** Every instance hit its advertised burst ceiling at test start and, where load exceeded burst capacity, enforced its advertised baseline. No surprises in either direction — the numbers on the spec sheet are real.

**`bw_out_allowance_exceeded` is not useful as a raw count, but is a valid signal when normalised.** Raw counts are not
comparable across load levels or instance types — the counter fires from boot and produces similar magnitudes in both free-flow and enforced states. Normalised to outbound packet rate, it becomes meaningful: all enforced tests crossed 50%, all free-flow tests stayed below. The AWS documentation is right that it is worth monitoring, the gap is that "noticeable rate" is never quantified. Our data suggests ~50% normalised rate as a starting threshold, to be calibrated for your own workload.

**Burst duration is non-deterministic and likely pool-density dependent.** Older instance pools (m5, m6i, m6g) triggered enforcement unpredictably — same instance, same load, different enforcement times across runs. Newer pools (m8i, m8g) never triggered enforcement in any 30-minute test. The most plausible explanation is host density: newer pools are less saturated, leaving more physical NIC headroom. As those pools fill up, burst behaviour will likely converge toward what the older pools show today. The baseline is the only number you can plan against long-term.

**Choosing between instance types** comes down to how much burst uncertainty is acceptable and over what time frame:

- **m8i.xlarge** — lower cost, 12.4 Gbps hard cap, 1.8 Gbps baseline. Burst held for 30+ minutes in our tests but enforced within minutes on equivalent older-generation instances under the same load. The baseline (1.8 Gbps) is the only number you can plan against long-term.
- **m5n.xlarge / m6in.xlarge** (n-series) — higher cost, 4.1 Gbps guaranteed baseline under normal conditions. The right choice if the requirement is a reserved network pipe without relying on burst availability.

For TCP workloads, baseline enforcement causes a temporary throughput reduction, not data loss — TCP congestion control backs off cleanly before generating retransmits, as confirmed by near-zero retransmit rates throughout all enforcement events in this benchmark. The practical operational impact depends on the workload: if send rate exceeds the enforced ceiling, receivers fall behind and data accumulates. Whether that is acceptable depends on latency tolerance and available buffer capacity.

### m8g vs m8i

Network benchmark results between m8i and m8g were comparable — queued packet differences at lower load levels may reflect host conditions at the time of testing rather than a systematic difference. There is no network-based reason to prefer m8i over m8g. If Graviton cost savings are a factor, m8g is a reasonable choice for this workload.

### m8in

Sadly, during my tests this type was not available in my target region due to capacity issues.

---

## Appendix: ENA Allowance Metrics Interpretation Guide

The [AWS documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-network-performance-ena.html) describes the ENA allowance metrics as counting packets "queued or dropped" by the ENA traffic shaping mechanism, and the [networking blog](https://aws.amazon.com/blogs/networking-and-content-delivery/amazon-ec2-instance-level-network-performance-metrics-uncover-new-insights/) recommends upsizing when they increment at a "noticeable rate." The gap is that "noticeable rate" is never quantified, and raw counts are not comparable across load levels or instance types. Normalised to packet rate, the metrics become meaningful.

**Available ENA allowance metrics:**

| Metric | Direction | When it fires |
|--------|-----------|--------------|
| `bw_out_allowance_exceeded` | Outbound | TX rate exceeds instance bandwidth limit |
| `bw_in_allowance_exceeded` | Inbound | RX rate exceeds instance bandwidth limit |
| `pps_allowance_exceeded` | Both | Packet rate (TX+RX combined) exceeds PPS limit |
| `conntrack_allowance_exceeded` | Both | Connection tracking table is full |
| `linklocal_allowance_exceeded` | Both | Link-local traffic (DNS, IMDS, NTP) rate exceeded |

For most workloads `bw_out_allowance_exceeded` is the primary signal — but `bw_in_allowance_exceeded` becomes equally important during events that generate heavy inbound traffic, such as partition rebalancing or follower catch-up replication after a node recovery. `pps_allowance_exceeded` is relevant for high packet-rate workloads with small messages. `conntrack_allowance_exceeded` and `linklocal_allowance_exceeded` indicate different problems unrelated to bandwidth.

**Normalised to packet rate query examples for Grafana:**

```
# Outbound (TX-heavy workloads)
rate(node_ethtool_bw_out_allowance_exceeded{device="ens5"}[5m])
/
rate(node_network_transmit_packets_total{device="ens5"}[5m]) * 100

# Inbound (RX-heavy — replication catch-up, data redistribution)
rate(node_ethtool_bw_in_allowance_exceeded{device="ens5"}[5m])
/
rate(node_network_receive_packets_total{device="ens5"}[5m]) * 100
```

> `node_ethtool_*` metrics require node_exporter to be started with `--collector.ethtool` — this collector is not enabled by default. Verify it is collecting with `curl -s localhost:9100/metrics | grep allowance` before relying on these queries.
{: .prompt-tip }

> The `device` label value varies by kernel version and instance generation — `ens5`, `eth0`, and other names are all
> possible, and the same instance type in the same pool may not be consistent. Verify with `ip link` or check what your nodeexporter is actually reporting before using these queries.
{: .prompt-warning }

From this benchmark (outbound only): all free-flow tests stayed below 50% queued rate, all enforced tests crossed 50%. This suggests ~50% as a starting point for an alert threshold, but these tests covered a limited set of instance types under specific load conditions at a specific time of day. The right threshold will vary by workload, instance type, and host density — treat this as a directional finding and calibrate against your own baseline before acting on it.

**Detecting enforcement:**

There is no dedicated instance-level metric that signals baseline enforcement. Both mechanisms observed in this benchmark — gradual throttling to baseline (m5/m6i/m6g) and hard cap (m8i/m8g at 12.4 Gbps) — occurred at the hypervisor layer. The normalised allowance rate crossing ~50% is the earliest available signal, `node_network_transmit_bytes_total` or `node_network_receive_bytes_total` flattening while load remains stable confirms it.

**TCP retransmit rate:**

TCP retransmit rate is a standard network health metric independent of cloud infrastructure — it reflects packet loss from any cause. Normalised to outgoing segments:

```
rate(node_netstat_Tcp_RetransSegs[5m])
/
rate(node_netstat_Tcp_OutSegs[5m]) * 100
```

General guidance: below 0.01% is normal for datacenter TCP, above ~0.1% warrants investigation, above 1% indicates a real problem. In the context of bandwidth enforcement specifically, the two signals together tell the full story: a spike in normalised allowance rate alone indicates the hypervisor is queuing (TCP backs off via RTT, no retransmits), a spike in both allowance rate and retransmit rate at the same time indicates the hypervisor queue itself filled and packets were dropped. In this benchmark, retransmit rate peaked at 0.0038% under full sustained enforcement — consistent with queuing only, the drop path was not reached.

**Further reading:**

- [AWS re:Post — Why does my EC2 instance exceed its network limits when my average utilization is low?](https://repost.aws/knowledge-center/ec2-instance-exceeding-network-limits) — Official AWS article on the same ENA metrics. Confirms the counter does not differentiate between queued and dropped packets. A comment from an AWS expert clarifies the distinction: for TCP, queuing raises RTT and congestion control backs off, drops produce retransmissions. Our near-zero retransmit rates under full sustained enforcement are consistent with the hypervisor queuing rather than dropping — we only triggered the queuing path. The drop path exists when the hypervisor queue itself fills, but TCP's proactive backoff (BBR uses RTT elevation as the signal) prevented us from reaching it. The article also notes that some queuing is expected even with BBR, which explains why instances running well below baseline still show non-zero allowance counter values. The article covers microburst scenarios (sub-second spikes invisible to 1-minute averages) — a distinct case from sustained throughput enforcement that our benchmark did not cover.

- [Pinterest Engineering — Handling network throttling with AWS EC2](https://medium.com/@Pinterest_Engineering/handling-network-throttling-with-aws-ec2-at-pinterest-fda0efc21083) — Pinterest observed TCP retransmits alongside elevated ENA allowance counters, which appears to contradict our near-zero retransmit finding. Testing methodology and client behaviour are not disclosed in detail, and retransmits can have many causes beyond hypervisor drops — client-side behaviour alone is sufficient to produce them without any packets being dropped at the hypervisor. Neither article uses normalised rates, the ~50% threshold is not referenced elsewhere.