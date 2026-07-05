# caustic-compact

A from-scratch compression library written in [Caustic](https://github.com/Caua726/Caustic), a systems language with its own compiler, assembler, and linker. It decompresses and compresses eleven formats — gzip, zlib, DEFLATE, LZ4, Snappy, LZMA, LZMA2, xz, bzip2, Zstandard, and Brotli — with no libc, no zlib, no external libraries. Just the Caustic standard library and Linux syscalls.

Every decoder and encoder interoperates with the standard tools: our gzip reads in `gunzip`, our `.xz` opens in `xz -d`, `.bz2` in `bzip2 -d`, `.zst` in `zstd -d`, `.br` in `brotli -d`, `.lz4` in `lz4 -d`, and framed Snappy round-trips against `python-snappy`.

```sh
build/cctool c gzip file > file.gz   # compress with any of the eleven formats
build/cctool d file.gz               # decompress (format autodetected from magic bytes)
```

Around 7,700 lines of Caustic across ~21 modules — one file per format under `formats/`, shared building blocks under `core/` — plus Brotli's embedded 122 KB static dictionary. x86_64 Linux.

## Formats

| Format | Decode | Encode |
|---|---|---|
| DEFLATE / zlib / gzip — RFC 1951 / 1950 / 1952 | yes | dynamic Huffman + hash-chain LZ77 with lazy matching + stored fallback |
| LZ4 — frame + block | yes | hash-chain matcher + lazy matching |
| Snappy — raw block + framed `.sz` | yes | greedy hash matcher (+ CRC-32C framing) |
| LZMA / LZMA2 / xz | yes | range coder + hash-chain matcher, rep-matches, lazy matching |
| bzip2 | yes | BWT (suffix sort) + MTF + Huffman, multi-block |
| Zstandard — RFC 8878 | yes | FSE-coded sequences + repeat-offsets + lazy matching (raw literals) |
| Brotli — RFC 7932 | yes | LZ77 + Huffman + repeat-offsets + lazy matching |

The keystone is `formats/inflate.cst` (raw DEFLATE); zlib, gzip, and PNG's IDAT stream are thin containers over it. zstd and brotli are full decoders — brotli ships the entire 122 KB static dictionary and its 121 word transforms; zstd has the FSE entropy decoder, Huffman literals, and the sequence machine with its offset history.

### Decode coverage

Beyond the base formats, the decoders handle the awkward corners of each spec:

- **gzip** — multi-member (concatenated) streams and FHCRC header-CRC verification.
- **xz** — the delta filter and the BCJ branch converters (x86, ARM, ARM-Thumb, PowerPC, SPARC), applied in reverse over the LZMA2 output.
- **brotli** — the incompatible large-window variant (WBITS up to 30).
- **zstd** — dictionaries, both raw-prefix and trained (`zstd -D`): the entropy tables, repeat offsets, and content are loaded from the dictionary.
- **bzip2** — the deprecated randomized-block flag (table extracted verbatim from libbz2).
- **Snappy** — the framed `.sz` stream with masked CRC-32C, alongside the raw block format.

Every checksum is verified on decode (CRC-32, Adler-32, xxHash64, CRC-32C, per-block and stream CRCs).

## Performance

The library was profiled with `perf` and tuned; the decoders now run within a small factor of the reference tools, and on some workloads faster. Representative single-threaded throughput:

- **Decompression** — brotli and lz4 decode *faster* than the reference tools on repetitive data; gzip decode reaches ~85 MB/s (2.4× the first working version) and zstd ~110 MB/s (~2×) on source code.
- **Compression, real-world data** — because `gzip -9`, `zstd -19`, and `brotli -q11` are slow at their maximum effort, our compressors produce output *faster* than them on source code while staying close in size.
- **Compression, size** — we match or beat `gzip -9`, `xz -9`, `bzip2 -9`, and `lz4 -9` on the test corpus; on highly-compressible real text we trail `zstd -19` and `brotli -q11`, which use optimal parsing (see below).

The high-impact wins came from profiling, not intuition: replacing per-operation function wrappers (xxHash64's u64 helpers were 26 % of zstd decode), single 64-bit loads in the bit readers, lookup tables for what were linear scans (brotli's 704-entry command search), 8-byte match comparison and copy, a 9-bit Huffman root table, slice-by-8 CRC-32, and a Manber-Myers singleton-skip in the bzip2 suffix sort.

## What's not there yet

An honest list of where it stops short of the reference implementations. These are compression-ratio and memory refinements — the formats themselves are complete and interoperable.

- **Optimal parsing.** The matchers are greedy hash chains with one-step lazy matching. On highly-compressible real text this trails `zstd -19`/`brotli -q11`, whose optimal parsers (btultra2 and friends) choose globally better match/literal splits. On binary and synthetic data the gap closes or reverses.
- **zstd literals are raw**, not Huffman-coded; the sequences are FSE-coded with repeat-offsets and cross-block matching. High-entropy literals (the common case) cost the same either way.
- **brotli** has no context modeling or static-dictionary *references* on the encode side (it ships and uses the dictionary on decode). Distance repeat-codes are used.
- **bzip2 encode uses one Huffman table per block** instead of the reference's up-to-six with per-group selectors. Skipping that overhead actually makes it *beat* `bzip2 -9` on this corpus.
- **Not constant-memory.** The codecs are one-shot; `stream.cst` gives fd I/O with magic autodetection and `examples/cctool` is a full CLI, but the backward-reading decoders (zstd, brotli) buffer the whole compressed input rather than streaming it.

## Build and test

The Caustic toolchain (`caustic`) must be on your `PATH`; the standard library resolves from the install path. The test runner is a Caustic program, and the compressed test vectors under `tests/fixtures/` are committed as data — there are no shell or other-language scripts in the repository.

```sh
caustic tests/run.cst -o tests/runtests   # build the runner
tests/runtests                            # decode + encode round-trips; exit 0 = all green
caustic -c caustic_compact.cst            # compile-check the whole library (no main)
caustic examples/cctool.cst -o cctool     # build the CLI
```

## Layout

```
core/      errno · bytes (mmap-backed) · bitreader / bitwriter · huffman (9-bit table)
           checksum (CRC-32 slice-by-8 · Adler-32 · xxHash32 / xxHash64 · CRC-32C)
formats/   inflate · zlib · gzip · deflate · lz4 · snappy · lzma · lzma2 · xz · bzip2 · zstd · brotli
           brotli_dict (122 KB dictionary) · brotli_transforms
examples/  cctool (compress/decompress any format, autodetect) · zcat · gzc
stream.cst (fd-oriented streaming layer with format autodetection)
tests/     run.cst (the runner) · fixtures/ (committed vectors)
```

Conventions, like the sibling [`caustic-net`](https://github.com/Caua726/caustic-net): errors are negative `i64` codes (`core/errno.cst`); the output buffer is `mmap`-backed so it grows past the 64 KB cap of the `bins` slab allocator; `snake_case` functions, `PascalCase` structs, `_prefix` private, `SCREAMING` constants. No generic `Result`/`Option` (the compiler can't construct them).
