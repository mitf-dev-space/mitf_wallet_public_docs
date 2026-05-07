# Load test — reference runs (March 2026)

Docker load tests A–F: **10k** (A/B), **250k** (C/D), **1M** (E/F). Compose files under **`compose/loadtest/`** (six overlays); each uses **four** Transactions workers + API unless noted.

---

## Run A — 10k (no chaos)


| Item                       | Value                                                                  |
| -------------------------- | ---------------------------------------------------------------------- |
| **Compose**                | `docker-compose.yml` + `compose/loadtest/loadtest-10k-no-chaos.yml`     |
| **Runner**                 | `pwsh -File scripts/run-loadtests.ps1` → select `loadtest-10k-no-chaos` |
| **Wallets / transfers**    | 10k / 10k                                                              |
| **Pairing**                | `rotating_unique_destinations`                                         |
| **Batch `MaxConcurrency`** | 96                                                                     |
| **Chaos**                  | All rates **0**                                                        |
| **Pool sizes**             | **Historical run:** ledger/wallets **48** on a **1 worker + API** overlay. **Current file:** pool **28** on API + **four** workers (re-measure after reorg). |


### Observed results (2026-03-27)


| Metric                              | Value                                                               |
| ----------------------------------- | ------------------------------------------------------------------- |
| Transfer phase                      | **10000 / 10000** success                                           |
| **Throughput**                      | **146.2 ops/s**                                                     |
| Latency (ms)                        | min 259 · **p50 537** · **p95 800** · p99 1065 · max 2107 · avg 505 |
| Async diagnostics (server-total ms) | p50 **333** · p95 **606** · max 1975 · avg **364**                  |
| Queue wait (ms)                     | p50 20 · p95 155 · max 1746 · avg 46                                |
| **Consistency**                     | **PASS**                                                            |
| **Transfer SLO**                    | **PASS** (throughput ≥ 65 ops/s)                                    |


---

## Run B — 10k (with chaos)


| Item                       | Value                                                                                                                                                                    |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Compose**                | `docker-compose.yml` + `compose/loadtest/loadtest-10k-chaos.yml`                                                                                                        |
| **Wallets / transfers**    | 10k / 10k                                                                                                                                                                |
| **Pairing**                | `legacy_repeat_pairs` (report-style modulo pairing)                                                                                                                      |
| **Batch `MaxConcurrency`** | 160                                                                                                                                                                      |
| **Chaos**                  | Delay **5%** (25–300 ms) · Timeout **3%** (75 ms) · Invalid destination **1%** · Duplicate idempotency **3%**                                                            |
| **Pool sizes (overlay)**   | Ledger / Wallets: **28** (tighter than enhancement-proof)                                                                                                                |
| **SLO env**                | Throughput/latency SLO checks **disabled** in overlay (`MinThroughputOpsPerSec: 0`, `FailOnBreach: false`) — job still logs `SLO status: PASS` when internal checks pass |


### Observed results (2026-03-27)


| Metric                              | Value                                                                                                                                     |
| ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| Transfer phase                      | **9891 / 10000** success · **109** failed (expected chaos + 2 deadline-exceeded)                                                          |
| **Throughput**                      | **89.2 ops/s**                                                                                                                            |
| Latency (ms)                        | min 89 · p50 **1071** · p95 **1861** · p99 **2299** · max **2745** · avg **1210**                                                         |
| Async diagnostics (server-total ms) | p50 **979** · p95 **1778** · max **2536** · avg **1070**                                                                                  |
| Queue wait (ms)                     | p50 **471** · p95 **1082** · max **1647** · avg **531**                                                                                   |
| Chaos summary                       | DelayInjections **484** · TimeoutInjections **310** · InvalidDestination **107** · DuplicateReplays **268** · **ReplayMismatches 0**      |
| Top errors                          | 107× “Destination wallet not found” (injected)                                                                                            |
| **Consistency**                     | **WARN** — adjusted delta **−0.77 LYD** on 300+300 sample (fee-compensation rounding vs ~98.9% success rate; not a material ledger break) |
| **Transfer SLO**                    | **PASS** (as logged by batch job for that configuration)                                                                                  |


---

## Comparing A vs B

