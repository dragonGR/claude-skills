---
name: ai-code-audit
description: Hostile audit of code an AI wrote or changed, including your own work, catching hallucinated packages, APIs and config, architectural drift, hidden failure paths, suppressed errors, placeholder logic, tests that cannot fail and unproven claims of done. Load it before saying a change is done, when reviewing an AI-written branch or pull request, and after any long agent session.
license: MIT
metadata:
  author: Alex Tsanis
---

# AI code audit

Treat every AI-written change as wrong until it survives checking. Not because the code looks bad: AI code usually looks good, compiles and passes the tests it brought with it. The failures hide in what looks plausible. A package name that sounds right, an option the library never had, a second logger next to the existing one, an error caught and logged so the flow "works", a test edited until it passes, a confident "done" with nothing behind it. This skill is the procedure for finding those before they ship.

It applies to your own output first. Before you tell the user a change is finished, audit it as if another model wrote it and you are paid to find its mistakes. It applies the same way to code from Codex, Gemini, Copilot or any other agent.

Two rules hold throughout:

- **Prove it against the source.** Every claim about an API, a package, a config key, a schema or a behavior is checked against the installed code, the lockfile, the migrations or the running program. Memory, the model's confidence and "it compiles" are not evidence.
- **The audit can hallucinate too.** A finding needs a file, a line and a reason it is reachable, and it must survive an honest attempt to disprove it. Report what you could not confirm as unconfirmed. A false alarm costs trust the same way a missed bug does.

## Procedure

Run every step. Skipping a step because the change "looks simple" is how drift and fake fixes get through.

1. **Pin the contract.** Write down what was asked, in the user's words, and what would prove it done. The audit measures the change against that, not against what the AI decided to build.
2. **Collect the whole change.** Diff against the merge base, plus untracked files, generated files and the lockfile. AI agents often leave the important part in a file nobody looks at. Commands: [references/verification-commands.md](references/verification-commands.md).
3. **List the claims.** Everything the AI said it did: "added tests", "handled errors", "verified on Amoy", "no breaking changes", "fixed the race". Each one is a claim to prove or reject in step 10.
4. **Hallucination pass.** For every new import, package, function call, option, config key, CLI flag, environment variable, route, database column, contract function and URL: find it in the installed source, lockfile, schema, migrations or docs for the pinned version. Anything you cannot find is a finding.
5. **Drift pass.** For each concern the change touches (HTTP calls, data access, validation, errors, logging, config, auth, state, styling), find how the repository already does it and compare. A second way of doing something the codebase already does is a finding even when both work. Procedure: [references/architecture-drift.md](references/architecture-drift.md).
6. **Failure-path pass.** For each new call that can fail (network, disk, database, chain, parse, external process), read what happens on error, timeout, partial success, retry and restart. Check that resources are released on the error path, not only on success.
7. **Suppression and placeholder pass.** Search the diff for silenced diagnostics, stubs, hardcoded success, commented-out checks and loosened configuration. Patterns per language are in the commands reference.
8. **Test pass.** Read test changes before code changes. Deleted, skipped or weakened tests, edited expected values and mocks that return the asserted value are the most common way AI work fakes a green suite. Then check that each new test attacks the change and fails without it: [references/attack-tests.md](references/attack-tests.md).
9. **Execute.** Run the real build, type check, lint and test suite on the change, and on the base when results need comparing. Read the output. A run you did not see is not evidence.
10. **Re-attack.** For every claim from step 3 and every fix in the change, try to break it: the sibling endpoint, the second caller, the empty input, the concurrent request, the reverted fix. Then try to disprove each of your own findings before reporting it.
11. **Report** in the format below.

## Failure catalogue

### Hallucinations

**Invented or wrong package.** An import of a package that does not exist, a misspelled name that resolves to someone else's package, or a real package at a version that lacks the API used. Attackers register names that models commonly invent. Check the lockfile, the registry entry (creation date, publisher, repository, download history) and that the version installed exports what the code imports.

