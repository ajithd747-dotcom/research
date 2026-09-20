# On-disk formats for a continuously-written crypto tick tape

Scope: trades, 1-minute candles, shallow order-book snapshots, ~30 futures
symbols, 2 venues, indefinite retention, ~2-4 GB/day, 238 GB free.
Python 3.14.4 standard build, no compiler, no sudo. Writer is crash-only
(SIGKILL is the normal shutdown path). Read pattern: symbol X, t0..t1.

Verified 2026-08-21 against PyPI's file listing for each package's current
release, the Parquet and Arrow format specs, sqlite.org, the HDF Group forum,
and real repositories (cryptofeed, nautilus_trader).

> Provenance note: this research was produced by a `research-web` subagent whose
> own operating constraints forbade writing files, so the controller persisted
> the returned content verbatim. Nothing here was re-derived or edited in
> transit. The UNVERIFIED section is the agent's own and is reproduced intact.

## 1. Realistic candidates, evaluated against every constraint

### Parquet (pyarrow)
- **cp314 wheel**: YES. `pyarrow-25.0.1` ships `cp314-cp314-manylinux_2_28_x86_64.whl`
  (and a `cp314t` free-threaded variant), confirmed against PyPI's JSON API file
  listing directly, not memory.
- **Crash-mid-write**: DISQUALIFYING. Parquet's footer (containing all row-group
  offsets) is written once, at `close()`, at the end of the file. A `SIGKILL`
  before that point leaves a file with real data on disk but no magic bytes /
  footer, and it is **completely unreadable** — not "readable up to the
  truncation point," totally lost. This is confirmed by the Parquet project's
  own error-recovery doc ("If an error happens while writing the file metadata,
  all the data written will be unreadable") and by a live DuckLake bug report
  showing the exact symptom: `No magic bytes found at end of file` after an
  unsafe shutdown, requiring manual deletion of the corrupt file and its catalog
  entry. The mitigating feature (`FlushWithFooter`, writing a cumulative footer
  every Nth row group so a reader can recover a partial file) is a **named,
  currently-open Arrow proposal (issue apache/arrow#40630, targeting the Go
  writer)**, not shipped in pyarrow's Python `ParquetWriter` today.
- **Compression**: excellent (snappy/zstd/gzip per column, dictionary + RLE
  encoding); best-in-class ratio of anything evaluated here.
- **Read performance**: excellent for "symbol X, t0..t1" if partitioned so the
  query maps to a directory prefix (see §2) — columnar, predicate pushdown on
  row-group stats.
- **Verdict**: disqualified by the crash-only requirement as a *long-lived,
  actively-appended* file. Usable only as a **finalized, rolled, already-closed**
  format — i.e., convert a completed hour/day of data to Parquet after it stops
  being written, never write ticks into an open Parquet file across a crash
  boundary.

### Plain append-only binary, fixed numpy dtype
- **cp314 wheel**: N/A — no dependency beyond numpy 2.5.2, already pinned and
  confirmed cp314 (21 wheel variants including `cp314` and `cp314t` for the
  current release, per PyPI).
- **Crash-mid-write**: SAFE by construction if records are fixed-width and each
  write is a single fixed-size struct pack + `os.write`/`file.write` of that
  exact byte count (no buffering that spans record boundaries at the Python
  level, though the OS page cache can still lose the last unflushed record — the
  bound is "at most one record," not "the file"). A reader opens the file,
  computes `len(file) // record_size`, and reads whole records; a partial tail
  record (file length not a multiple of record_size) is trivially detected and
  dropped. `numpy.memmap` over such a file is a direct, zero-copy read path —
  already in the pinned stack.
- **Compression**: none intrinsic. Can be added as a second, offline pass
  (compress a *closed* file), not on the live file.
- **Read performance**: excellent for the target query if the file is one
  symbol per file (see §2) — `memmap` + binary search on a sorted timestamp
  column, or just scan since files are already scoped to one symbol.
- **Verdict**: the only format on this list where the disqualifying failure mode
  (whole-file loss) cannot happen, using zero new dependencies.

### SQLite (stdlib `sqlite3`)
- **cp314 wheel**: N/A — stdlib, always present.
- **Crash-mid-write**: SAFE in WAL mode. Per sqlite.org's own WAL documentation:
  "transactions are durable across application crashes (or SIGKILL or similar)"
  in WAL mode (not necessarily across OS crashes/power loss — that's a stronger
  guarantee this project doesn't need). Each WAL frame carries a checksum; a
  torn/partial frame from a kill mid-write is detected and discarded by the
  recovery process the next connection runs on open. Default rollback-journal
  mode is comparatively fragile (a kill between journal deletion and write
  completion can corrupt); **this only holds if WAL mode is explicitly turned
  on**, which is a one-line `PRAGMA journal_mode=WAL`.
- **Compression**: none built in (rows are stored, not columns; no columnar
  compression). Page-level, if enabled via a VFS extension, is not stdlib.
- **Read performance**: fine for symbol+time-range with an index on
  `(symbol, ts)`, but this is row storage — scanning wide time ranges across
  many columns costs more than a columnar format, and single-writer contention
  is a real limit at multi-GB/day multi-symbol ingestion (WAL allows one writer
  at a time; concurrent writers across symbols would need one DB file per
  symbol or serialized writes).
- **Verdict**: safe and zero-dependency, but the worse read/compression profile
  than binary+memmap for this specific "columnar time-range scan" workload
  makes it a better fit for metadata/index bookkeeping than for the tick tape
  itself.

### HDF5 (h5py)
- **cp314 wheel**: YES. `h5py-3.16.0` ships `cp314-cp314-manylinux_2_28_x86_64.whl`
  and a `cp314t` variant, confirmed via PyPI's file listing.
- **Crash-mid-write**: DISQUALIFYING outside SWMR, and even SWMR is a partial
  answer. HDF5's B-tree-based internal metadata is not written transactionally;
  the HDF Group's own forum states a crash before proper file close can corrupt
  the file, and their "crashproofing" work (metadata journaling / WAL) is
  explicitly still under investigation, not shipped. SWMR mode (available since
  1.10) does "leave a file in a valid state even if the writing application
  crashes before closing" — but SWMR requires the *reader* and *writer* to
  cooperate under a specific protocol (single writer, readers polling), adds
  real operational complexity, and the HDF Group itself frames it as reducing
  rather than eliminating corruption risk ("more difficult to corrupt," not
  "cannot").
- **Compression**: good (chunked, gzip/lzf/szip per dataset).
- **Read performance**: good for chunked time-range reads if chunked correctly.
- **Verdict**: disqualified for the live-write file under this project's
  crash-only rule; SWMR narrows but does not close the gap, and the added
  protocol complexity buys little over the binary+memmap approach which has no
  such gap at all.

### Arrow IPC Streaming format (`.arrows`, via pyarrow)
- **cp314 wheel**: YES — same pyarrow wheel as Parquet above.
- **Crash-mid-write**: SAFE in the streaming variant specifically (Feather V2 /
  Arrow **File** format is not — see next paragraph). Per the Arrow Columnar
  spec: the streaming format is "a sequence of encapsulated messages"; each
  message is self-framed by an 8-byte continuation+length prefix, and there is
  **no footer at all** — EOS is an *optional* trailing marker, not a
  requirement to parse what came before. A reader reads message by message and
  simply stops at the first incomplete/unreadable message, i.e., "readable up
  to the truncation point" is a property of the format itself, not a
  best-effort recovery hack. This is a real, structural difference from
  Parquet.
- **Arrow IPC File format (Feather V2)**: explicitly built as "the stream
  format" plus a magic-string header/footer wrapper for random access — inherits
  Parquet's exact disqualifying property (footer at the end, written once).
  **Do not use Feather/IPC-File for the live file; only the raw streaming
  variant is crash-safe.**
- **Compression**: pyarrow supports per-buffer LZ4/ZSTD compression inside the
  IPC format (Arrow's own "Buffer compression" spec extension), reasonably
  good, though not as strong as Parquet's per-column dictionary+RLE+codec
  stacking.
- **Read performance**: good — columnar record batches, but you pay
  read-the-whole-batch-to-get-any-row-in-it costs vs. Parquet's row-group
  statistics/predicate pushdown; no built-in "skip to timestamp t0" index the
  way a sorted binary file with memmap gives for free.
- **Verdict**: a legitimate, verified-safe candidate — the closest formal
  alternative to hand-rolled binary+memmap, with more type richness and
  built-in compression, at the cost of a much larger dependency (pyarrow) and
  a slightly worse random-access story for the specific t0..t1-by-symbol query.

## 2. Partitioning: what real systems do

Practitioner evidence, from an actual repository rather than a blog:

- **NautilusTrader's Parquet catalog** (`nautechsystems/nautilus_trader`, docs
  at `docs/integrations/tardis.md`) partitions as
  `{data_type}/{instrument_id}/{filename}`, **one file per UTC day per
  instrument**, filename carrying the ISO-8601 timestamp range, e.g.
  `order_book_deltas/BTC-PERPETUAL.DERIBIT/2023-10-01T00-00-00-000000000Z_...parquet`.
  This is the concrete answer to "file-per-what": **symbol (or instrument) is
  the top-level partition, day is the rolling boundary inside it** — not hour
  (too many files at this data volume), not one unbounded file (defeats
  range-pruning and crash containment), not by venue as the primary axis
  (venue is folded into the instrument identifier instead).
- Applied to this project's constraints (2-4 GB/day across ~30 symbols × 2
  venues ≈ tens of MB/symbol/day): **day-per-file is still the right rolling
  boundary** — hourly would produce ~60 partitions/day at this scale for
  negligible read-speed benefit, and a single unbounded file both defeats
  range-pruning and maximizes crash blast radius (see §3).
- `bmoscon/cryptofeed`'s actual backend implementations (checked directly via
  the GitHub API — `cryptofeed/backends/`) write to external systems
  (Arctic/MongoDB, InfluxDB, QuestDB, Postgres, Redis, Kafka), **not to
  Parquet/Arrow files on local disk at all**. This is useful negative evidence:
  a mature, widely-used crypto feed handler chose "hand it to a purpose-built
  time-series or document store" over "manage the file format directly" —
  which this project cannot do (those are servers, not embeddable libraries,
  and are out of scope per the settled stack).