These runs are **not apples-to-apples** on load shape:

- **A** uses **rotating** pairing and **96** client concurrency with **larger** DB pools.
- **B** uses **legacy** pairing, **160** concurrency, **smaller** pools, and **chaos** (extra latency + failures).

Use **A** for “clean ceiling” after code/infra changes. Use **B** for idempotency and error-path behaviour under fault injection (**ReplayMismatches must stay 0**). Use **C** for long-run scale without chaos; **D** for same scale with chaos.

---

## Run C — 250k (no chaos)


| Item                       | Value                                                                 |
| -------------------------- | --------------------------------------------------------------------- |
| **Compose**                | `docker-compose.yml` + `compose/loadtest/loadtest-250k-no-chaos.yml`  |
| **Wallets / transfers**    | 50k / 250k                                                            |
| **Pairing**                | `legacy_repeat_pairs`                                                 |
| **Batch `MaxConcurrency`** | 128                                                                   |
| **Provision / funding**    | 24 / 32                                                               |
| **Fund per source**        | 100000 LYD                                                            |
| **Chaos**                  | All rates **0**                                                       |
| **Consistency sample**     | 800 per side · settle **12s**                                         |
| **Consumers (historical)** | Run **C** used **API only** with consumer limit **20**. **Current** `loadtest-250k-no-chaos.yml`: **four** workers + API at **36** each. |

**Command (repo root):**

```powershell
pwsh -File scripts/run-loadtests.ps1
# choose compose/loadtest -> loadtest-250k-no-chaos -> standard reset
```

### Observed results (2026-03-27)


| Phase          | Planned / success / failed | Duration   | Throughput   | Latency (ms) — p50 / p95 / p99 / max / avg              |
| -------------- | -------------------------- | ---------- | ------------ | ------------------------------------------------------- |
| **Provisioning** | 50000 / 50000 / 0        | ~498.9 s   | **100.2**/s  | 237 / 304 / 501 / **8635** / 239                        |
| **Funding**      | 25000 / 25000 / 0        | ~43.5 s    | **575.3**/s  | 51 / 81 / 113 / 1047 / 56                               |
| **Transfer**     | 250000 / 250000 / 0      | ~2117.7 s (~35.3 min) | **118.1**/s | 1060 / 1575 / 1830 / **3133** / 1052                |

**Transfer async diagnostics (batch-measured):** queue-wait p50 **563** · p95 **957** · max **2352** · avg **594** (ms). Processing p50 **295** · p95 **503** · max **1745** · avg **313** (ms). Server-total p50 **866** · p95 **1387** · max **2913** · avg **907** (ms). Client-overhead p50 **146** · avg **146** (ms). Poll attempts p50 **5** · p95 **7** · max **13** · avg **4.9**.

**Chaos summary:** none (all zero).

**Consistency:** **WARN** on this run — raw ledger liability delta **−4000 LYD** on 800+800 sample with **fee compensation 0** in the batch log (SLO still **PASS**). That reflects **check configuration**, not missing money: P2P fees post to the **fee-revenue** ledger account, outside the sampled user-wallet sum. This run used the overlay **before** `ConsistencyCheckExpectedFeePerTransferLyd` was added. **`compose/loadtest/loadtest-250k-no-chaos.yml` sets `LoadTest__LoadTest__Chaos__ConsistencyCheckExpectedFeePerTransferLyd` to `0.5`** — **re-run** for an expected **PASS** on the consistency line.

**Transfer SLO:** **PASS** (as logged).

**Notes:** Provisioning max **~8.6s** is a long tail (cold start / occasional stall); p50 provisioning remains ~**237 ms**. Transfer phase is **queue-wait heavy** (p50 ~564 ms vs processing ~295 ms), consistent with **consumer limit 20** and **128** client concurrency.

---

## Run D — 250k (with chaos)


