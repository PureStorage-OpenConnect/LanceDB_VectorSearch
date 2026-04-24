# LanceDB on S3 — Benchmarks

Two standalone Rust binaries for exercising LanceDB against an S3-compatible
object store (tested on FlashBlade, but works with AWS S3, MinIO, etc.):

1. **`lancedb-qps-10m`** — end-to-end pipeline benchmark: ingest → compact →
   IVF_PQ index → concurrent QPS sweep.
2. **`lancedb-atomic-demo`** — side-by-side demo showing why LanceDB's
   conditional-PUT commits prevent the silent data loss you get with a naive
   last-writer-wins S3 PUT.

---

## Prerequisites

### 1. Install Rust / Cargo

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
rustc --version    # 1.75+ recommended
```

### 2. S3 credentials

Both binaries read credentials from environment variables. Nothing is baked in.
You need an endpoint, an access key, a secret key, and a bucket you can write
to.

### 3. Build

```bash
cd benchmark_framework/streaming-bench
cargo build --release --bin lancedb-qps-10m --bin lancedb-atomic-demo
```

Binaries land in `./target/release/`.

---

## Scenario 1 — `lancedb-qps-10m` (end-to-end pipeline benchmark)

Runs the full lifecycle of a vector table against S3 and prints a summary.

### Phases

```
Phase 1: INGEST     Read .arrow IPC or .parquet files → push to S3 (parallel writers)
Phase 2: COMPACT    Merge fragments to target_rows_per_fragment + prune old versions
Phase 3: INDEX      Build IVF_PQ index (256 partitions, 64 sub-vectors, cosine)
Phase 4: WARMUP     Fire 50 concurrent queries to warm the connection pool + S3 caches
Phase 5: QPS SWEEP  Ramp concurrency through multiple tiers, 30s each, record p50/p99/p99.9
```

Each tier uses shared atomic counters and a lock-free latency window. Only
queries that return actual result batches count as OK; failures are reported
separately with an error rate.

### Environment variables

Required:

| Variable          | Description                                                |
|-------------------|------------------------------------------------------------|
| `S3_ENDPOINT`     | S3 endpoint URL (e.g. `http://10.0.0.1` or AWS regional)   |
| `S3_ACCESS_KEY`   | Access key                                                 |
| `S3_SECRET_KEY`   | Secret key                                                 |
| `DATA_DIR`        | Directory of `.arrow` or `.parquet` files to ingest        |
| `QUERY_FILE`      | `.arrow` or `.parquet` file with query vectors             |

Optional:

| Variable                   | Default             | Description                                  |
|----------------------------|---------------------|----------------------------------------------|
| `S3_BUCKET`                | `lance-demo`        | Bucket name                                  |
| `DB_PATH`                  | `lance-data-10m`    | Lance DB path inside bucket                  |
| `TABLE_NAME`               | `wiki_10m`          | Table name                                   |
| `SELECT_COL`               | `id`                | Metadata column returned with each hit       |
| `PHASES`                   | `ingest,index,search` | Any comma-separated subset                 |
| `K`                        | `10`                | Top-k nearest neighbors                      |
| `NPROBES`                  | `5`                 | IVF partitions to probe                     |
| `REFINE_FACTOR`            | `1`                 | Re-ranking factor                            |
| `TARGET_ROWS_PER_FRAGMENT` | `1000000`           | Rows per fragment after compaction           |

### How to run

```bash
export S3_ENDPOINT="http://<your-endpoint>"
export S3_ACCESS_KEY="<access-key>"
export S3_SECRET_KEY="<secret-key>"
export S3_BUCKET="wiki-bench"
export DB_PATH="lance-data-10m"
export TABLE_NAME="wiki_10m"
export DATA_DIR="/path/to/wiki-embeddings"
export QUERY_FILE="/path/to/wiki-embeddings/000.parquet"
export SELECT_COL="id"
export PHASES="ingest,index,search"

./target/release/lancedb-qps-10m
```

To re-run just the search phase against an existing table:

```bash
PHASES=search ./target/release/lancedb-qps-10m
```

### Sample output (10M × 1024-dim Wiki embeddings)

```
--- Phase: Ingest ---
Ingestion complete: 10000000 rows, ~100 fragments in 21.70s

--- Phase: Compaction + Cleanup ---
  target_rows_per_fragment: 1000000
Fragments before: ~100
Fragments after:  10
Rows after:       10000000

--- Phase: IVF_PQ Index ---
Index built in 86.55s

--- Phase: QPS Benchmark ---
  k=10, nprobes=5, refine_factor=1, select=[id]

Threads    QPS          p50(ms)      p99(ms)      p99.9(ms)    OK         Errors
--------------------------------------------------------------------------------
10          697.0       14.12        21.05        35.55        20910      0
25         1422.8       17.12        26.44        31.35        42683      0
50         2208.7       21.92        37.08        43.32        66262      0
100        2616.0       36.85        67.05        78.09        78480      0
150        2738.0       54.55       101.02       117.18        79139      0

================================================================================
                        BENCHMARK SUMMARY
================================================================================
  Total Rows:           10000000
  Ingest Throughput:    460904 rows/s
  Index Build Time:     86.55s
  Best QPS:             2638.0 (at 150 threads)
  Best p50:             14.12 ms (at 10 threads)
  Best p99:             21.05 ms (at 10 threads)
  Error rate:           0.0000%
================================================================================
```

