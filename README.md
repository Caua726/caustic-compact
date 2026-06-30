# caustic-compact

A from-scratch compression library written in [Caustic](https://github.com/Caua726/Caustic), a systems language with its own compiler, assembler, and linker. It decompresses and compresses eleven formats — gzip, zlib, DEFLATE, LZ4, Snappy, LZMA, LZMA2, xz, bzip2, Zstandard, and Brotli — with no libc, no zlib, no external libraries. Just the Caustic standard library and Linux syscalls.

Every decoder and encoder interoperates with the standard tools: our gzip reads in `gunzip`, our `.xz` opens in `xz -d`, `.bz2` in `bzip2 -d`, `.zst` in `zstd -d`, `.br` in `brotli -d`.

```sh
build/zcat file.gz         # decompress a gzip file to stdout
build/gzc  file > file.gz  # compress a file to gzip
```

Around 4,600 lines of Caustic across ~20 modules — one file per format under `formats/`, shared building blocks under `core/` — plus Brotli's embedded 122 KB static dictionary. x86_64 Linux.

## Formats

| Format | Decode | Encode |
|---|---|---|
| DEFLATE / zlib / gzip — RFC 1951 / 1950 / 1952 | yes | yes — fixed Huffman + LZ77 |
| LZ4 — frame + block | yes | yes — greedy hash matcher |
| Snappy — raw block | yes | yes — greedy hash matcher |
| LZMA / LZMA2 / xz | yes | yes — range coder + greedy matcher |
| bzip2 | yes | yes — BWT + MTF + Huffman |
| Zstandard — RFC 8878 | yes | yes — FSE-coded sequences (raw literals) |
| Brotli — RFC 7932 | yes | store mode (uncompressed meta-blocks) |

The keystone is `formats/inflate.cst` (raw DEFLATE); zlib, gzip, and PNG's IDAT stream are thin containers over it. zstd and brotli are full decoders — brotli ships the entire 122 KB static dictionary and its 121 word transforms; zstd has the FSE entropy decoder, Huffman literals, and the sequence machine with its offset history.

The whole library, decode and encode, is exercised by one test runner (itself a Caustic program) against vectors produced by the real system tools.

## What's not there yet

This is an honest list of where it stops short of the reference implementations.

- **zstd encode uses raw literals** — the LZ77 sequences are FSE-coded with cross-block matching, which is competitive with `zstd -19` on the test corpus (often within a few percent, sometimes smaller). Literals are not yet Huffman-coded, so inputs whose post-match literals are *both* large and low-entropy would lag; high-entropy literals (the common case) cost the same either way.
- **brotli encode is store mode** — valid and tool-decodable, but does not actually compress. The real prefix-code/context encoder is not written.
- **DEFLATE encode uses fixed Huffman only** — no dynamic Huffman and no stored-block fallback, so incompressible data grows a few percent.
- **bzip2 encode uses two identical Huffman tables** instead of the reference's up-to-six (multi-block for inputs over 900 KB is supported).
- **Decode does not verify checksums** for zstd / xz / bzip2 (gzip / zlib / lz4 do).
- **Snappy is the raw block format**, not the framed `.sz` stream format.
- One-shot and in-memory — no streaming API. No large-window Brotli, no xz delta/BCJ filters, no dictionaries.

## Build and test

The Caustic toolchain (`caustic`) must be on your `PATH`; the standard library resolves from the install path. The test runner is a Caustic program, and the compressed test vectors under `tests/fixtures/` are committed as data — there are no shell or other-language scripts in the repository.

```sh
caustic tests/run.cst -o tests/runtests   # build the runner
tests/runtests                            # decode + encode round-trips; exit 0 = all green
caustic -c caustic_compact.cst            # compile-check the whole library (no main)
```

## Layout

```
core/      errno · bytes (mmap-backed) · bitreader / bitwriter · huffman · checksum (CRC-32 / Adler-32 / xxHash32)
formats/   inflate · zlib · gzip · deflate · lz4 · snappy · lzma · lzma2 · xz · bzip2 · zstd · brotli
           brotli_dict (122 KB dictionary) · brotli_transforms
examples/  zcat (gunzip-to-stdout) · gzc (gzip a file)
tests/     run.cst (the runner) · fixtures/ (committed vectors)
```

Conventions, like the sibling [`caustic-net`](https://github.com/Caua726/caustic-net): errors are negative `i64` codes (`core/errno.cst`); the output buffer is `mmap`-backed so it grows past the 64 KB cap of the `bins` slab allocator; `snake_case` functions, `PascalCase` structs, `_prefix` private, `SCREAMING` constants. No generic `Result`/`Option` (the compiler can't construct them).

## License

MIT — see [LICENSE](LICENSE).

The compression *formats* are open specifications, implemented here from their RFCs. A few parts are ported from, or embed data from, permissively licensed reference implementations (Brotli — MIT; the Zstandard educational decoder — BSD; bzip2 — BSD-style; xxHash — BSD; LZMA — public domain). None are copyleft. Their copyright notices are in [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
