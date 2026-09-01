# Eviction policy for MoE expert caching — measured

Trace: OLMoE-1B-7B (64 experts/layer, 8 active, 16 layers), 160 generated
tokens across 5 prompts, 20,480 real expert requests over 1,007 distinct
(layer, expert) slots. Router hooks on the live model — not a simulation of
the workload, only of the cache.

Expert size 14 MB (bf16). NVMe measured at 5.6 GB/s.

## Hit rate by policy and cache size

| cache | LRU | LFU | segmented | random | Belady (optimal) |
|------:|----:|----:|----------:|-------:|-----------------:|
| 8     | 0.0% | 1.8% | 1.9% | 0.0% | 5.4% |
| 16    | 0.0% | 3.2% | 3.1% | 0.0% | 11.5% |
| 32    | 0.0% | 9.1% | 5.6% | 0.8% | 22.9% |
| 64    | 0.0% | **17.0%** | 10.3% | 6.7% | 40.2% |
| 128   | 30.1% | 28.0% | 18.1% | 22.9% | 57.9% |
| 256   | 53.8% | 46.3% | 48.6% | 45.2% | 74.0% |

## The finding

**LRU returns a 0% hit rate at every cache size below 128, and LRU is the
default choice.**

The cause is structural. Each token walks all 16 layers in the same fixed
order, touching 16 x 8 = 128 distinct slots per cycle. A cache smaller than
one full cycle is completely evicted before the cycle comes back around --
textbook LRU thrashing on a sequential scan. Recency is exactly the wrong
signal when access is cyclic.

Frequency is the right signal, because reuse is real but not recent: the
hottest 128 slots serve 31.2% of all requests, and consecutive tokens share
41.8% of their experts on average.

**LFU beats LRU by an unbounded margin below the cycle length**, and by a
smaller margin above it (where it loses slightly, 46.3% vs 53.8% at 256).

## What did not work

A segmented cache -- pin the hottest N/2 slots by frequency, run LRU on the
remainder -- underperformed plain LFU everywhere (10.3% vs 17.0% at cache
64). Pinning on first-half frequency was too coarse; it protected globally
hot experts while starving the working set. Recorded because it looked
obviously correct beforehand and was not.

## Headroom

Belady's optimum reaches 40.2% at cache 64 against LFU's 17.0%. More than
half the achievable hit rate is still unclaimed by any online policy tested,
which is where router-lookahead prefetch would apply: the router for layer L
resolves before layer L+1 executes, so the next requests are knowable
slightly before they are needed.

## Caveats

* One model, one architecture, greedy decoding, 5 prompts. Expert routing
  is input-dependent and these prompts are short.
* Cache is simulated; the trace is real. No PCIe, kernel-launch or
  allocator overhead is modelled, so hit rate is the measure here, not
  wall-clock.
* 160 tokens is a short trace. Long-context routing behaviour may differ.

## Method

The measurement harness that produced these numbers is not public. The
methodology, the failed approaches, and the reasoning behind each choice are
part of paid engagements — see https://georgemoncure.github.io/

If you want a figure here checked against your own hardware, get in touch and
I will run it.
