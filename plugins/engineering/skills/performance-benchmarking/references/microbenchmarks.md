# Microbenchmarks by ecosystem

Read this before writing or reviewing a function-level benchmark, or before trusting a number one produced.

## Rules that apply to every harness

- Benchmark the function the profile blamed, with inputs shaped like production: realistic sizes, several distinct values, and the same mix of edge cases the fast path special-cases.
- Hide inputs from the optimizer and consume every output.
- Keep setup out of the timer and real costs (GC, drop, allocation) inside it.
- Keep baseline and candidate in the same file or the same harness run so they share machine state. Save the baseline under a name and compare against it.
- The base-then-candidate recipes below are the minimum. If you run all of A and then all of B, the variant that runs second inherits a hotter chip, a different neighbor or a background job. When the effect is near the noise floor, build both once and run them as alternating pairs, as the Go example does and environment-and-ci.md describes.
- Run the correctness gate on the candidate first. A benchmark of a wrong function is worthless.
- Check scaling: 10x the input should take roughly the time the algorithm predicts. A flat line means the work was optimized away or cached.

## Rust: criterion

```toml
# Cargo.toml
[dev-dependencies]
criterion = "0.8.2"

[[bench]]
name = "parse"
harness = false
```

```rust
// benches/parse.rs
use criterion::{criterion_group, criterion_main, BatchSize, BenchmarkId, Criterion};
use std::hint::black_box;

fn bench_parse(c: &mut Criterion) {
    let corpus = mycrate::fixtures::load_header_corpus();
    let mut group = c.benchmark_group("parse_header");
    for (name, input) in corpus.iter().map(|s| (s.name.as_str(), s.bytes.as_slice())) {
        group.bench_with_input(BenchmarkId::from_parameter(name), input, |b, input| {
            b.iter(|| black_box(mycrate::parse_header(black_box(input))))
        });
    }
    group.finish();
}

fn bench_sort(c: &mut Criterion) {
    let rows = mycrate::fixtures::load_rows();
    c.bench_function("sort_rows", |b| {
        // The clone runs in the untimed setup closure; only the sort is measured.
        b.iter_batched(|| rows.clone(), |mut r| { mycrate::sort_rows(&mut r); r }, BatchSize::SmallInput)
    });
}

criterion_group!(benches, bench_parse, bench_sort);
criterion_main!(benches);
```

- `criterion::black_box` is deprecated. Use `std::hint::black_box`. It is best effort, so check scaling as well.
- `b.iter` times the routine and the drop of its output. If the routine consumes an input that is expensive to build, use `iter_batched` so setup is not timed.
- Defaults: 100 samples, 3 s warm-up, 5 s measurement, 1% noise threshold, 0.05 significance level. Criterion reports "no change" for differences inside the noise threshold even if they are statistically significant. Don't lower the threshold to manufacture a win.
- Baselines: `cargo bench -- --save-baseline main` on the base revision, then `cargo bench -- --baseline main` on the candidate. `--baseline` compares without overwriting. Pass a filter regex to run one group: `cargo bench -- parse_header`.
- `--profile-time <seconds>` iterates without analysis, which is useful for attaching a profiler to a benchmark.

## Rust: divan

```toml
[dev-dependencies]
divan = "0.1.21"

[[bench]]
name = "parse"
harness = false
```

```rust
// benches/parse.rs
use divan::{AllocProfiler, Bencher};

#[global_allocator]
static ALLOC: AllocProfiler = AllocProfiler::system();

fn main() {
    divan::main();
}

#[divan::bench(args = [64, 1024, 65_536])]
fn encode(bencher: Bencher, len: usize) {
    bencher
        .with_inputs(|| mycrate::fixtures::payload(len))
        .bench_values(|payload| mycrate::encode(divan::black_box(payload)));
}
```

- Input generation in `with_inputs` is not timed. `bench_values` passes inputs by value, `bench_refs` by mutable reference.
- `AllocProfiler` reports allocation counts and bytes per iteration next to the timings. Use it whenever the claim is "fewer allocations".
- CLI knobs: `--sample-count`, `--sample-size`, `--min-time`. Divan has no built-in baseline comparison, so for before/after evidence run both revisions on the same machine several times, or use criterion.

## Python: pyperf

pyperf spawns worker processes, calibrates loops, discards warm-ups and stores every value in JSON. Use it when the result will justify a change.

```python
# bench_normalize.py
import pyperf

from app.text import normalize
from app.fixtures import load_titles

TITLES = load_titles()

def run_all(titles=TITLES):
    for t in titles:
        normalize(t)

runner = pyperf.Runner()
runner.bench_func("normalize_titles", run_all)
```

```bash
python -m pyperf system tune          # Linux, needs root; `system reset` afterwards
python bench_normalize.py -o base.json          # on the base revision
python bench_normalize.py -o candidate.json     # on the candidate
python -m pyperf compare_to base.json candidate.json --table
python -m pyperf check candidate.json           # warns on unstable results
```

- Defaults are 20 worker processes with 3 values each (6 and 10 under a JIT) plus warm-ups. `--rigorous` doubles the processes and `--fast` is for rough answers only.
- `compare_to` runs a two-sample t-test and says "not significant" when it cannot tell the runs apart. Report that verdict. Don't report the raw percentages as a win.
- `check` warns when the standard deviation exceeds 10% of the mean, or when the shortest raw value (one timed batch of calibrated loops, not a single call) is under a millisecond.

## Python: pytest-benchmark

Good for keeping benchmarks next to tests and gating regressions.