**Recommended partition key for this project**: `{venue}/{symbol}/{date}.bin`
(or `.arrow` if the Arrow-IPC path is chosen), one open file per
(venue, symbol) at any time, rolled at UTC midnight. At 30 symbols × 2 venues
that is 60 files open concurrently at peak, well within normal fd limits, and
roughly 60 new files/day — nowhere near "millions of tiny files."

## 3. Parquet specifically — what happens on a killed writer

Concrete and confirmed from two independent sources (Parquet's own
error-recovery spec page, and a live bug report against DuckLake using
Parquet under the hood):

- The **footer** — the only place row-group offsets, schema, and the trailing
  magic bytes live — is written **once, at file close**. There is no
  incremental footer in the shipped pyarrow writer.
- A writer killed before `close()` leaves a file with real row-group bytes on
  disk but **no valid trailer**. Attempting to open it fails outright — the
  DuckLake issue's exact error is `No magic bytes found at end of file`. This
  is not partial data loss; it is total loss of that file, and if the catalog
  already registered it, the catalog entry itself needs manual cleanup too.
- **The standard mitigation is not "make Parquet crash-safe"** — it is "never
  let Parquet be the live-write format": accumulate into an in-memory buffer
  or a crash-safe intermediate format (binary/Arrow-stream), and **roll/convert
  to a closed, finalized Parquet file** on a schedule (e.g., end of UTC day),
  so the worst a crash costs is the still-open buffer for the current period,
  never a file that already has a `.parquet` extension. This matches exactly
  what NautilusTrader's day-boundary partitioning implies operationally, even
  though its docs don't spell out the crash-recovery reasoning explicitly.
