# Benchmark and Regression Workflow

Use this workflow when evaluating performance-sensitive changes. Run commands on an otherwise idle machine and record the CPU model, operating system, Go version, commit, and whether assembly is enabled.

## Correctness

Run the complete test suite first:

```sh
go test ./...
```

Run race-enabled tests before changing stream concurrency code:

```sh
go test -race ./...
```

Run fallback-path tests before changing assembly or code guarded by build tags:

```sh
go test -tags noasm ./...
```

## Benchmarks

Run the full benchmark suite with allocation reporting:

```sh
go test -run '^$' -bench . -benchmem
```

Run the same benchmark surface without assembly:

```sh
go test -run '^$' -bench . -benchmem -tags noasm
```

For noisy results, repeat targeted benchmarks and compare medians instead of a single run:

```sh
go test -run '^$' -bench '^BenchmarkBlockCompress/Pg1661Fast$' -benchmem -count=5
go test -run '^$' -bench '^BenchmarkBlockDecompress$' -benchmem -count=5
go test -run '^$' -bench '^BenchmarkFrameDecompress/Pg1661ChecksumOn$' -benchmem -count=5
```

The top-level benchmark fixtures cover representative compressible, text, numeric, and incompressible corpora:

- `testdata/pg1661.txt.gz`
- `testdata/Mark.Twain-Tom.Sawyer.txt.gz`
- `testdata/e.txt.gz`
- `testdata/random.data.gz`

## Profiles

Capture CPU profiles for the hot paths before deciding on low-level changes:

```sh
go test -run '^$' -bench '^BenchmarkBlockCompress/Pg1661Fast$' -benchtime=5s -benchmem -cpuprofile /tmp/lz4-block-compress.cpu
go tool pprof -top /tmp/lz4-block-compress.cpu
go tool pprof -list CompressBlock /tmp/lz4-block-compress.cpu

go test -run '^$' -bench '^BenchmarkBlockDecompress$' -benchtime=5s -benchmem -cpuprofile /tmp/lz4-block-decompress.cpu
go tool pprof -top /tmp/lz4-block-decompress.cpu

go test -run '^$' -bench '^BenchmarkFrameDecompress/Pg1661ChecksumOn$' -benchtime=5s -benchmem -cpuprofile /tmp/lz4-frame-decompress-checksum.cpu
go tool pprof -top /tmp/lz4-frame-decompress-checksum.cpu
```

Only add assembly when profiles identify a small, stable hot path whose speedup justifies an architecture-specific implementation and a `noasm` fallback.

## Corruption Regressions

Checksum and malformed-input behavior is part of compatibility. Run the focused regression subset when touching frame parsing, checksums, or decompression:

```sh
go test ./... -run 'Test(BlockChecksumMismatch|CloseRChecksumMismatch|DescriptorInitRBadChecksum|UncompressBadBlock)'
```

## Fuzz Harness

The `fuzz/` module uses the older `go-fuzz` API. Keep it compiling against the current library:

```sh
(cd fuzz && go test ./...)
```

For deeper local fuzzing, install `go-fuzz` tools and run one target at a time:

```sh
cd fuzz
go install github.com/dvyukov/go-fuzz/go-fuzz@latest
go install github.com/dvyukov/go-fuzz/go-fuzz-build@latest
go-fuzz-build
go-fuzz -bin=./lz4-fuzz.zip -func=Fuzz -workdir=corpus -procs=1
```
