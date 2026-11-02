# quantalib-field-notes

Field notes on performance tricks observed in
[mihakralj/QuanTAlib](https://github.com/mihakralj/QuanTAlib)
(Apache-2.0), and their applicability to
[needle-cs](https://github.com/...) (zero-alloc quantized LLM inference;
AVX2 default, NEON port, NativeAOT C-ABI + fully-unmanaged Roslyn future).

All QuanTAlib observations are attributed to mihakralj/QuanTAlib under the
Apache-2.0 license. Quotes below are minimal excerpts used for attribution
and technical reference. File:line anchors refer to QuanTAlib checkout at
commit `c6ffb9f4` (main, 2026-11-02); they may drift as upstream evolves.

## Contents

- `Tricks.md` — catalog of every trick in all five areas (SIMD, memory,
  parallel/API/codegen, math, bench), with QuanTAlib file:line anchors.
- `Applicability.md` — verdict for every trick vs needle-cs:
  **USE** (file:line to change + expected gain + effort),
  **ADAPT** (what changes for int8/quant/unmanaged future), or
  **SKIP** (why). No maybes.
- `Repro.md` — how each headline claim was verified: the test/bench
  command that proves it.

## License note

This repo's own prose/analysis is original. All QuanTAlib code references
and minimal quotes remain under their original Apache-2.0 license and
copyright (mihakralj/QuanTAlib). See NOTICE-equivalent attribution in each
file header.