- A proposed fix (periodic cumulative footer flush, `FlushWithFooter`) exists
  as an **open, unimplemented** Arrow issue (apache/arrow#40630) — do not plan
  around it being available.

## 4. What practitioners actually do

- **NautilusTrader** (`nautechsystems/nautilus_trader`): Parquet catalog,
  written via pyarrow datasets, **partitioned by instrument and UTC day**, used
  specifically for crypto tick/bar backtesting data sourced from Tardis. This
  is the closest real analog to this project's read pattern.
- **cryptofeed** (`bmoscon/cryptofeed`): a WebSocket feed handler with pluggable
  backends — confirmed by listing `cryptofeed/backends/` directly — targeting
  Arctic (MongoDB-backed columnar store), InfluxDB, QuestDB, Postgres, Redis,
  Kafka, GCP Pub/Sub. **No local Parquet/Arrow/HDF5 file backend exists in the
  shipped code.** The project's own choice is "delegate durability to a
  database," which this project has already ruled out by fixing the stack to
  SQLite + memmap + files.
- **Tardis.dev** (`tardis-dev/tardis-machine`): caches historical slices on
  disk **gzip-compressed**, decompressed on demand — i.e., a commercial,
  production crypto-tick provider uses plain compressed flat files for its
  local cache, not a database or columnar format, reinforcing that
  flat-file-plus-compression is a legitimate, battle-tested choice at this
  data class.

## 5. Compression that survives crash-only writing

- **gzip**: a gzip stream is a concatenation of independent "members"; each
  member has its own header/trailer. If you `flush()`/close a member after
  each rolled batch (rather than one giant member for the whole file),
  truncation loses only the incomplete trailing member — everything before it
  decompresses normally. This is exactly Tardis's approach (§4).
- **zstd**: same shape, at the frame level. A zstd stream can be multiple
  concatenated frames; each complete frame is independently decodable.
  Truncation mid-frame loses that frame (blocks within a frame depend on prior
  blocks in the same frame, so a partial frame is not recoverable), but
  **prior complete frames remain fully readable**. The mitigation is the same
  discipline as gzip: flush a new frame at each natural boundary (e.g., once
  per rolled time-window or once per N records), not one frame for an entire
  day. `zstandard` 0.25.0 has a cp314 wheel confirmed on PyPI (manylinux
  x86_64/aarch64 among others).
- **lz4**: frame format is block-based similarly; `lz4` 4.4.5 has a cp314
  wheel confirmed on PyPI. Weaker ratio than zstd generally.
- **Bottom line**: any of these codecs is crash-tolerant **only if the writer
  deliberately closes/flushes a compression frame at each rollover boundary**
  — none of them make an arbitrarily-long single compressed stream safe to
  truncate mid-frame.

## Comparison table

| Format | cp314 wheel | Crash mid-write | Compression | Range-by-symbol read | New dependency |
|---|---|---|---|---|---|
| Parquet (pyarrow) | Yes (25.0.1) | **Disqualifying** — whole file unreadable, no footer, `FlushWithFooter` unshipped | Best of the group | Excellent (row-group pruning) | pyarrow (large) |
| Binary + numpy dtype + memmap | N/A (numpy already pinned, cp314 confirmed) | Safe — worst case is one lost tail record | None intrinsic (offline pass only) | Excellent if 1 file/symbol/day | None |
| SQLite (stdlib, WAL) | N/A (stdlib) | Safe (WAL, torn frames discarded on recovery) — must explicitly enable WAL | None built-in | Fine, not columnar-fast | None |
| HDF5 (h5py) | Yes (3.16.0) | **Disqualifying** outside SWMR; SWMR narrows but the HDF Group itself says only "more difficult to corrupt," not safe, and journaling/WAL is still unshipped | Good | Good if chunked well | h5py (large) |
| Arrow IPC **Streaming** (`.arrows`) | Yes (same pyarrow wheel) | Safe — message-framed, no footer, reads up to first incomplete message | Good (per-buffer LZ4/ZSTD) | Good, no built-in predicate pushdown | pyarrow (large) |
| Arrow IPC **File**/Feather V2 | Yes | **Disqualifying** — same footer-at-end design as Parquet | Good | Excellent (random access) | pyarrow (large) |

## UNVERIFIED

1. **Did not empirically test truncated-file readability** for pyarrow's IPC
   stream reader or `ParquetWriter` on this box. `pip`/`venv` are not available
   on the system Python here (`/usr/bin/python3` has no `pip` module, and
   `python3 -m venv` fails for lack of `python3-venv`, and installing into the
   project's own `.venv` would have modified project state, which this
   research task should not do). The claims above rest on the Arrow/Parquet
   format specifications and a live third-party bug report (DuckLake), not on
   a test run in this environment.
2. **`python-zstandard` 0.25.0's exact per-frame-flush API** (`flush(FLUSH_FRAME)`
   or equivalent) was inferred from the general zstd frame-format spec, not
   verified against `python-zstandard`'s own API docs for that specific method
   name/behavior in this session.