```python
def test_normalize_titles(benchmark, titles):
    result = benchmark(normalize_all, titles)
    assert result == expected_normalized(titles)

def test_sort_rows(benchmark, rows):
    # setup runs outside the timer; with setup, iterations must be 1
    benchmark.pedantic(sort_rows, setup=lambda: ((list(rows),), {}), rounds=50, iterations=1)
```

```bash
pytest --benchmark-only --benchmark-autosave                  # base revision
pytest --benchmark-only --benchmark-compare --benchmark-compare-fail=median:5%
```

- `--benchmark-compare` with no argument compares against the latest saved run. `--benchmark-compare-fail` takes `stat:percent` (`min:5%`) or `stat:seconds` (`mean:0.001`).
- That check is a threshold on one statistic, not a significance test (5.3.0 computes `current / previous * 100 - 100`). A 5% gate on a benchmark whose runs differ by 8% fails at random, and a real 4% regression passes. Measure the A/A spread first and set the threshold above it.
- Saved runs from another machine are not a baseline. Autosave on the same runner in the same job.
- `--benchmark-disable-gc` hides GC cost. Leave GC on unless you are deliberately isolating it.
- `--benchmark-disable` runs each benchmark once with no timing, which is a cheap way to keep benchmark code from rotting in the normal test run.

## Python: timeit

Fine for a quick look, weak as evidence. `timeit` turns GC off during timing by default. For allocation-heavy code, pass `"gc.enable()"` as the first setup statement. Use `repeat()` and read the whole vector. The Python docs point to the minimum as the lower bound for pure CPU-bound snippets, but that says nothing about the tail.

## JavaScript: mitata

```js
import { bench, do_not_optimize, run, summary } from 'mitata';
import { parseQuery, parseQueryFast } from '../src/query.js';
import { loadQueries } from './fixtures.js';

const queries = loadQueries();

summary(() => {
  bench('parseQuery', function* () {
    yield {
      [0]() { return queries; },
      bench(qs) {
        for (const q of qs) do_not_optimize(parseQuery(q));
      },
    };
  });
  bench('parseQueryFast', function* () {
    yield {
      [0]() { return queries; },
      bench(qs) {
        for (const q of qs) do_not_optimize(parseQueryFast(q));
      },
    };
  });
});

await run();
```

- `do_not_optimize(value)` forces a side effect so the JIT cannot delete the work.
- Computed parameters (`[0]() { ... }` returning the input) stop the JIT from treating inputs as loop invariants or constants.
- `.gc('inner')` runs GC before every iteration to separate GC noise. It needs `node --expose-gc`.
- `.args(name, values)` and `.range(name, start, end)` parameterize input size, so scaling problems show up.

## JavaScript: tinybench

```js
import { Bench } from 'tinybench';
import { parseQuery, parseQueryFast } from '../src/query.js';
import { loadQueries } from './fixtures.js';

const queries = loadQueries();
const bench = new Bench();
let sink;

bench
  .add('parseQuery', () => { for (const q of queries) sink = parseQuery(q); })
  .add('parseQueryFast', () => { for (const q of queries) sink = parseQueryFast(q); });

await bench.run();
if (sink === undefined) throw new Error('benchmark produced no result');
console.table(bench.table());
```

Tinybench times whatever the task does, so unused work is at the JIT's mercy. Write results to a variable that escapes the task and is read afterwards, vary the inputs, and check the relative margin of error in the table before comparing rows.

JIT order effects are real. A function that has seen several object shapes during an earlier benchmark can be slower in a later one. When two variants share helpers, run each variant in its own process, or swap the order and confirm the ranking holds.

## Go

```go
func BenchmarkParseHeader(b *testing.B) {
	corpus := loadHeaderCorpus(b)
	b.ReportAllocs()
	for b.Loop() {
		for _, h := range corpus {
			parseHeader(h)
		}
	}
}
```

```bash
go test -c -o "$OUT/old.test" ./parser     # on the base revision
go test -c -o "$OUT/new.test" ./parser     # on the candidate
cd parser    # go test runs from the package directory, and testdata paths depend on it
for i in $(seq "$RUNS"); do
  "$OUT/old.test" -test.run='^$' -test.bench=ParseHeader >> "$OUT/old.txt"
  "$OUT/new.test" -test.run='^$' -test.bench=ParseHeader >> "$OUT/new.txt"
done
benchstat "$OUT/old.txt" "$OUT/new.txt"
```

- `b.Loop` keeps arguments and results inside the loop alive and resets the timer on its first call, so setup before the loop is excluded. It arrived in Go 1.24, but on 1.24 and 1.25 it also blocked inlining in the loop body, which added allocations and slowed some benchmarks. Since Go 1.26 it no longer does, so use 1.26 or later and don't compare `b.Loop` results across that boundary. With the older `for i := 0; i < b.N; i++` form, assign results to a package-level sink and call `b.ResetTimer()` after setup.
- benchstat wants at least 10 runs per side (`RUNS` above), and its docs say to pick a count and stick to it. `~` in its output means no significant difference.

## Command-line programs: hyperfine

```bash
hyperfine --warmup 3 --runs 30 --export-json results.json 'old-tool input.dat' 'new-tool input.dat'
hyperfine --prepare 'sync; echo 3 | sudo tee /proc/sys/vm/drop_caches' 'new-tool input.dat'   # cold page cache
hyperfine -N 'new-tool --version'   # no intermediate shell, for very short commands
```

hyperfine finishes every run of the first command before it starts the second, so the second inherits whatever heat or background load built up. For small differences, repeat the whole invocation with the commands in swapped order and check the ranking holds. `--parameter-scan` varies one parameter across a range, which is how to show scaling. Check exit codes: hyperfine fails on a non-zero exit unless `--ignore-failure` is set, and a benchmark that exits early is fast for the wrong reason.