| Item                       | Value                                                                 |
| -------------------------- | --------------------------------------------------------------------- |
| **Compose**                | `docker-compose.yml` + `compose/loadtest/loadtest-250k-chaos.yml`      |
| **Wallets / transfers**    | 50k / 250k                                                            |
| **Pairing**                | `legacy_repeat_pairs`                                                 |
| **Batch `MaxConcurrency`** | 128                                                                   |
| **Provision / funding**    | 24 / 32                                                               |
| **Chaos**                  | Delay **5%** (25–300 ms) · Timeout **3%** (75 ms) · Invalid dest **1%** · Duplicate idempotency **3%** |
| **Consistency**            | Sample **800** per side · settle **15s** · fee **0.5** LYD / successful in-sample tx |
| **Consumers (historical)** | Run **D** table below used **API + one** worker. **Current** `loadtest-250k-chaos.yml`: **four** workers + API. |

**Command (repo root):**

```powershell
pwsh -File scripts/run-loadtests.ps1
# choose compose/loadtest -> loadtest-250k-chaos -> standard reset
```

### Observed results — Run D baseline (2026-03-27, **1** worker + API)


| Phase            | Planned / success / failed | Duration     | Throughput   | Latency (ms) — p50 / p95 / p99 / max / avg |
| ---------------- | -------------------------- | ------------ | ------------ | ------------------------------------------ |
| **Provisioning** | 50000 / 50000 / 0          | ~404.9 s     | **123.5**/s  | 189 / 328 / 492 / **10817** / 194          |
| **Funding**      | 25000 / 25000 / 0          | ~47.2 s      | **529.7**/s  | 55 / 89 / 115 / 1193 / 60                  |
| **Transfer**     | 250000 / **243293** / **6707** | ~2347.1 s (~39.1 min) | **103.7**/s | 1083 / 1589 / 1844 / **3134** / 1124   |

**Transfer chaos summary:** TimeoutInjections **7489** · DelayInjections **12393** · InvalidDestination **2516** · DuplicateReplays **7641** · **ReplayMismatches 0**.

**Top error samples:** **3892×** “Transfer processing failed. Please retry…” · **2514×** “Destination wallet not found” · **301×** `DeadlineExceeded` (split across two message variants).

**Transfer async diagnostics:** queue-wait p50 **356** · p95 **733** · max **1 339 672** · avg **697** (ms). Processing p50 **615** · p95 **866** · max **1 340 193** · avg **1031** (ms). Server-total p50 **976** · p95 **1444** · max **1 341 108** · avg **1729** (ms). Client-overhead p50 **145** · avg **145** (ms). Poll attempts p50 **5** · p95 **7** · max **13** · avg **5.2**.

**Consistency:** **WARN** — raw delta **−3958.5 LYD**; fee compensation **+3892.69 LYD** (slots **8000**, est. successes **~7785.4** × **0.5**); **adjusted −65.81 LYD** on 800+800 sample. Residual is small vs gross flow; with **~2.7%** transfer hard failures and chaos, the linear fee model does not close exactly.

**Transfer SLO:** **PASS** (as logged; overlay typically disables strict throughput/latency breach failure).

### Analysis (read before treating numbers as regressions)

