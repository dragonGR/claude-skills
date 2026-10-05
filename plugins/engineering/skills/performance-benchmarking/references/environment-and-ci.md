# Benchmark environment, noise and CI gates

Read this when setting up a machine for benchmarks, deciding whether a difference is real, or adding a performance gate to CI.

## Measure the noise before the effect

Run the baseline against itself (an A/A comparison) with the same harness, the same run count and the same machine you will use for A/B. Whatever difference that shows is the noise floor. A claimed improvement smaller than that is no improvement. Repeat the A/A check whenever the machine, kernel, runtime version or harness settings change.

## Local machine setup (Linux)

- Plug laptops into power. The CPU slows down on battery.
- Close browsers, IDE indexers, sync clients, containers and anything else that wakes up periodically.
- Fix the frequency. `python -m pyperf system tune` (root) sets the `performance` governor, raises the minimum frequency, turns off Turbo Boost through `intel_pstate` or the MSR, and moves IRQs off the benchmark CPUs. `python -m pyperf system show` reports the current state and `python -m pyperf system reset` undoes it. It's useful for non-Python benchmarks too.
- pyperf's turbo handling is Intel-only (both the `intel_pstate` file and the MSR are Intel interfaces). On AMD, boost stays on after `system tune`. Write `0` to `/sys/devices/system/cpu/cpufreq/boost` under acpi-cpufreq, or to each `/sys/devices/system/cpu/cpuN/cpufreq/boost` under amd-pstate, or turn off Core Performance Boost in firmware. Check with `cpupower frequency-info` that boost reads as inactive.
- Turbo and thermal limits are the main laptop problem. Frequency rises for the first seconds, then falls as the chip heats, so whichever variant runs first wins. With turbo off and variants alternated, the order effect goes away.
- For low-noise work, isolate cores on the kernel command line (`isolcpus=`, `nohz_full=`, `rcu_nocbs=` for the same CPU list) and pin the benchmark to them (`taskset -c`, or pyperf `--affinity`). pyperf's docs warn against combining `nohz_full` with the `intel_pstate` driver.
- Leave ASLR on. pyperf's docs are explicit that it must not be disabled by hand. Layout variation is part of real performance, and multiple processes average it out.
- Record the environment with every result: CPU model, core count, kernel, governor and turbo state, runtime version, compiler flags, build profile.

## Cloud and shared hosts

- Burstable instance types (AWS T-family and similar) depend on a CPU credit balance. In standard mode they fall back to a baseline when credits run out. T3, T3a and T4g launch in unlimited mode by default, which keeps bursting and bills the surplus. Either way the result depends on credit state and the account's settings. Never benchmark on them.
- Virtual machines share caches, memory bandwidth and network with neighbors you can't see. For small effects, use dedicated or bare-metal hosts. Otherwise run A and B interleaved on the same instance in the same session, so the neighbors affect both equally.
- Containers with CPU limits are throttled per quota period. Read `nr_throttled` and `throttled_usec` in the cgroup's `cpu.stat` before and after a run. If they moved, the result includes throttling.
- Don't compare results across instance types, regions, runtime versions or days. The comparison is only valid inside one session.

## Statistics that hold up

- Several runs per variant, alternating (ABABAB, or randomized order), not all of A then all of B. benchstat asks for at least 10 runs per side. pyperf uses 20 processes by default. Criterion takes 100 samples.
- Use a statistical test, not eyeballed percentages: criterion's change report (1% noise threshold, 0.05 significance), `benchstat` (`~` means no significant difference), `pyperf compare_to` ("not significant"). pytest-benchmark runs no test: `--benchmark-compare-fail` is a percentage threshold on one statistic.
- Look at the distribution. Two humps usually mean two code paths (cache hit and miss, JIT tiers, GC and no GC). Report them separately rather than averaging them into a number that matches neither.
- Many comparisons produce false alarms. At a 0.05 significance level, a suite of 200 unchanged benchmarks flags about 10 of them. Before acting on one flagged benchmark in a large suite, rerun it and look for a plausible mechanism.
- Don't rerun until you get the answer you want. Decide the run count before looking at results.
- For load tests, one run per step at steady state is common. Still, repeat the critical step at least once, and treat a p99 that moves between repeats by more than the claimed gain as noise.

## CI performance gates

Shared CI runners are noisy. Gate on something that survives that noise:

1. Relative comparison in one job. Check out base and head, build both, and run them on the same runner in alternating order. Fail on a statistically significant regression larger than the measured A/A noise for that suite. Never compare against numbers stored from a different runner.
2. Instruction counts for CPU-bound library code. Counting instructions under Valgrind's cachegrind is nearly deterministic across runs on the same binary and toolchain. It doesn't see cache-miss or I/O effects, so pair it with occasional timing runs on quiet hardware.
3. Dedicated hardware for trends. A self-hosted, tuned runner that benchmarks main on every merge catches the slow creep that per-PR gates miss: twenty PRs that each cost 0.5% never trip a 5% gate.

Rules for the gate itself:

- Thresholds come from the measured noise of each suite and live in config next to the suite, not scattered through the pipeline as literals.
- A failed gate is investigated, not retried until green. If the benchmark is too noisy to gate on, fix the benchmark or move it to the trend job.
- Gate on the metric users feel (p99 of an end-to-end scenario, allocations per request, binary size, cold start), not only on the easiest microbenchmark.
- Keep raw results as build artifacts so a regression can be bisected later.

## Result ledger

Keep one per investigation, next to the change or in the PR description:

```text
variant       hypothesis                        p50     p99     runs  correct  notes
baseline      main @ <sha>                      38 ms   120 ms  10    yes      A/A noise ±3% p99
batch-reads   N+1 on line items                 14 ms    42 ms  10    yes      chosen
parallel-16   concurrency hides round trips     11 ms   310 ms  10    yes      pool waits; 429s from provider
fast-parse    skip schema validation             9 ms    40 ms  10    no       rejected: removes input validation
```

Include the command, commit, machine and data set under the table, so someone else can rerun it.
