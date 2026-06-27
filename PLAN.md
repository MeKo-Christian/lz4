# LZ4 Performance Plan

> **Agent note:** Implement task-by-task with `superpowers:subagent-driven-development` or `superpowers:executing-plans`. Track progress with the checkboxes below.

**Goal:** Improve real-world LZ4 compression/decompression throughput without changing block/frame format semantics or correctness guarantees.

**Approach:** Keep the work evidence-driven: fix benchmarks first, optimize the Go block compressor, add narrow assembly only where profiles prove it helps, then revisit stream overhead.

## Current Findings

- The fast block compressor is the main Go-side hotspot. Prior profiles point at hash-table probing/bookkeeping and repeated little-endian loads in `internal/lz4block.(*Compressor).CompressBlock`.
- Existing `amd64` decode assembly is already valuable: `BenchmarkUncompressPg1661` was about `291 us/op` with assembly vs `856 us/op` with `-tags noasm`.
- Checksum work is a major frame-decode cost after decode assembly, with `internal/xxh32.updateGo` previously around one quarter of stream decode CPU time.
- Benchmark fidelity needed repair: one decode benchmark used a framed `.lz4` file as a raw block, and stream-compress benchmarks did not reset the backing buffer.
- Stream concurrency may have overhead from per-block goroutines/channels, but it should be measured after the codec/checksum work.

## Optimization Order

1. Repair and expand benchmarks/profiling.
2. Optimize the fast block compressor in pure Go.
3. Add checksum assembly for `amd64`; consider `arm64` only after measuring impact.
4. Re-measure frame paths with checksums on/off.
5. Revisit stream concurrency overhead.
6. Decide whether any additional assembly is justified.

## Completed Work

### Task 1: Benchmark Fidelity

**Files:** `bench_test.go`, optional `internal/lz4block/bench_test.go`

- [x] Benchmark raw block decompression with a real raw block, not a framed `.lz4` file.
- [x] Reset the backing `bytes.Buffer` in stream-compress benchmarks.
- [x] Add checksum-on/off stream reader and writer benchmarks.
- [x] Add concurrent frame encode/decode benchmarks using `ConcurrencyOption(runtime.GOMAXPROCS(0))`.
- [x] Use names that distinguish block vs frame and checksum vs no-checksum paths.
- [x] Capture baseline `go test -run '^$' -bench . -benchmem`.
- [x] Capture baseline `go test -run '^$' -bench . -benchmem -tags noasm`.
- [x] Save baseline numbers in a commit message or short benchmark note before hot-code changes.

### Task 2: Fast Go Compressor

**Files:** `internal/lz4block/block.go`, `internal/lz4block/block_test.go`, benchmarks

- [x] Profile repaired block-compress benchmarks and confirm the hot symbols.
- [x] Simplify `get`/`put` overhead where possible.
- [x] Benchmark single-entry table layout vs the existing table plus bitmap.
- [x] Evaluate lower-overhead little-endian loads in the match scan.
- [x] Test cheaper hash or `hashLog` changes without unacceptable ratio loss.
- [x] Add and measure an early incompressible-block bailout.
- [x] Verify format compatibility with block round-trip tests and fuzz-style corpora.

**Success criteria:** Measurable block-compress improvement with no regressions in block tests, representative corpora, or compression ratio.

### Task 3: Checksum Assembly

**Files:** `internal/xxh32/xxh32zero_amd64.*`, optional `internal/xxh32/xxh32zero_arm64.*`, `internal/xxh32/xxh32zero_test.go`

- [x] Implement `ChecksumZero` and `update` for `amd64` in Plan 9 assembly.
- [x] Keep Go fallback paths for correctness and portability.
- [x] Add focused `ChecksumZero` and streaming `update` benchmarks.
- [x] Re-run frame decode benchmarks with checksums enabled and disabled.
- [x] Decide whether to do `arm64` immediately.

**Decision:** Defer `arm64`; first measure broader frame-level wins from the `amd64` path.

## Remaining Work

### Task 4: Stream Concurrency

**Files:** `internal/lz4stream/block.go`, `writer.go`, `reader.go`, `bench_test.go`

- [x] Measure concurrent encode/decode with large blocks and multiple cores after Tasks 1-3.
- [x] Replace writer-side per-block compression goroutine creation with a reusable worker pool.
- [x] Evaluate reader-side worker pooling; defer because the prototype regressed concurrent decompression.
- [x] Evaluate nested channel orchestration; defer replacement because profiles/benchmarks do not justify the broader rewrite.
- [x] Preserve ordered output and first-error semantics.
- [x] Re-check memory retention and buffer-pool behavior.

**Task 4 note:** Fixed a concurrent writer reuse deadlock where `Reset` tried to close an already-drained concurrent block manager. Concurrent frame compression now uses a bounded worker pool instead of one goroutine per block, and `Writer.ReadFrom` no longer double-calls `OnBlockDone`. Reader-side worker pooling was prototyped and reverted because it slowed concurrent decompression; current `benchmem`/profile results still show high reader allocations, but not enough evidence for a broader channel orchestration rewrite in this pass.

**Success criteria:** Better throughput for `ConcurrencyOption(n>1)` without higher allocations or single-threaded regressions.

### Task 5: More Assembly Decision

**Files:** `internal/lz4block/block.go`, `internal/lz4block/decode_amd64.s`

- [x] Re-profile after Tasks 1-4.
- [x] Leave decoder assembly alone unless measurements show a specific missed fast path.
- [x] If compressor time is still concentrated in a tiny stable loop, prototype a narrow `amd64` helper.
- [x] Reject broad encoder assembly unless repaired benchmarks show a large enough win to justify maintenance cost.

**Task 5 note:** Fresh `amd64` profiles still show block decompression dominated by the existing `decodeBlock` assembly, with `BenchmarkBlockDecompress` around `160 us/op` with assembly versus `499 us/op` with `-tags noasm`; no missed decoder fast path was isolated. Frame decode with checksums remains split between `decodeBlock` and `internal/xxh32.update` (`Pg1661ChecksumOn` profile: about 63% decode, 30% checksum), so additional decoder assembly is not the next lever. Fast block compression is still dominated by `internal/lz4block.(*Compressor).CompressBlock`, but line-level samples are spread across the Go hash/probe/match loop (`get`, `put`, `blockHash`, little-endian loads, and match extension) rather than a tiny stable helper. A narrow `amd64` helper is therefore not justified, and broad encoder assembly is rejected for this pass under the maintenance-cost decision rule.

**Decision rule:** Do not write large compressor assembly until Go-level structural changes are exhausted and profiles prove it is worthwhile.

## Verification

- [ ] Run `go test ./...`.
- [ ] Run `go test -run '^$' -bench . -benchmem`.
- [ ] Run `go test -run '^$' -bench . -benchmem -tags noasm`.
- [ ] Compare ratio and throughput on `pg1661`, `Mark.Twain-Tom.Sawyer`, `e.txt`, and `random.data`.
- [ ] Verify checksum-enabled frame decode still rejects corrupted data.
- [ ] Verify block and frame fuzz/regression tests still pass.

## Exit Criteria

- Benchmarks measure real block/frame paths correctly.
- Fast block compression is faster on representative corpora.
- Checksum-enabled frame decode is faster on `amd64`.
- Any new assembly has tests, `noasm` fallbacks, and benchmark justification.
- The repo has a repeatable measurement workflow for future performance work.
