# claude-skills

Engineering skills for [Claude Code](https://code.claude.com), written by Alex Tsanis for production work where security, data integrity and honest verification matter more than speed of output.

## Install

```text
/plugin marketplace add dragonGR/claude-skills
/plugin install engineering@dragonGR
```

To receive updates automatically, open `/plugin`, go to **Marketplaces**, select `dragonGR` and enable auto-update. Otherwise run `claude plugin marketplace update dragonGR`.

## What is in it

The `engineering` plugin contains these skills. Claude loads one when the task matches its description.

| Skill | Covers |
| --- | --- |
| `security-engineering` | Threat mapping, hostile-caller review, authorization, sessions, JWT and OAuth, injection, SSRF, archives, webhooks, secrets and CI, Kubernetes, exploit triage |
| `solidity-engineering` | Solidity with Foundry on Polygon PoS and other EVM chains: access control, accounting, reentrancy, EIP-7702, signatures and Keccak hashing, upgrades, oracles, compiler bugs, fuzz and invariant tests, deployment and verification |
| `backend-architecture` | Idempotency keys and unknown outcomes, timeouts and retries, outbox and consumers, job leases, state machines, caching, webhooks, API changes, the graceful-shutdown sequence |
| `database-engineering` | Constraints, locking and isolation, migrations and backfills, indexing, pools, row-level security, PostgreSQL, SQLite and Cloudflare D1 |
| `infrastructure-ops` | Dockerfiles, Kubernetes, GitHub Actions and OIDC, Terraform and OpenTofu, deploys and rollback, alerting, backups |
| `git-workflows` | Safe Git for agents and people: recovery with reflog, rebase and fixups, cherry-pick, reverting merges, conflicts and rerere, bisect, worktrees, force pushes with lease, leaked secrets, signing, hygiene |
| `nodejs-engineering` | The Node process in production: crash policy, SIGTERM shutdown code, keep-alive and timeouts, heap limits in containers, the thread pool, debugging and diagnostics, Node version upgrades |
| `typescript-engineering` | TypeScript and JavaScript on any runtime: runtime validation, async lifetimes, money and dates, TypeScript 6 and 7, ESM and CJS, npm, pnpm and Bun supply chain and upgrades, Workers and Bun |
| `python-engineering` | Python 3.11 to 3.14: asyncio, exceptions, HTTP clients, Decimal money, datetimes, Pydantic, SQLAlchemy 2, untrusted input, packaging with uv, pytest |
| `rust-engineering` | Panics on untrusted input, overflow and casts, Tokio cancellation and shutdown, unsafe and FFI, serde, SQL, errors, Cargo features and `cargo deny` |
| `frontend-engineering` | React 19, Vite 8 and React Router 8: server state with TanStack Query, effects, forms and Actions, env and bundle exposure, client trust limits, wallet frontends |
| `react-motion` | Motion, GSAP and view transitions in React: exits, layout animation, route transitions, native dialogs, reduced motion and pause controls, frame cost |
| `accessibility` | WCAG 2.2 AA implementation and audits: semantics, ARIA, keyboard and focus, dialogs, comboboxes, live regions, forms, contrast, target size |
| `ai-code-audit` | Hostile audit of AI-written changes, committed or not: hallucinated packages, APIs and config, architectural drift, swallowed errors, placeholders, faked tests, attack tests that must fail without the fix, unproven claims |
| `test-engineering` | Risk-based test strategy, real dependencies with Testcontainers and MSW, concurrency, idempotency and authorization tests, property-based and mutation testing, determinism, Playwright |
| `performance-benchmarking` | Baselines, profiling, microbenchmarks, load tests, browser measurement (Core Web Vitals, INP, long animation frames), noise control and CI gates |
| `human-writing` | Drafting and editing prose in English or Greek so it reads like a careful engineer wrote it |
| `seo-engineering` | Crawlability, rendering and prerendering for React and Vite SPAs, canonicals, real 404s, hreflang, sitemaps, structured data, Core Web Vitals, AI crawlers |

Version-specific advice was checked against releases current on 2026-10-05.

## License

MIT. Copyright Alex Tsanis.