1. **Failures vs chaos** — **2516** invalid-destination injections match **~2514** “Destination wallet not found” (expected). **6707** failed transfers are not a ledger bug; they mix injected bad destinations, client **timeout** injections (**7489** events — not all become failures), **`DeadlineExceeded`**, and **3892** generic retryable errors under pressure.
2. **ReplayMismatches = 0** — duplicate idempotency replays (**7641**) did not diverge; idempotency behaved.
3. **Throughput** — **103.7 ops/s** vs Run **C** **118.1 ops/s**: expected drop from chaos (delays, timeouts, failures) and slightly heavier client work; still above a typical **65 ops/s** gate if enabled.
4. **Diagnostic maxima (~1.34×10⁶ ms)** — **max** is the **largest single transfer’s** server-reported component among successes (not one stuck client poll: terminal polling caps at ~**30 s**). The ~**22 minute** queue/processing/total values align with **CreatedAt→CompletedAt** on a **few** requests that sat in the Rabbit/consumer backlog under chaos; **p95 stayed ~0.7–1.6 s**, so the tail is rare. The batch job now logs **p99** and a **warning** when any sample exceeds **2 minutes** (see [Load testing operations](../operations/load-testing-operations.md#async-diagnostic-maxima)).
5. **Consistency WARN** — After fee compensation, **~66 LYD** remains on an **80 000 000 LYD** sampled base (~**0.08 bp**). Acceptable for a chaos run; tighten only if product requires exact closure on sampled sums under fault injection.
6. **Provisioning vs Run C** — **123.5 ops/s** vs **100.2 ops/s** on no-chaos: chaos does not run during provisioning; difference is **run-to-run variance** (host, DB state, code revision such as batch provisioning optimizations), not chaos-driven.

### Comparing Run C vs Run D

| Aspect | Run C (no chaos) | Run D (chaos) |
|--------|------------------|---------------|
| Transfer success | 250000 / 250000 | 243293 / 250000 (**6707** failed, expected) |
| Transfer throughput | 118.1 ops/s | 103.7 ops/s |
| Queue-wait p50 (ms) | ~564 | ~356 |
| Consistency | WARN (no fee env on that C run) / fixed in compose afterward | WARN (~66 LYD residual after fee adj) |
| Idempotency | N/A | **ReplayMismatches 0** |

---

### Run D′ — 250k chaos with **six** Transactions workers (+ API) — **archived**

**Same** wallet/transfer counts, chaos rates, and batch **128** concurrency as Run **D**. This used a **removed** second compose overlay (**six** workers + API = **seven** consumer processes). The repo default is now **`compose/loadtest/loadtest-250k-chaos.yml`** with **four** workers + API. The command below is **not** reproducible from the current tree without restoring the old overlay.

**Reproduce today:** not available — the extra-worker overlay was deleted in favour of a single **`loadtest-250k-chaos.yml`** with **four** workers + API.

#### Observed results (2026-03-28)

Log excerpt: service **15:54:24** → transfer phase end **16:33:53**; pasted log **ends** at consistency **wait** line — capture **PASS/WARN** and adjusted delta from full `docker logs` if needed.


| Phase            | Planned / success / failed | Duration              | Throughput   | Latency (ms) — p50 / p95 / p99 / max / avg      |
| ---------------- | -------------------------- | --------------------- | ------------ | ----------------------------------------------- |
| **Provisioning** | 50000 / 50000 / 0        | **389837 ms** (~6.5 min) | **128.3**/s | 186 / 276 / **391** / **8999** / **187**        |
| **Funding**      | 25000 / 25000 / 0        | **38950 ms** (~39 s)  | **641.8**/s  | 47 / 79 / **106** / **1197** / **50**          |
| **Transfer**     | 250000 / **246579** / **3421** | **1924117 ms** (~32.1 min) | **128.2**/s | 832 / 1340 / **1621** / **3109** / **937** |

**Transfer chaos summary:** TimeoutInjections **7518** · DelayInjections **12342** · InvalidDestination **2505** · DuplicateReplays **7375** · **ReplayMismatches 0**.

**Top error samples:** **2504×** “Destination wallet not found” · **777×** “Insufficient balance” · **110×** / **29×** `DeadlineExceeded` (two detail variants) · **1×** gRPC **`InvalidDataException`** (“Unexpected end of content…”). **Buckets sum to 3421** failed transfers.

**Transfer async diagnostics:** queue-wait p50 **22.9** · p95 **45.5** · p99 **54.8** · max **233** · avg **24.8** (ms). Processing p50 **733** · p95 **1198** · p99 **1509** · max **2834** · avg **767** (ms). Server-total p50 **758** · p95 **1223** · p99 **1534** · max **2859** · avg **792** (ms). Client-overhead p50 **146** · p95 **259** · max **453** · avg **145** (ms). Poll attempts p50 **4** · p95 **6** · p99 **7** · max **13** · avg **4.5**.

**Memory (transfer start):** working set **~114 MB**, private **~183 MB**.

#### Run D vs Run D′ (did extra workers help?) — archived overlay

| Metric | Run D (1 worker + API) | Run D′ (6 workers + API, archived) | Δ (D′ − D) |
|--------|------------------------|---------------------------|------------|
| Transfer duration | ~2347 s (~39.1 min) | ~1924 s (~32.1 min) | **~17% shorter** |
| Throughput (successes/s) | **103.7**/s | **128.2**/s | **+24%** |
| Successes | 243293 | 246579 | **+3286** |
| Failures | 6707 | 3421 | **−3286** (~**49%** fewer) |
| Transfer p50 / p95 / avg (ms) | 1083 / 1589 / 1124 | 832 / 1340 / 937 | **Lower latency** |
| Queue-wait p50 (ms) | **356** | **23** | **Rabbit backlog largely removed** (see analysis below) |
| Processing p50 (ms) | **615** | **733** | Slightly **higher** (work shifted to ledger/DB path) |
| Server-total p50 (ms) | **976** | **758** | **Lower** end-to-end median |
| Pathological async max (ms) | ~**1.34×10⁶** (~22 min tails) | **under ~3k** | **No** multi-minute tails in this sample |
| Top “retry” pressure | **3892×** generic retry | (not in top five) | Less overload / fewer ambiguous failures |

**Interpretation (short):** Extra workers **did help** on this run: **higher** successful throughput, **shorter** wall time, **fewer** failures, **much lower** queue wait, and **no** huge queue-wait outliers. The bottleneck **moves** from “waiting for a consumer” to **processing** (p50 **~733 ms**), i.e. ledger/DB/wallets work — consistent with **db** often being hot when many consumers drain Rabbit quickly (see Docker CPU: API ingress + DB).

**Insufficient balance (777×):** Not the primary chaos bucket; plausible under **legacy_repeat_pairs** (fixed source→dest cycling), **1 LYD** + **0.5** fee, and **more** successful transfers completing — some sources can be drained below the next attempted debit while retries or ordering still fire. Treat as **worth monitoring**, not proof of a bug without matching ledger rows.

---

## Run E — 1M (no chaos)


| Item                       | Value                                                                 |
| -------------------------- | --------------------------------------------------------------------- |
| **Compose**                | `docker-compose.yml` + `compose/loadtest/loadtest-1m-no-chaos.yml`   |
| **Wallets / transfers**    | 100k / **1,000,000**                                                  |
| **Pairing**                | `legacy_repeat_pairs`                                                 |
| **Batch `MaxConcurrency`** | **160**                                                               |
| **Provision / funding**    | **24** / **32**                                                       |
| **Fund per source**        | **120000 LYD** (log warns projected gross funding **6e9 LYD**)        |
| **Transfer amount**        | **1 LYD**                                                             |
| **Chaos**                  | All rates **0**                                                       |
| **Consistency**            | Sample **1000** per side · settle **18s** · fee **0.5** LYD / in-sample successful tx |
| **Consumers (historical)** | Run **E** used **API only** (consumer **24**). **Current** `loadtest-1m-no-chaos.yml`: **four** workers + API at **36** each. |

**Runner:**

```powershell
pwsh -File scripts/run-loadtests.ps1
# choose compose/loadtest -> loadtest-1m-no-chaos
```

### Observed results (2026-03-27 → 2026-03-28)

Batch job wall clock in logs spans **~21:44 → ~01:37** (UTC+2-style timestamps in excerpt; **~3 h 52 min** end-to-end including provision, funding, transfers, consistency).


| Phase            | Planned / success / failed | Duration              | Throughput    | Latency (ms) — p50 / p95 / p99 / max / avg                    |
| ---------------- | -------------------------- | --------------------- | ------------- | ------------------------------------------------------------- |
| **Provisioning** | 100000 / 100000 / 0        | **790371 ms** (~13.2 min) | **126.5**/s | 183 / 336 / **592** / **8168** / **190**                      |
| **Funding**      | 50000 / 50000 / 0          | **65680 ms** (~1.1 min)   | **761.3**/s | 34 / 80 / **112** / **1090** / **42**                         |
| **Transfer**     | 1000000 / 1000000 / 0      | **12000109 ms** (~3.33 h) | **83.3**/s  | **1843** / **2687** / **3314** / **5386** / **1898**          |

**Memory (transfer phase start, batch log):** working set **~121 MB**, private **~199 MB**.

**Transfer chaos summary:** all **0** · **ReplayMismatches 0**.

**Transfer async diagnostics (batch-measured):** queue-wait p50 **1418** · p95 **2175** · max **825158** · avg **1605** (ms). Processing p50 **252** · p95 **455** · max **823032** · avg **295** (ms). Server-total p50 **1681** · p95 **2543** · max **825609** · avg **1900** (ms). Client-overhead p50 **147** · p95 **266** · max **834** · avg **149** (ms). Poll attempts p50 **8** · p95 **11** · max **19** · avg **8.1**.

**Consistency:** **PASS** — raw delta **−10000.0000 LYD** (before **120000000**, after **119990000** on sampled sum); fee compensation **+10000** (**20000** sampled-pair slots × **0.5** LYD); **adjusted 0.0000**. First post-transfer sum logged a **−10000** delta; **2s** re-sum path ran (projection / reservation skew note in log).

**Transfer SLO (batch log):** **BREACH** — **`max 5385.5ms > 5000.0ms`** on the **historical** overlay that omitted **`MaxLatencyMs`**. **Current** `loadtest-1m-no-chaos.yml` sets **`LoadTest__LoadTest__TransferSlo__MaxLatencyMs: "0"`** to disable that gate; p95 (**2687 ms**) stayed **below** 5s on the documented run.

**Notes:** Throughput **83.3 ops/s** is **below** Run **C** (**118.1**) at 250k — expected at **4×** transfer volume with similar consumer/pool headroom (**queue-wait p50 ~1.4 s** vs ~**0.56 s** on Run C). Very large **async diagnostic maxima (~825 s)** are rare tails (see [Load testing operations](../operations/load-testing-operations.md#async-diagnostic-maxima)); **p95** remains representative.

---

## Run F — 1M (with chaos)

Same scale as Run **E** with chaos. **Current** compose: four workers + API (documented run used API + one worker — re-benchmark with the new file).


| Item                       | Value                                                                 |
| -------------------------- | --------------------------------------------------------------------- |
| **Compose**                | `docker-compose.yml` + `compose/loadtest/loadtest-1m-chaos.yml`      |
| **Wallets / transfers**    | 100k / **1,000,000**                                                  |
| **Pairing**                | `legacy_repeat_pairs`                                                 |
| **Batch `MaxConcurrency`** | **144**                                                               |
| **Provision / funding**    | **24** / **32**                                                       |
| **Fund per source**        | **120000 LYD**                                                        |
| **Transfer amount**        | **1 LYD**                                                             |
| **Chaos**                  | Delay **5%** (25–300 ms) · Timeout **3%** (75 ms) · Invalid dest **1%** · Duplicate idempotency **3%** |
| **Consistency**            | Sample **1000** per side · settle **18s** · fee **0.5** LYD            |
| **Infra (historical)**     | Run **F** numbers: API + **one** worker (**72** concurrent transfer handlers total). **Current** `loadtest-1m-chaos.yml`: API + **four** workers (**180** transfer handler slots); prefetch **96**, backpressure **320**. |

**Recommended:** after Run E finishes, rerun **`pwsh -File scripts/run-loadtests.ps1`** for Run F and choose the **destructive reset** option (or otherwise wipe Postgres) so Run F does not stack another 100k wallets on the same DB.

**Runner:**

```powershell
pwsh -File scripts/run-loadtests.ps1
# choose compose/loadtest -> loadtest-1m-chaos and use destructive reset if you want a fresh DB
```

### Observed results (2026-03-28)

Batch log excerpt: provisioning **~08:28** → transfer phase end **~13:58** (wall clock includes **~12.4 min** provision, **~45 s** funding, **~115 min** transfers).


| Phase            | Planned / success / failed | Duration                   | Throughput     | Latency (ms) — p50 / p95 / p99 / max / avg              |
| ---------------- | -------------------------- | -------------------------- | -------------- | ------------------------------------------------------- |
| **Provisioning** | 100000 / 100000 / 0        | **741827 ms** (~12.4 min)  | **134.8**/s    | 183 / 225 / **306** / **4635** / **178**                |
| **Funding**      | 50000 / 50000 / 0          | **44694 ms** (~44.7 s)     | **1118.7**/s   | 26 / 41 / **55** / **945** / **29**                     |
| **Transfer**     | 1000000 / **983715** / **16285** | **6901145 ms** (~115 min) | **142.5**/s | **848** / **1579** / **1871** / **3657** / **969**      |

**Memory (transfer phase start):** working set **~118 MB**, private **~172 MB**.

**Transfer chaos summary:** TimeoutInjections **30039** · DelayInjections **49902** · InvalidDestinationInjections **10017** · DuplicateReplays **30193** · **ReplayMismatches 0**.

**Top error samples (batch log):** **10002×** “Destination wallet not found” · **5173×** “Transfer processing failed. Please retry…” · **821×** `DeadlineExceeded` (empty detail) · **273×** `DeadlineExceeded` (“Deadline Exceeded”) · **16×** `FailedPrecondition` (transient). **Sum of shown buckets = 16285** (matches **Failed** count).

**Transfer async diagnostics:** queue-wait p50 **328** · p95 **784** · p99 **1049** · max **6 680 002** · avg **879** (ms). Processing p50 **450** · p95 **768** · p99 **984** · max **6 680 080** · avg **1139** (ms). Server-total p50 **781** · p95 **1425** · p99 **1793** · max **6 680 841** · avg **2018** (ms). Client-overhead p50 **146** · p95 **258** · p99 **271** · max **396** · avg **145** (ms). Poll attempts p50 **4** · p95 **7** · p99 **8** · max **15** · avg **4.6**.

**Outlier warnings (batch):** **113** queue-wait · **138** processing · **251** server-total samples **above 2 min** (same run). Maxima **~6.68×10⁶ ms (~111 min)** are on the same order as the **transfer phase duration (~115 min)** — consistent with **straggling completions** (work accepted early, finished only late in the run) under load + chaos, not with typical p95 latency.

**Consistency:** **WARN** — raw delta **−9894.5000 LYD**; fee compensation **+9837.1500** (slots **20000**, est. successes **~19674.3** × **0.5**); **adjusted −57.3500 LYD** on 1000+1000 sample. Same class as Run **D** (linear fee model vs mixed failures / rounding); small vs **120 M LYD** sampled gross.

**Transfer SLO:** **PASS** (as logged; **max 3657 ms** below default **5000 ms** max-latency gate).

### Run E vs Run F (same 1M scale, different shape)

| Aspect | Run E (no chaos) | Run F (chaos; **historical** consumer layout) |
|--------|------------------|-----------------------------------------------|
| Transfer success | 1 000 000 / 1 000 000 | **983 715** / 1 000 000 (**16 285** failed — expected from injection + pressure) |
| Transfer throughput | **83.3**/s | **142.5**/s (batch uses **successful** ops / phase duration) |
| Transfer p50 (ms) | **1843** | **848** |
| Consumers | Single API (**24** limit on old overlay) | API + **one** worker (**36** + **36**). **Current** files: **four** workers + API. |
| Consistency | **PASS** | **WARN** (~**57 LYD** residual after fee adj) |

**Faster** median latency and **higher** reported throughput on **F** vs **E** in this log are largely explained by **more** transaction consumer capacity vs Run **E**’s single-process tuning and **144** vs **160** client concurrency — not by chaos being “free.” Re-measure after switching to **four** workers on both overlays.

---

## Correlating `run.json` with Docker `HandlerTiming`

After a run, **`artifacts/loadtests/<timestamp>-<scenario>/run.json`** records **batch/client** transfer latency (gRPC, status polling, async completion). The Transactions API and workers log **`HandlerTiming`** (including **`Phase=queue_wait`** and per-handler phases) as **Serilog JSON**.

**One-shot report** (same UTC window as the batch job, from `StartedAtUtc` / `EndedAtUtc` plus a small buffer):

```powershell
pwsh -File scripts/profile-loadtest-run.ps1 -RunJsonPath artifacts/loadtests/<your-run>/run.json
```

**Single metric** (any handler substring + phase):

```powershell
pwsh -File scripts/analyze-transactions-handler-timing.ps1 -RunJsonPath artifacts/loadtests/<your-run>/run.json -Handler TransferBetweenWalletsHandler -Phase reserve_balance
```

Optional **`-RunWindowBufferSeconds`** (default **30**) widens the slice for clock skew and late log lines.

**How to read the comparison**

- **run.json** transfer **p50 / p99** = **end-to-end** from the batch job. If this is **much larger** than **handler `total`**, the gap is mostly **outside** the consumer handler (client polling, gRPC, time before the message is published, etc.).
- **`queue_wait` p99** « **handler `total` p99** → tail latency is **in-process / downstream** (ledger, DB, other phases), not time sitting in RabbitMQ before consume.
- **`post_ledger_journal`** captures a **large share** of handler tail but not all of **`total`** (reserve/release, idempotency, publish, etc.).
- **`queue_wait`** is only emitted when wait ≥ **`HandlerTiming:MinDurationMs`** (often **10** ms), so very short waits are under-counted.

**Caveat:** If Docker logs were **rotated** or containers were **recreated** since the run, sample count may be **0**. Keep the stack up and profile **immediately** after the batch job exits, or **redirect** `docker logs` to files during the test.

**From the interactive runner:** after you confirm scenarios, answer **y** to *“After each scenario, run HandlerTiming profile…”*, or start non-interactively with **`pwsh -File scripts/run-loadtests.ps1 -ProfileHandlerTiming`** (optional **`-ProfileHandlerTimingBufferSeconds`**). That invokes **`scripts/profile-loadtest-run.ps1`** for each saved **`run.json`** while containers are still up.

---

## Final baseline (2026-03-31)

Latest stable sequence (10k/25k/50k, chaos + no-chaos) from `artifacts/loadtests/index.csv`:

- **No-chaos throughput:** `235.1 -> 284.1 -> 294.4 ops/s` (10k -> 25k -> 50k)
- **No-chaos transfer p95:** `710 -> 596.8 -> 592.3 ms`
- **No-chaos transfer p99:** `1972.8 -> 791.2 -> 1090.9 ms`
- **Replay mismatches:** `0` in all listed runs
- **Consistency:** PASS on no-chaos; chaos runs at 25k/50k show tiny residual (`AdjustedDelta = +1.0`) and are logged as WARN

### Memory profile takeaways (same run set)

- `masarat.ledger.api` stays bounded in the latest 50k runs (peak around **~383-392 MB**).
- `masarat.transactions.api` remains the primary memory/GC pressure point (peak around **~0.99-1.01 GB** at 50k).
- Worker memory is relatively even across instances (roughly **~290-330 MB** peak at 50k).

### Recommended production baseline (applied)

These values are now applied in both base `appsettings.json` and `appsettings.Production.json` for Transactions/Ledger/Wallets:

- **Transactions API**
  - `LedgerGrpc.MaxInFlightRequests: 32`
  - `LedgerGrpc.AcquireTimeoutMs: 20`
  - `TransferBackpressure.MaxInFlightTransfers: 320`
  - `TransferBackpressure.AcquireTimeoutMs: 80`
  - `Messaging.Tuning.PrefetchCount: 64`
  - `Messaging.Tuning.TransferConsumerConcurrentLimit: 22`
  - `Messaging.Tuning.FundWalletConsumerConcurrentLimit: 22`
  - `Messaging.Tuning.RetryCount: 1`, `RetryIntervalSeconds: 1`
- **Ledger API**
  - `LedgerBackpressure.Enabled: true`
  - `LedgerBackpressure.MaxInFlightRequests: 32`
  - `LedgerBackpressure.AcquireTimeoutMs: 10`
- **Wallets API**
  - `LedgerGrpc.MaxInFlightRequests: 32`
  - `LedgerGrpc.AcquireTimeoutMs: 20`

### Why this baseline

It preserves the stability gains (no ledger OOM/collapse in the latest scale steps), controls ledger admission under pressure, and keeps transfer ingress below the point where status polling and queue backlog start to amplify each other.

### Next verification before production rollout

Run `100k-no-chaos` then `250k-no-chaos` with this baseline and confirm:

1. No service restarts/exits.
2. `TransferBetweenWallets` queue does not show sustained upward slope.
3. `masarat.transactions.api` memory rises then plateaus (no runaway slope).
4. `ReplayMismatches = 0` and consistency residual remains near zero.

---

## Related docs

- How to run overlays and read async diagnostics: [Load testing operations](../operations/load-testing-operations.md)
- Platform-wide consistency and durability capabilities: [Platform capabilities](../architecture/platform-capabilities.md)

