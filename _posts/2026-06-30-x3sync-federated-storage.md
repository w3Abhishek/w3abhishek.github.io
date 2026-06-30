---
title: "X3Sync: Federating Free Cloud Storage with Zero-Knowledge Encryption"
date: 2026-06-30 12:00:00 +0530
categories: [research, systems]
tags: [zero-knowledge, cloudflare-workers, cloud-storage, encryption, research]
image:
  path: https://i.ibb.co/whw826cD/a9a8533cd3f519ab8928ef5696f16f9a.jpg
  alt: "Server racks in a data center"
description: "X3Sync: a research proof-of-concept federating free-tier cloud storage (Google Drive, Dropbox, Koofr) into a zero-knowledge encrypted namespace on Cloudflare Workers."
---

X3Sync is a research proof-of-concept I built that aggregates free-tier cloud storage (Google Drive, Dropbox, Koofr) into a single zero-knowledge encrypted namespace, coordinated entirely on Cloudflare Workers. Paper is out, preprint is live, MDPI submission is in.

**Preprint DOI:** [10.5281/zenodo.21043094](https://doi.org/10.5281/zenodo.21043094)
**Code:** [github.com/vaultdmn/x3sync-core](https://github.com/vaultdmn/x3sync-core)

---

## The problem

Free-tier cloud storage is everywhere but fragmented. 15GB here, 2GB there, 10GB somewhere else. Nobody combines them, and nobody encrypts them properly before upload either — you're trusting Google/Dropbox/whoever with plaintext access to your files by default.

X3Sync tries to fix both problems at once: aggregate the free tiers into one usable pool, and make sure no provider — or even the coordination layer itself, depending on mode — ever sees your plaintext.

---

## What's actually in it

Four pieces I think are worth talking about:

**A commutative storage model.** Adding a new provider to the pool shouldn't require rebalancing existing data. I formalized this as an append-only, commutative operation — capacity just adds up, no reshuffling.

**A universal provider abstraction.** Google Drive, Dropbox, and Koofr all have wildly different auth flows and APIs. Every provider module implements the same 7-method interface, so adding a new one (OneDrive, say) means writing one module and registering it — zero changes to routing logic.

**Dual-mode decryption.** This is the part I spent the most time formalizing. Two modes:
- **Sovereign Mode** — client does all the decryption, worker is a dumb relay. Zero-knowledge against both the storage provider and the coordination infra.
- **Edge Mode** — worker decrypts via an ephemeral X25519 exchange and streams plaintext to you. Faster perceived throughput for large files, but the worker has transient access to plaintext during the session.

I benchmarked both — Sovereign Mode actually has *lower* time-to-first-byte (~900ms vs ~2.4s for Edge), which surprised me. Edge Mode's win shows up in sustained throughput for big files instead.

**A serverless coordination layer.** Runs entirely on Cloudflare Workers, within the 128MB memory cap, using the Streams API so files of arbitrary size can pass through without ever being materialized in memory.

> Decryption mode is a per-request client parameter, defaulting to Sovereign Mode. Pick Edge only when you trust the coordination infra and want faster sustained throughput on large files.
{: .prompt-tip }

---

## Numbers that mattered

- zstd gets up to **69.4x** compression on source code (88%+ on JSON/text)
- AES-256-GCM sustains **1.3–1.6 GB/s** — never the bottleneck
- Upload pipeline is **network-bound**, >99.7% of total time is just transfer
- At RF=2 replication, **100% retrieval success** under single-provider failure, with only **83ms** failover overhead

The whole thing aggregates to 27GB of free capacity across three providers in the current PoC.

---

## Where it's headed

Paper is submitted to *Future Internet* (MDPI), preprint is live on Zenodo with a DOI. Next on the list: erasure coding instead of straight replication (current RF=3 wastes 66% of capacity — Reed-Solomon could cut that significantly), and a real streaming throughput benchmark for Edge Mode at scale.

Code's open source if you want to dig into the provider abstraction or the chunking math. Happy to talk through any of the architecture decisions — drop a comment or hit me up on [X](https://x.com/pyvrma).
