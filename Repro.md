# Repro — how each headline claim was verified

Attribution: all repos/files below are mihakralj/QuanTAlib (Apache-2.0) except
where `needle-cs` is named. Commands run from the repo root unless stated.

## QuanTAlib C# head-to-head numbers

Claim: e.g. SMA 500K bars / period 220 — QuanTAlib Span 287.9 µs / 0 B vs TALib
364.8 µs / 32 B vs Wickra 1561.5 / 4 MB vs Skender 68377.5 / 42 MB vs Ooples
346187 / 151 MB (`docs/benchmarks.md:37-113`, `README.md:26-36`); per-mode EMA
table Span 423.8 µs / 0 B → Streaming 1536.8 / 176 B → Batch 2388.1 / 16 MB →
Eventing 3381.1 / 16.8 MB (`benchmarks.md:119-126`).

Verify:

```bash
cd perf
dotnet run -c Release -- --filter *Sma*     # single category (comment Benchmark.cs:21)
dotnet run -c Release                       # full 6-category head-to-head
```

Method: BenchmarkDotNet `Job.ShortRun.WithRuntime(Core10_0)`, `Mean+StdDev`,
`MemoryDiagnoser` (`Benchmark.cs:23-28`); inputs from one 500K-bar GBM
(`mu:0.05, sigma:0.2, seed:42`, `GlobalSetup :70-123`) with each rival's native
input (`double[]`, `List<Quote>`, `List<TickerData>`, `long[]`) materialized once.
Single measured env published: .NET 10.0.12 x64 RyuJIT, AVX-512F+CD+BW+DQ+VL+VBMI,
BDN 0.15.3 (`benchmarks.md:5-15`). No second machine / ARM run / tier-forced row
exists in-repo — "ratios hold" (`:15`) is unverified extrapolation.

## Period-sweep (anti-cherry-pick)

Claim: leadership holds across periods, not just period 220.

```bash
dotnet run --project progressive.csproj -c Release   # from perf/
```

`Progressive.cs:69-113` (`BarCount=1_000_000`, `[Params(10,50,100,500,1000,5000)]`),
`PivotPrinter.Print (:236-321)` pivots `Summary.Reports` into library×period
markdown. Fencing: each bench project `Compile Remove`s the sibling file
(`perf.csproj:17-20`, `progressive.csproj:26-29`); both state NativeAOT is not
BDN-compatible, bench JIT-only (`perf.csproj:12-13`).

## Python mirror numbers

Claim: quantalib vs pandas-ta/wickra/pandas-builtins incl. the honest WMA-603×
footnote (pandas has no native WMA → `rolling().apply(python lambda)`).

```bash
python tests/benchmark.py --bars 500000 --period 220 --iterations 10  # from python/
```

Same GBM params as C# (`benchmark.py:66-94`); warmup 2 + `perf_counter_ns`
mean/std (`:110-125`); degrades gracefully without optional deps (`:40-59`).
Pinned env recorded in-repo: py 3.12.10, numpy 2.2.6, pandas 3.0.1,
pandas-ta 0.4.71b0, wickra 1.0.6, quantalib 0.8.12 (`benchmarks.md:132-143`).

## Math headline claims (test-cited)

| Claim | Verify (from repo root) |
|---|---|
| erf max err ~2.5e-5 (`Normdist.cs:85`) | `dotnet test --filter Normdist.Validation` — `Normdist.Validation.Tests.cs:11-16` (`ApproxTolerance=1e-4` headroom), `:95-96` (`Φ(1.96)=0.975`), `:115-119` symmetry, `:135-154` MathNet cross-check |
| Lanczos ~15 digits / Lentz eps 1e-14 (`Betadist.cs:106`, `:176-244`) | `dotnet test --filter Betadist.Validation` — closed forms `I(0.25;2,2)=0.15625` at `1e-6` (`:78-91`), symmetry (`:100-104`) |
| atanh identity <1e-12; Fisher parity 1e-9 (`Fisher.cs:161-163`) | `dotnet test --filter Fisher.Validation` — `:43-55` identity, `:137-139` manual loop, `:369-375` Skender parity |
| EMA warmup-compensator first-bar exact (`Ema.cs:247-279`) | `pytest python/tests/test_ema.py` (`:128-130` `result[0]==100`); `dotnet test --filter Rma` (`Rma.Tests.cs:86-89` 95%-coverage bar count, `:105-111` `Rma==Ema(1/N)`) |
| DEMA/TEMA literal-composition 1e-9 | `dotnet test --filter Dema` (`Dema.Tests.cs:25-28` `2·E₁−E₂`) |
| One-vs-two-pass gap quantified (rolling `1e-7` vs elsewhere `1e-10`) | `dotnet test --filter StdDev.Validation` — `:10-12` tolerances, `:351-361` `StdDev==√Variance`, `:520-525` pop-vs-sample `{100,110}` |
| CMA Welford 1e10-base stability | `dotnet test --filter Cma.Validation` (`Cma.Validation.Tests.cs:125-150`) |
| Idempotent dispose under 10× `Task.Run` | `dotnet test --filter Lsma` (`Lsma.Tests.cs:260-274`) |

## Negative / audit findings (verified by reading, not running)

- **No CI perf gate**: all three workflows read end-to-end — `Publish.yml`
  (ReSharper/Snyk/Semgrep/Sonar/DeepSource/Codacy, perf excluded from
  restore/build `:69-77,145-149`), `Release.yml` (6-RID NativeAOT wheel matrix,
  no threshold), `pages.yml` (docs). No bench job, threshold, baseline file,
  hardware matrix, or BDN artifact upload. Agent playbook asks for benchmarks
  per change (`beast.agent.md`) as process, not gate.
- **No over-read**: stencil loads start at `i=lag` (`Slope/Accel/Jerk`), sliding
  cores load `i−period` with `i≥period` and exact-multiple `simdEnd`
  (`Sma.cs:460-463`), `vectorEnd=len−len%W` loops use `i<vectorEnd`
  (`ErrorHelpers.cs:1037`) — verified by reading, no fuzz run in-repo.
- **Unverified framing** (reproducible numbers, loaded presentation): "0.58
  ns/value" (`README.md:24`, vectorized-batch division, no latency experiment);
  cross-hardware ratio claim (`benchmarks.md:15`, one AVX-512 machine); "FMA
  explains most of the speedup" (`:33,51-53`, no no-FMA ablation/disassembly);
  competitor-shape asymmetry disclosed (`README.md:36`) but ranked in one column.

## needle-cs side (what already exists — no repro needed)

- Alloc gates: `src/Needle.Cli/BenchE2E.cs:118-123,273-292` (`--max-managed-bytes/
  --max-native-allocs`, nonzero exit) + `tests/Needle.Tests/BenchHarnessTests.cs`.
- Protocol: `tools/bench_ab.py` (interleaved A/B), `BenchStats.cs:47-66`
  (best-of-9), machine stamp `BenchStats.cs:170-186`, `tools/native_bench.py`
  (upstream `.so` via ctypes), `tools/malloc_counter.c` (`LD_PRELOAD` cross-check).
- Gaps to close per `Applicability.md` BENCH rows: `--contexts` sweep (BENCH-01),
  versions object (BENCH-02), standing per-mode table (BENCH-03), consolidated
  `docs/benchmarks.md` (BENCH-04), FFI note (BENCH-05), `bench_ab.py`
  `--threshold-pct` (BENCH-07 delta).