At 150 concurrent clients, QPS plateaus near the 10 GbE NIC ceiling. Reducing
the result payload (`SELECT_COL=id` vs a larger text column) is the single
biggest lever for higher QPS on a network-constrained host.

---

## Scenario 2 — `lancedb-atomic-demo` (conditional-PUT safety demo)

Demonstrates why LanceDB's `conditional_put = etag` storage option is essential
when multiple writers can commit concurrently. It runs the same N-writer race
twice against the same S3 key:

- **Test 1** — plain `PUT` (no conditional header): every writer gets HTTP 200,
  but only one body survives. The rest are silently lost.
- **Test 2** — `PUT` with `If-None-Match: *`: exactly one writer gets HTTP 200,
  the rest get HTTP 412 Precondition Failed. No silent loss.

This is the primitive LanceDB uses to make manifest commits on S3 safe under
concurrent writers and compaction.

### Environment variables

| Variable                  | Required | Description                     |
|---------------------------|----------|---------------------------------|
| `S3_ENDPOINT`             | yes      | S3 endpoint URL                 |
| `S3_ACCESS_KEY_ID`        | yes      | Access key (or `AWS_ACCESS_KEY_ID`)     |
| `S3_SECRET_ACCESS_KEY`    | yes      | Secret key (or `AWS_SECRET_ACCESS_KEY`) |
| `S3_BUCKET`               | yes      | Bucket name                     |
| `NUM_WRITERS`             | no       | Concurrent writers (default 50) |

Note: `atomic_demo` uses the AWS-style env var names (`S3_ACCESS_KEY_ID`),
whereas `qps_10m` uses the shorter `S3_ACCESS_KEY`. Export both if running
back-to-back.

### How to run

```bash
export S3_ENDPOINT="http://<your-endpoint>"
export S3_ACCESS_KEY_ID="<access-key>"
export S3_SECRET_ACCESS_KEY="<secret-key>"
export S3_BUCKET="wiki-bench"
export NUM_WRITERS=50

./target/release/lancedb-atomic-demo
```

The bucket must exist and be writable. The demo writes and deletes a single
key under `atomic_test/manifest.json`.

### Sample output

```
  S3 Atomic Commit Demo  ·  50 concurrent writers  ·  same key

──────  TEST 1  ·  STANDARD S3 PUT  (no conditional headers)  ──────
  [W 00]  →  HTTP 200 OK
  [W 01]  →  HTTP 200 OK
  ... (all 50 return 200) ...

  HTTP 200 OK returned : 50 / 50   ← every writer "succeeded"
  Actually on S3       :  1        ← only one writer's data survived
  SILENTLY LOST        : 49        ← zero errors, zero warnings
  ✗  SILENT DATA CORRUPTION

──────  TEST 2  ·  CONDITIONAL PUT  (If-None-Match: *)  ──────
  [W 17]  →  HTTP 200 OK   ← WINNER
  [W 00]  →  HTTP 412 rejected
  ... (49 rejections) ...

  HTTP 200 OK returned :  1 / 50   ← exactly one winner
  HTTP 412 rejected    : 49 / 50   ← immediate, explicit failure
  Silently lost        :  0
  ✓  SAFE ATOMIC COMMIT

╔══════════════════════════════════════════════════════════════════╗
║         50 CONCURRENT WRITERS  ·  RACING TO COMMIT               ║
╠════════════════════════════════╦═════════════════════════════════╣
║  STANDARD S3 PUT               ║  CONDITIONAL PUT (ETag)         ║
║  Accepted       : 50 / 50      ║  Accepted       :  1 / 50       ║
║  Survived       :  1 / 50      ║  Rejected (412) : 49 / 50       ║
║  SILENTLY LOST  : 49           ║  Silently lost  :  0            ║
║  ✗  SILENT CORRUPTION          ║  ✓  SAFE ATOMIC COMMIT          ║
╚════════════════════════════════╩═════════════════════════════════╝
```

---

## Notes

- The S3 backend must support `If-None-Match: *` on PUT (AWS S3 since Nov 2024,
  FlashBlade, recent MinIO). Older/compatible stores that ignore the header
  will fall back to last-writer-wins — the demo is a quick way to verify.
- `lancedb-qps-10m` defaults `TARGET_ROWS_PER_FRAGMENT=1000000`. For a 10M-row
  dataset that collapses ~100 source fragments down to ~10, which materially
  reduces per-query fan-out during search.
