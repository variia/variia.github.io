---
title: "EBS gp3 vs Local NVMe: When Network Storage Wins"
date: 2025-05-18 17:00:00 +0100
author: Iván Ádám Vári
description: A practical guide to evaluating EBS gp3 against local NVMe SSDs. Covers I/O profiling, fio benchmarking with realistic block sizes, latency analysis, right-sizing provisioned IOPS and throughput, and the cost model that makes over-provisioning expensive.
keywords: aws,ebs,gp3,nvme,storage,benchmark,fio,iops,throughput,latency,provisioning
categories:
  - AWS
tags:
  - aws
  - cloud
  - storage
  - ebs
  - performance
---

The common assumption is that local NVMe SSDs are always faster than EBS. That assumption is wrong — or at least, it depends
entirely on your I/O profile and the EC2 instance generation you are running.

This post walks through a real evaluation of EBS gp3 versus local NVMe for a write-heavy database workload. The results were
surprising: **EBS gp3 outperformed local NVMe on newer instance types across all metrics**. More importantly, the post explains
*why* block size matters for benchmarking, how to determine your own I/O profile, and how to right-size gp3 provisioning without
wasting money.

> This evaluation applies to general-purpose, memory and cpu optimized instance families (m, r, c-series). Storage-optimized instances
(i3, i4i, is1, im4gn, i8, etc) are purpose-built around local NVMe — EBS is not a substitute for those.
{: .prompt-info }