**Invented API.** A method, option or overload that sounds right: `fs.promises.exists()`, `prisma.$upsertMany()`, `useQuery({ onSuccess })` on a TanStack Query version that removed it. Open the type definitions or source of the installed version and find the symbol. Type checking catches some of this; it misses `any`-typed clients, dynamic languages, string-keyed options and anything behind a cast.

**Stale API from training data.** The API existed, but the installed major version renamed or removed it, or changed its default. Read the changelog between the version the code assumes and the version in the lockfile.

**Invented configuration.** A config key, CLI flag or environment variable that the tool never reads. Many tools ignore unknown keys silently, so the setting looks applied and does nothing. Confirm each key in the tool's docs or schema for the pinned version, and confirm each environment variable is actually set in every environment (`.env.example`, deployment manifests, CI, secret store).

**Invented schema and endpoints.** A query against a column no migration creates, a frontend call to a route nothing serves, a contract call to a function the ABI does not have, an event name nobody emits. Check migrations, route tables, ABIs and generated clients.

**Invented facts in prose.** Comments, docs and commit messages stating limits, defaults, versions or guarantees ("retries are idempotent", "the limit is 100 per second") that nothing in the code or docs supports. Wrong documentation misleads every later reader and the next AI that reads it.

### Architectural drift

**A second way of doing an existing thing.** A new HTTP client next to the shared one, a new logger, a new validation library, a new error class hierarchy, a new date library, a new state store, a fetch wrapper that duplicates the existing one. Each is small; together they split the codebase into dialects nobody owns.

**Bypassing a layer.** A route handler that queries the database directly when every other handler goes through a repository or service; a component calling the API directly instead of the data hooks; a contract script that skips the shared deployment config.

**Moving authority.** A check that belonged on the server, in the database or in the contract now happens in the client, the UI or a script, or a value the server should derive is now taken from the request. This is drift and a security bug at once.

**A second source of truth.** A constant, enum, ABI, schema or address copied into a new file instead of imported; generated files edited by hand; configuration read from a new place.

**Duplicated helpers.** A new `formatCurrency`, `retry`, `sleep`, `chunk` or `isValidAddress` when one already exists. Search for the name and for the behavior before accepting a new helper.

**Unrequested structure.** New abstraction layers, factories, generic wrappers, plugin systems or configuration options with one caller and no request for them.

### Failure handling

**Errors swallowed so the flow works.** `catch` that logs and continues, returns `null`, an empty list or a default; `except: pass`; `.catch(() => {})`; `let _ = result`; `try?`. The happy path passes the demo and the failure becomes silent data loss.

**Fallbacks that hide misconfiguration.** `process.env.X ?? "default"` for secrets, URLs, chain ids or limits; a missing config producing a working-looking system pointed at the wrong place.

**Success reported on partial failure.** A batch that returns 200 when some items failed, a loop that `continue`s past errors, a job marked done when one step threw.

**Missing deadlines and cleanup.** Network calls without timeouts, retries without limits or idempotency, files, connections, listeners and subprocesses left open on the error path.

### Faking progress

**Placeholder logic presented as done.** `// TODO: implement`, `return true`, `return []`, hardcoded sample data, `pass`, `unimplemented!()`, a mock implementation left in production code, a feature flag that keeps the new path off, an early return "for now", a check commented out "temporarily".

**Silenced diagnostics.** `as any`, `as unknown as T`, `@ts-ignore`, `@ts-expect-error` without a reason, `eslint-disable`, `# type: ignore`, `# noqa`, `#[allow(...)]`, `.unwrap()` on input, `// nolint`, lint or compiler config loosened, CI steps removed or marked `continue-on-error`, coverage thresholds lowered.

**Tests made to pass.** Tests deleted or skipped, assertions removed or weakened, expected values edited to match new output, snapshots regenerated without review, mocks that return exactly what the assertion checks, auth or validation disabled in the test harness. A suite that went green this way proves the opposite of what it claims.