3. **Concurrent-writer behavior of SQLite WAL at this project's ingestion
   rate** (60 open files, if SQLite were chosen for the tape itself rather
   than metadata) was not load-tested; the WAL single-writer-at-a-time
   constraint is documented but its practical throughput ceiling for this
   workload is not measured.
4. **How NautilusTrader's own pipeline detects and handles a truncated/corrupt
   Parquet file left by a prior crash** (i.e., does it delete-and-refetch, or
   fail loudly) was not found in the fetched doc page; only the partitioning
   scheme was confirmed.
5. **QuestDB and Arctic** (used by cryptofeed) were not evaluated against the
   crash-mid-write / cp314-wheel constraints at all, since they are external
   database servers rather than embeddable file formats and are out of scope
   given the settled stack (SQLite + memmap already fixed) — flagged here so
   this omission is explicit rather than silent.

## Recommendation

**Use a plain append-only binary file with a fixed numpy dtype, one file per
(venue, symbol, UTC day), read via `numpy.memmap`** — matching the pattern
already pinned in this project's stack (RL-058: file-backed `numpy.memmap` for
durable numeric state).

**The trade-off this accepts**: no built-in compression on the live file (a
day's file must be compressed or converted to Parquet/Arrow-stream *after* it
rolls and is closed, as a separate offline step, to get the storage-efficiency
benefit those formats offer) and no query engine — range-by-symbol reads are
hand-written slicing on a memory-mapped sorted-by-time array, not SQL or
Arrow/pyarrow compute. In exchange it is the only format evaluated where the
crash-only requirement is met **by construction** rather than by discipline
around when to roll files — the worst a `SIGKILL` costs is one unflushed
record, never a file, and it needs zero new dependencies beyond what's already
pinned and already confirmed on cp314.

If the offline compression step is wanted, Arrow IPC Streaming
(`pa.ipc.new_stream`, not the File/Feather-V2 variant) is the next-best choice
for the *closed, rolled* files — it is the only one of the richer formats that
is itself crash-tolerant if it ever needed to be written live, and it composes
cleanly with pyarrow's own conversion path to Parquet for long-term archival
once a day's data is finalized and provably closed.
