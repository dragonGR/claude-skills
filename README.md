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
| `security-engineering` | Threat mapping, hostile-caller review, authorization, sessions, injection, SSRF, webhooks, secrets and CI, Kubernetes, exploit triage |
| `solidity-engineering` | Smart contracts on Polygon PoS with Foundry: access control, accounting, reentrancy, signatures and Keccak hashing, upgrades, oracles, unbounded loops, fuzz and invariant tests, deployment and verification |
| `backend-architecture` | State ownership, idempotency, unknown outcomes, messaging, consistency, API contracts, recovery |
| `database-engineering` | Constraints, concurrency control, migrations, indexing, PostgreSQL, SQLite and D1 differences |
| `infrastructure-ops` | Containers, infrastructure as code, CI/CD, deploy and rollback, observability, backups |
| `git-workflows` | Safe Git for agents and people: recovery with reflog, rebase and fixups, cherry-pick, reverting merges, conflicts and rerere, bisect, worktrees, force pushes with lease, leaked secrets, hygiene |
| `nodejs-engineering` | The Node runtime in production: crashes and shutdown, server timeouts, memory, the thread pool, debugging and diagnostics, upgrading Node and dependencies |
| `typescript-engineering` | Type design, runtime validation, errors, async lifetimes, Node security |
| `python-engineering` | Typing, boundary validation, errors, asyncio, resources, packaging |
| `rust-engineering` | Type-driven design, error types, Tokio, unsafe and FFI, tooling |
| `frontend-engineering` | React, Vite and React Router: state ownership, server state with TanStack Query, async UI states, env and bundle exposure, client trust limits, wallet frontends |
| `react-motion` | Motion (framer-motion) and GSAP in React: exits, layout animation, route transitions with React Router, reduced motion, frame performance |
| `accessibility` | WCAG 2.2 AA implementation and audits |
| `ai-code-audit` | Hostile audit of AI-written changes: hallucinated packages, APIs and config, architectural drift, swallowed errors, placeholders, faked tests, attack tests that must fail without the fix, unproven claims |
| `test-engineering` | Risk-based test strategy, adversarial and concurrency tests, property testing, flaky tests |
| `performance-benchmarking` | Baselines, profiling, noise control, promotion gates |
| `human-writing` | Drafting and editing prose that reads like a person wrote it |
| `seo-engineering` | Crawlability, rendering and prerendering for React and Vite SPAs, canonicals, real 404s, hreflang, sitemaps, structured data, Core Web Vitals |

## License

MIT. Copyright Alex Tsanis.