### Scope and claims

**Unrequested changes.** Renamed public functions, reformatted files, dependency bumps, refactors and "cleanups" that nobody asked for. They hide the real change in noise and break callers outside the diff.

**Claims without evidence.** "Tested", "verified", "fixed", "no breaking changes", "works on mainnet" with no command output, test or measurement behind them. "Should work" is a guess.

**Over-production.** Comments that restate the code, changelog comments in source, new README or docs files nobody requested, unused exports, dead branches. Each one is something a person has to read and maintain.

## Attack tests

Every behavior change needs at least one test that attacks it, and a test only counts if it was seen to fail without the change. The minimum for each kind of change:

| Change | The test must show |
| --- | --- |
| Bug fix | the original failure reproduced: fails on the base, passes with the fix |
| Access or ownership check | another user, another tenant, a lower role and an anonymous caller are all refused, and the protected state is unchanged |
| Validation | malformed, oversized, negative, zero, unknown-field and boundary inputs are rejected for the right reason |
| Money, counters, limits | two concurrent requests cannot both succeed past the limit; a replayed request does not apply twice |
| External call | timeout, error response and malformed response leave state correct and are reported, not swallowed |
| Contract function | wrong caller, replayed signature, reentrant callback and extreme amounts revert, and balances are unchanged |

Prove the "fails without it" part by running the new test against the code with the fix reverted, in a separate worktree so nothing in the working tree is touched. Procedure and examples: [references/attack-tests.md](references/attack-tests.md). Test design in depth (authorization matrices, concurrency, idempotency, mutation testing) belongs to the test-engineering skill.

## Report

Lead with the verdict, then the evidence.

- **Verdict:** blocked (a confirmed defect, hallucination, fake test or unproven core claim), changes needed, or acceptable.
- **Findings,** most severe first. Each with file and line, what is wrong, how it was confirmed (the command, the missing symbol, the failing test), the impact, and the fix. Mark each confirmed or unconfirmed, and say what would settle an unconfirmed one.
- **Claims:** each claim from step 3 with the evidence found, or "no evidence".
- **What was run:** commands and results, including anything that could not be run and why.
- **Dropped:** suspicions you investigated and disproved, in one line each, so nobody chases them again.

Do not pad the report with style notes or praise. If the change is good, say so in one line and list what you checked.

## Checklist

- Does every new import resolve to a real, expected package at a version that has the API used?
- Does every called method, option, config key, CLI flag and environment variable exist for the pinned version, and is each environment variable set everywhere it is needed?
- Does every column, route, ABI function and event the change uses exist?
- Does the change use the repository's existing client, logger, validation, errors, config, data access and state patterns, with no duplicated helper and no new source of truth?
- Did any authority move to the client or the request?
- Does every failure path either handle the error correctly or surface it, with deadlines and cleanup?
- Is there no placeholder logic, hardcoded success, commented-out check or fallback for missing configuration?
- Is there no new suppression, loosened config, removed CI step or lowered threshold?
- Were no tests deleted, skipped or weakened, and no expected values edited to match new output?
- Does each new test attack the change, and was it seen to fail without the change?
- Did the real build, type check, lint and tests run, with the output read?
- Is every file in the diff explained by the task?
- Is every claim of done, tested or fixed backed by evidence in the report?
- Did every finding survive an attempt to disprove it?

## References

- [references/verification-commands.md](references/verification-commands.md): read when collecting the change and checking packages, APIs, config, schema and suppressions; commands and search patterns for TypeScript and Node, Python, Rust and Solidity.
- [references/architecture-drift.md](references/architecture-drift.md): read for the drift pass; how to map the repository's conventions quickly and judge a deviation.
- [references/attack-tests.md](references/attack-tests.md): read when checking or writing tests for an AI change; spotting faked test results, proving a test fails without the fix, and attack test examples.

Related skills: test-engineering for test design in depth, security-engineering for security findings, and the language skills for language-specific failure modes.