> **Update March 2026**: AWS has [significantly increased gp3 volume limits](https://aws.amazon.com/about-aws/whats-new/2025/09/amazon-ebs-size-provisioned-performance-gp3-volumes/)
— IOPS from 16,000 to **80,000**, throughput from 1,000 MiB/s to **2,000 MiB/s**, and volume size from 16 TiB to 64 TiB. These new limits do not
invalidate the test results below but they significantly expand the range of workloads where EBS `gp3` is viable — including read-heavy and
low-residency workloads that were previously only feasible on either local NVMe or the more expensive `io1` and `io2` EBS volumes.
{: .prompt-info }

> The new limits make it very easy to over-provision and incur significant costs. Maxing IOPS to 80,000 alone adds **$385/month per volume** 
— nearly 5x the storage cost of a 1 TB volume. Across a cluster, this can add thousands per month. **Always [right-size based on your I/O profile](#step-1-determine-your-applications-io-block-size)**, 
not by defaulting to maximum values. See the [cost breakdown](#ebs-cost-model) below.
{: .prompt-danger }

---

## When does EBS gp3 make sense?

Not every workload benefits from EBS. Whether gp3 is viable depends on your application's I/O profile:

| Workload characteristic | EBS gp3 viable? | Recommendation |
|------------------------|-----------------|----------------|
| High data residency, write-dominant | Yes | EBS gp3 — this post's tested scenario |
| High data residency, read-dominant | Yes | Reads served from RAM anyway |
| Low residency, write-dominant | Likely yes | Benchmark — writes are async, latency tolerant |
| Low residency, read-dominant | Benchmark first | Test at realistic block sizes, compare clat p99 vs local NVMe |
| Dataset far exceeds RAM, high read IOPS (>50K) | Caution | Local NVMe may still be necessary for latency and most likely cheaper over all|

The fundamental weakness of EBS compared to local NVMe is **small-block random read latency**. EBS reads traverse the network
to reach the storage backend — local NVMe does not. This difference is masked when data is served from RAM but becomes critical
when it is not.

If your application keeps its working set in memory (high cache hit ratio / high data residency), disk reads are limited to
background operations like replication, compaction, and index maintenance. These are not latency-sensitive. In this scenario,
EBS is not just viable — it can be *faster* than local NVMe on newer instance types, as the benchmarks below demonstrate.

If your application regularly fetches data from disk on the client request path (low residency), every read becomes
latency-sensitive and local NVMe has the advantage. With the new gp3 limits (80,000 IOPS), EBS can now match the IOPS demand
of many read-heavy workloads, but **latency (+ costs) remains a concern** — benchmark with realistic block sizes and compare
[clat percentiles](#interpreting-latency-clat-results) before assuming the new limits make EBS viable for your read-heavy profile.

---

## Why block size matters for benchmarking

> Applications don't expose I/O block size configuration — the storage engine determines it internally. Running `fio` with default 4K blocks may 
produce misleading results because the actual application I/O pattern is almost always different.
{: .prompt-warning }

The relationship between IOPS and throughput depends entirely on how large each I/O operation is. The same application can look
IOPS-bound or throughput-bound depending on its block size:

```
Throughput = IOPS x Block Size

Example:
  10,000 IOPS x 4 KB  =  40 MiB/s   (IOPS-bound, throughput is low)
  10,000 IOPS x 64 KB = 640 MiB/s   (throughput-bound, IOPS is low)
```

### Deriving your I/O profile from production metrics

Look at your monitoring dashboards (Grafana, CloudWatch, Datadog, etc.) for two disk metrics on the same server, over the same
time period:

- **Disk Write IOPS** (or Read IOPS)
- **Disk Write Bytes/sec** (or Read Bytes/sec)

Then calculate:

```
Average Block Size = Bytes per second / IOPS
```

For example, if you see 8,000 write IOPS and 500 MiB/s write throughput:

```
500 MiB / 8,000 = ~64 KB per write operation
```

> Do this for both reads and writes — they often differ. Reads may be small (4-16K) while writes may be large (64-256K), or vice versa, depending on the application.
{: .prompt-tip }

The workload (for our use case: in-memory NoSQL server ) tested in this post had the following profile observed over a 90+ day production period:

- **Writes**: ~10,000 IOPS at ~500 MiB/s = **~50-64 KB per write operation**
- **Reads**: Low baseline IOPS with large byte spikes — large sequential reads (replication, compaction), not small random reads
- **Read source**: Nearly all application reads served from RAM (~100% resident), disk reads were background operations

---

## Test methodology

Based on the production I/O profile, fio was configured with mixed block sizes — small random reads (4-16K) and fixed 64K writes:

```bash
fio --directory=<mount> --name fio_rnd_rwmix --randrepeat=1 \
    --ioengine=libaio --direct=1 --bsrange=4k-16k,64k-64k \
    --iodepth=64 --size=2G --readwrite=randrw --rwmixread=33 \
    --numjobs=16 --time_based --runtime=900 --group_reporting
```

| Parameter | Value | Reason |
|-----------|-------|--------|
| `ioengine=libaio` | Linux native async I/O | Realistic for database workloads |
| `direct=1` | Bypass OS page cache | Most databases manage their own caching |
| `bsrange=4k-16k,64k-64k` | Mixed read/write sizes | Matches observed I/O pattern |
| `iodepth=64` | Deep queue | Realistic for busy database under peak load |
| `numjobs=16` | 16 parallel workers | Simulates concurrent application threads |
| `rwmixread=33` | 33% read, 67% write | Reflects write-heavy production workload |
| `runtime=900` | 15 minutes | Sustained test to eliminate burst/cache effects |
| `group_reporting` | Aggregate all jobs | Single summary output |

> All EBS volumes provisioned at maximum gp3 settings at the time of testing: 16,000 IOPS, 1,000 MiB/s throughput.
{: .prompt-info }

All tests ran on identically configured instances, repeated multiple times for consistency and reproducibility.

---

## Results

### Impact of block size on benchmarks

This is why using the correct block size matters. Both tests ran on an r5d.4xlarge (EBS bandwidth: 4,750 Mbps):

**4K block size (fio default):**

| Metric | Local NVMe | EBS gp3 |
|--------|-----------|---------|
| Read BW | 221 MiB/s | 20.7 MiB/s |
| Write BW | 449 MiB/s | 42.1 MiB/s |
| Read IOPS | 56,000 | 5,302 |
| Write IOPS | 114,000 | 10,700 |

With 4K blocks, NVMe dominates because EBS is capped at 16K IOPS and each operation is tiny.

**Production-like block sizes (mixed: 4k-16k read, 64k write):**

| Metric | Local NVMe | EBS gp3 |
|--------|-----------|---------|
| Read BW | 249 MiB/s | 188 MiB/s |
| Write BW | 505 MiB/s | 381 MiB/s |
| Read IOPS | 4,056 | 3,043 |
| Write IOPS | 8,044 | 6,018 |

With realistic block sizes, the gap narrows significantly. EBS is still slower here, but the bottleneck is the **instance-level
EBS bandwidth limit of 4,750 Mbps**, not EBS itself.

### Newer instance generation removes the bottleneck

Testing on r6id.4xlarge (EBS bandwidth: 10 Gbps) with production-like block sizes:

| Metric | Local NVMe | EBS gp3 | Delta |
|--------|-----------|---------|-------|
| Read BW | 286 MiB/s | **333 MiB/s** | +16% EBS |
| Write BW | 581 MiB/s | **675 MiB/s** | +16% EBS |
| Read IOPS | 4,601 | **5,477** | +19% EBS |
| Write IOPS | 9,233 | **10,800** | +17% EBS |

**EBS gp3 outperforms local NVMe on the newer instance across all metrics.**

### Latency comparison (15-minute sustained test)

Latency measured on r6id.4xlarge under full saturation with `iodepth=64` and 16 concurrent jobs:

**Read completion latency (clat):**

| Percentile | Local NVMe | EBS gp3 | Delta |
|------------|-----------|---------|-------|
| p50 | 61 ms | 57 ms | -4 ms |
| p90 | 92 ms | 97 ms | +5 ms |
| p95 | 104 ms | 110 ms | +6 ms |
| p99 | 120 ms | 133 ms | +13 ms |
| p99.5 | 125 ms | 142 ms | +17 ms |
| p99.9 | 134 ms | 159 ms | +25 ms |
| p99.99 | 180 ms | 180 ms | 0 ms |

**Write completion latency (clat):**

| Percentile | Local NVMe | EBS gp3 | Delta |
|------------|-----------|---------|-------|
| p50 | 74 ms | 60 ms | -14 ms |
| p90 | 106 ms | 101 ms | -5 ms |
| p95 | 118 ms | 113 ms | -5 ms |
| p99 | 133 ms | 136 ms | +3 ms |
| p99.5 | 138 ms | 146 ms | +8 ms |
| p99.9 | 148 ms | 163 ms | +15 ms |
| p99.99 | 197 ms | 186 ms | -11 ms |

**Key observations:**

- Median (p50) latency is actually **better on EBS** for both reads and writes
- Tail latency (p99.9) is ~15-25 ms higher on EBS — negligible for applications with async disk persistence
- p99.99 is comparable or better on EBS
- For workloads where data is served from RAM, application read latency is unaffected — disk latency only affects background operations

---

## Key takeaways

1. **Block size is critical for benchmarking** — testing with default 4K blocks makes EBS look 10x worse than NVMe, testing with realistic block sizes shows EBS is actually faster on newer instances
2. **Instance-level EBS bandwidth is often the real bottleneck** — older instance generations (r5, m5, c5) cap at ~593 MiB/s, upgrading the instance generation can be more impactful than changing storage type
3. **EBS gp3 can outperform local NVMe on throughput** — on r6i/r7i class instances with 10+ Gbps EBS bandwidth
4. **Latency is comparable under saturation** — tail latency differences of ~15ms are irrelevant for async persistence with high-residency workloads
5. **Operational benefits matter** — EBS volumes survive instance stop/start, local NVMe SSDs require full data recovery on instance maintenance events

---

## Guide to right-sizing EBS gp3 provisioning

This section is a general guide for any workload. It describes how to determine the right gp3 IOPS and throughput settings based
on your application's I/O profile.

> While the examples use AWS technology, the techniques described here are generic and should work on any cloud vendor with network-attached block storage.
{: .prompt-info }

### The problem with over-provisioning

gp3 charges separately for IOPS and throughput above the baseline (3,000 IOPS / 125 MiB/s). Setting every volume to maximum
"just in case" wastes money. Most applications don't need anywhere near the limits. The key is figuring out what your
application actually requires.

### Step 1: Determine your application's I/O block size

The relationship between IOPS and throughput depends entirely on how large each I/O operation is:

```
Throughput = IOPS x Block Size
```

Use your monitoring dashboards to calculate the average block size as described in
[Why block size matters](#why-block-size-matters-for-benchmarking) above.

### Step 2: Identify peak usage, not average

Look at your metrics during **peak hours**, not averages. Provision for the peak with some headroom (~20-30%) so the disk is
never the bottleneck. An undersized disk during peak causes latency spikes and queue buildup.

From your dashboards, note:

- **Peak write IOPS** and **peak write throughput**
- **Peak read IOPS** and **peak read throughput**

### Step 3: Determine which dimension is your bottleneck

gp3 has two independent limits — IOPS and throughput. Depending on your block size, one will be the constraint:

| Block Size | Bottleneck | Example |
|-----------|-----------|---------|
| Small (4-16 KB) | IOPS | Databases with small random lookups |
| Medium (32-64 KB) | Either | Depends on the workload |
| Large (128-256 KB) | Throughput | Streaming, logging, sequential writes |

**Small block sizes (4-16 KB):** You'll hit the IOPS limit before the throughput limit. Provision IOPS to match your peak.
Throughput can stay at baseline.

```
Example: 12,000 IOPS x 8 KB = 96 MiB/s
  Provision: 12,000 IOPS / 125 MiB/s (baseline throughput is enough)
```

**Large block sizes (64-256 KB):** You'll hit the throughput limit before IOPS. Provision throughput to match your peak. IOPS
can stay lower.

```
Example: 5,000 IOPS x 128 KB = 640 MiB/s
  Provision: 5,000 IOPS / 640 MiB/s (IOPS baseline of 3,000 is almost enough)
```

### Step 4: Account for instance-level EBS bandwidth

Even if you provision gp3 at high values, the EC2 instance type has its own EBS bandwidth cap. Check the
[AWS documentation](https://docs.aws.amazon.com/ec2/latest/instancetypes/gp.html) for your instance type's EBS bandwidth.

| Instance Generation | Typical EBS Bandwidth |
|--------------------|----------------------|
| r5 / m5 / c5 | 4,750 Mbps (~593 MiB/s) |
| r6i / m6i / c6i | 10,000 Mbps (~1,250 MiB/s) |
| r7i / m7i / c7i | 10,000+ Mbps |

There is no point provisioning beyond what the instance can deliver. If your instance caps at 593 MiB/s, provisioning 1,000
MiB/s on gp3 wastes money.

### Step 5: Validate with fio

Before deploying, validate your provisioning with `fio` using block sizes that match your application (see
[Test methodology](#test-methodology) above). Key points:

- **Never benchmark with default 4K blocks** unless your application actually does 4K I/O
- Use `--direct=1` to bypass OS cache
- Run tests long enough to exhaust any burst credits (at least 20-30 minutes)
- Use `--group_reporting` to get aggregated results (especially for `clat` stats)
- Compare IOPS and throughput against your provisioned limits — you should see the volume hitting the limits you set, not the instance bandwidth limit

Provision only what you need based on your peak I/O profile. The baseline is generous enough for many workloads — only increase
when your metrics show you are hitting the limit.

> You can freely adjust EBS throughput and IOPS values even on mounted/in-use volumes, at the cost of some time, may vary (this maybe AWS specific).
{: .prompt-tip }

### Interpreting latency (clat) results

When running fio with `--group_reporting`, you get completion latency (clat) percentile tables. These numbers depend heavily on
test parameters — particularly `iodepth` and `numjobs` — so absolute values are only meaningful in context.

**What clat measures:** The time from when an I/O request is submitted to the device until it completes. Higher queue depths mean
more requests waiting, so latency increases. A p99 of 100ms under `iodepth=64` with 16 jobs is not the same as p99 of 100ms
under `iodepth=1`.

**General guidance for database workloads under saturated conditions** (high iodepth, many concurrent jobs — worst case):

| Percentile | What it tells you | Acceptable range | Concern threshold |
|------------|-------------------|------------------|-------------------|
| p50 (median) | Typical request latency | < 100 ms | > 150 ms |
| p99 | Latency for 1-in-100 requests | < 200 ms | > 300 ms |
| p99.9 | Worst-case tail latency | < 250 ms | > 500 ms |
| p99.99 | Extreme outliers | < 500 ms | > 1,000 ms |

**Important caveats:**

- These thresholds assume **saturated conditions** (high iodepth, many jobs). Under light load, latencies should be significantly lower — if you see p50 > 10ms under light load, something is wrong.
- **Async vs sync persistence matters.** Applications with async disk writes tolerate higher disk latency because clients don't wait for disk. Applications where clients block on disk I/O (e.g., synchronous database commits) need much tighter latency — halve the thresholds above.
- **Compare, don't evaluate in isolation.** The primary value of clat percentiles is comparing two storage options under identical test conditions. A 15ms difference at p99.9 between two options is negligible. A 10x difference indicates a real problem.
- **Watch for cliffs, not gradual increases.** A smooth curve from p50 to p99.99 (e.g., 60ms → 80ms → 130ms → 160ms → 180ms) indicates predictable behaviour. A sudden jump (e.g., 60ms → 80ms → 130ms → 160ms → 900ms) at p99.99 indicates occasional stalls — investigate whether it is device-level queuing, instance throttling, or EBS credit exhaustion.

**How to read clat in the context of EBS vs NVMe:**

| Scenario | What it means |
|----------|--------------|
| EBS p50 ≤ NVMe p50 | EBS handles typical load as well or better |
| EBS p99 within 20% of NVMe p99 | Tail latency difference is acceptable for most workloads |
| EBS p99.9 more than 2x NVMe p99.9 | Investigate — may indicate EBS network jitter under load |
| EBS p99.99 significantly worse | Likely sporadic, acceptable unless your application is latency-critical at extreme percentiles |

---

## EBS cost model

gp3 pricing has three independent components (rates as of March 2026, eu-central-1 —
[verify current pricing](https://aws.amazon.com/ebs/pricing/)):

| Component | Rate (per month) | Baseline (included free) |
|-----------|-----------------|-------------------------|
| Storage | $0.08 / GB | — |
| IOPS | $0.005 / IOPS | First 3,000 IOPS |
| Throughput | $0.06 / MB/s | First 125 MB/s |

**1 TB volume — cost by provisioning level:**

| Configuration | Storage | IOPS | Throughput | **Total/month** |
|--------------|---------|------|------------|-----------------|
| Baseline (3K IOPS / 125 MB/s) | $80 | $0 | $0 | **$80** |
| Right-sized example (10K IOPS / 500 MB/s) | $80 | $35 | $22.50 | **$137.50** |
| Old max (16K IOPS / 1,000 MB/s) | $80 | $65 | $52.50 | **$197.50** |
| New max (80K IOPS / 2,000 MB/s) | $80 | $385 | $112.50 | **$577.50** |

**4 TB volume — cost by provisioning level:**

| Configuration | Storage | IOPS | Throughput | **Total/month** |
|--------------|---------|------|------------|-----------------|
| Baseline (3K IOPS / 125 MB/s) | $320 | $0 | $0 | **$320** |
| Right-sized example (10K IOPS / 500 MB/s) | $320 | $35 | $22.50 | **$377.50** |
| Old max (16K IOPS / 1,000 MB/s) | $320 | $65 | $52.50 | **$437.50** |
| New max (80K IOPS / 2,000 MB/s) | $320 | $385 | $112.50 | **$817.50** |

**Multi-node impact** — multiply per-volume cost by node count:

| Cluster size | Baseline (1 TB) | New max (1 TB) | Wasted/month |
|-------------|-----------------|----------------|--------------|
| 4 nodes | $320 | $2,310 | $1,990 |
| 8 nodes | $640 | $4,620 | $3,980 |

The IOPS component dominates the cost at high provisioning levels. Always
[determine your actual I/O profile](#step-1-determine-your-applications-io-block-size) before provisioning — most workloads need
a fraction of the maximum.

---

## Further reading

- [Amazon EBS performance](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-performance.html) — AWS documentation covering EBS performance characteristics, volume types, and instance bandwidth limits
- [Benchmark EBS volumes](https://docs.aws.amazon.com/ebs/latest/userguide/benchmark_procedures.html) — AWS guide to benchmarking EBS with fio
