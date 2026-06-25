# caustic-compact

A pure-Caustic, zero-dependency (de)compression library — DEFLATE/zlib/gzip
today, with Brotli/Zstd/LZ4/Snappy and encoders landing phase by phase. Built
on nothing but the Caustic stdlib (`std/mem`, `std/io`): **no libc, no zlib, no
external libraries**. Philosophy (same as caustic-net): direct functions, one
file per format, no framework.

## Keystone: `inflate`

Everything web decodes through one routine — raw DEFLATE (RFC 1951):

```cst
fn inflate(src as *u8, src_len as i64, out as *bytes.Bytes) as i64;  // bytes, or -err
```

zlib (RFC 1950), gzip (RFC 1952) and PNG IDAT are thin containers over it:

```cst
fn zlib_decompress(src as *u8, src_len as i64, out as *bytes.Bytes) as i64;  // + Adler-32
fn gzip_decompress(src as *u8, src_len as i64, out as *bytes.Bytes) as i64;  // + CRC-32 + ISIZE
```

Output accumulates in `core/bytes.cst`'s `Bytes` — a growable buffer backed
directly by `mmap` (so decompressed output can grow without the 64 KB cap of the
`bins` slab allocator).

## Layout

```
core/      errno · bytes (mmap-backed) · bitreader (LSB-first) · huffman (canonical) · checksum (CRC-32/Adler-32)
formats/   inflate · zlib · gzip · deflate · lz4 · snappy · lzma · lzma2 · xz · bzip2 · zstd · brotli*
examples/  zcat (gunzip-to-stdout)
tests/     run.sh · gen_fixtures.sh · test_*
```
`*` = on the roadmap, not yet landed.

## Conventions

- **Errors:** every entry point returns a non-negative count on success or a
  negative code from `core/errno.cst` (`Z_EOF`, `Z_BADHUFF`, `Z_BADDIST`,
  `Z_BADSUM`, …). No generic `Result`/`Option` (the compiler can't construct them).
- **Memory:** small fixed structures (Huffman tables, constant tables) use the
  stdlib `bins` allocator per module; the unbounded output buffer uses `mmap`
  via `core/bytes.cst`.
- **Naming:** `snake_case` fns, `PascalCase` structs, `_prefix` private,
  `SCREAMING` constants `with imut`.

## Status

| Phase | Module | State |
|---|---|---|
| 0 | `core/` errno · bytes · bitreader · huffman · checksum | ✅ green (`tests/test_core`) |
| 1 | `formats/inflate` (stored + fixed + dynamic, overlap-safe LZ77) | ✅ green (`tests/test_inflate`, 10 fixtures) |
| 2 | `formats/zlib` · `formats/gzip` (+ checksum/ISIZE verify) | ✅ green (`tests/test_gzip`, incl. system `gzip`) |
| 3 | `formats/deflate` encode (+ `core/bitwriter`) | ✅ green (`tests/test_deflate` round-trips) |
| 4 | gzip/zlib encode (→ HTTP gzip, PNG IDAT) | ✅ green (output reads in system `gunzip`) |
| 5 | `formats/lz4` decode (block + frame, xxHash32) | ✅ green (incl. system `lz4`) |
| 6 | `formats/snappy` decode (varint + copy types 1/2/3) | ✅ green |
| 7 | `formats/lzma` · `formats/lzma2` · `formats/xz` decode (range coder) | ✅ green (incl. system `xz`) |
| 8 | `formats/bzip2` decode (Huffman + MTF + inverse BWT + RLE) | ✅ green (incl. system `bzip2`) |
| 9 | `formats/zstd` decode (RFC 8878, FSE + Huffman + sequences) | ✅ green (incl. system `zstd -19`) |
| 10 | `formats/brotli` decode (RFC 7932) | ⏳ next |

## Build & test

The Caustic toolchain (`caustic`) must be installed; the stdlib resolves from
the install path, so `use "std/mem.cst"` just works. The test runner is itself
a Caustic program (`tests/run.cst`) — no shell or other languages in the repo;
the compressed test vectors under `tests/fixtures/` are committed as data.

```sh
caustic tests/run.cst -o tests/runtests   # build the test runner
tests/runtests                            # run every suite, exit 0 = all green
caustic -c caustic_compact.cst            # compile-check the whole library (no main)
caustic examples/zcat.cst -o zcat && ./zcat file.gz   # decompress a gzip file
```
