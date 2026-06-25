# Third-party notices

caustic-compact implements open, published compression formats (the IETF RFCs
and public format specifications for DEFLATE, zlib, gzip, LZ4, Snappy, LZMA,
xz, bzip2, Zstandard, and Brotli). Implementing a format from its specification
carries no licensing obligation.

Some parts, however, are ported from — or embed data from — permissively
licensed reference implementations. None are copyleft/GPL. Their copyright
notices and licenses are reproduced below, as required by those licenses.

---

## Brotli — `formats/brotli.cst`, `formats/brotli_dict.cst`, `formats/brotli_transforms.cst`

The Brotli decoder is a port of the reference Java decoder (`org.brotli.dec`).
The 122,784-byte static dictionary and the 121 word transforms are the data
from the reference implementation.

> Copyright (c) 2009, 2010, 2013-2016 by the Brotli Authors.
>
> Licensed under the MIT License. Permission is hereby granted, free of charge,
> to any person obtaining a copy of this software and associated documentation
> files (the "Software"), to deal in the Software without restriction,
> including without limitation the rights to use, copy, modify, merge, publish,
> distribute, sublicense, and/or sell copies of the Software, and to permit
> persons to whom the Software is furnished to do so, subject to the inclusion
> of the above copyright notice. THE SOFTWARE IS PROVIDED "AS IS", WITHOUT
> WARRANTY OF ANY KIND.

Source: https://github.com/google/brotli (MIT License)

---

## Zstandard — `formats/zstd.cst`

The Zstandard decoder is a port of the educational decoder
(`doc/educational_decoder/zstd_decompress.c`).

> Copyright (c) Meta Platforms, Inc. and affiliates. All rights reserved.
>
> This source code is licensed under both the BSD-style license (found in the
> LICENSE file in the root directory of the zstd source tree) and the GPLv2
> (found in the COPYING file in the root directory of the zstd source tree).
> This port is used under the BSD-style license.

Source: https://github.com/facebook/zstd (BSD / GPLv2 dual license; used under BSD)

---

## bzip2 — `formats/bzip2.cst`

The length-limited Huffman code-length builder (`_mk_lengths`,
`BZ2_hbMakeCodeLengths`) follows the bzip2 reference.

> This program, "bzip2", the associated library "libbzip2", and all
> documentation, are copyright (C) 1996-2010 Julian R Seward. All rights
> reserved. Redistribution and use in source and binary forms, with or without
> modification, are permitted provided that the conditions of the bzip2 BSD-style
> license are met (see https://sourceware.org/bzip2/).

Source: https://sourceware.org/bzip2/ (BSD-style license)

---

## xxHash — `core/checksum.cst`

The xxHash32 implementation used for LZ4 frame checksums follows the xxHash
specification and reference.

> xxHash - Extremely Fast Hash algorithm
> Copyright (C) 2012-present, Yann Collet.
> BSD 2-Clause License.

Source: https://github.com/Cyan4973/xxHash (BSD 2-Clause License)

---

## LZMA — `formats/lzma.cst`, `formats/lzma2.cst`

The LZMA range coder and state machine follow the LZMA specification. Igor
Pavlov placed the LZMA SDK in the **public domain**, which imposes no
restrictions.

Source: https://www.7-zip.org/sdk.html (public domain)

---

The DEFLATE/zlib/gzip, LZ4 (block + frame), and Snappy (raw block)
implementations were written from their published format specifications and do
not derive from any specific implementation's source code.
