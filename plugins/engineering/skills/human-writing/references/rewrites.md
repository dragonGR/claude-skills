# Worked rewrites

Read this when you need to see a full rewrite rather than a single pattern: a design-doc paragraph, a PR description, release notes, an answer to a user, and a draft that over-corrected into fake casualness. Each shows the draft, the rewrite and what changed. Numbers and names are invented for the example; in real work they come from the source and stay exactly as written.

## Design-doc paragraph

Draft:

> In today's rapidly evolving microservices landscape, caching plays a pivotal role in ensuring optimal performance. Our new caching layer serves as a robust foundation for the platform — not only reducing latency, but also significantly lowering infrastructure costs. Moreover, it seamlessly integrates with our existing services, empowering teams to focus on what matters most. It's important to note that cache invalidation remains a complex challenge. In conclusion, this solution represents a major step forward.

Source facts the author gave: the cache sits in front of the pricing service; median latency went from 180 ms to 35 ms in the load test; Redis cost is about $400 a month; invalidation is by TTL of 60 seconds, so prices can be up to a minute stale; checkout bypasses the cache; nobody has measured the infrastructure savings.

Rewrite:

> The pricing service now reads through a Redis cache. In the load test, median latency dropped from 180 ms to 35 ms. Entries expire after 60 seconds and nothing invalidates them earlier, so a price change can take up to a minute to show. That is acceptable for catalog pages but not for checkout, which still reads from the service directly. The cache costs about $400 a month; we have not measured whether it saves anything elsewhere.

What changed. The opener about the landscape and the closing "step forward" went, because neither carried a fact. "Robust foundation", "seamlessly integrates" and "empowering teams" were replaced by the facts they were standing in for, or dropped when no fact existed. The claim about lower costs was removed because it was never measured, and the rewrite says so, which is more useful than the claim. "Cache invalidation remains a complex challenge" became the actual limitation and who it affects. The em dash and the "not only…but also" went with the sentence they were in. The rewrite is about as long as the draft, and every sentence in it carries a fact the draft left out.

## Pull request description

Draft:

> ## Summary
> This PR introduces a comprehensive overhaul of the webhook handling logic, ensuring greater reliability and security.
>
> ## Changes
> - **Idempotency:** Added deduplication
> - **Reliability:** Improved retry handling
> - **Performance:** Optimized database queries
>
> ## Impact
> These changes significantly improve the robustness of our webhook pipeline, paving the way for future scalability.

Rewrite:

> Stripe redelivers an event when our endpoint times out or returns an error, and the handler processed every delivery. When the ledger write was slow, a top-up could be credited twice; support found three cases in September. The handler now inserts `event.id` into `webhook_events`, which has a unique index, in the same transaction as the ledger write, and acknowledges a delivery whose ID is already there without processing it again.
>
> Fixes #418.
>
> Verified: `npm test`, including new tests for a redelivered event and for two concurrent deliveries of the same event. On staging, resending 20 already-processed events from the Stripe Dashboard produced no new ledger rows.
>
> Risk: the migration adds a unique index on `webhook_events(event_id)`. The table has 40k rows; the index is built with `CREATE UNIQUE INDEX CONCURRENTLY` in its own migration, because PostgreSQL does not allow that statement inside a transaction. A query on production found no duplicate `event_id`s in the existing rows. If the build fails anyway, it leaves an invalid index that has to be dropped before the migration is retried.

What changed. The template headings and bold-label bullets split one bug into three "changes". The rewrite says what was wrong, how it showed up, what the fix does and why it prevents duplicates. "Optimized database queries" had nothing in the diff behind it and was dropped. The verification line lists what was actually run, and the risk line gives the reviewer the one thing to check in the migration.

## Release notes

Draft:

> # 🎉 Version 2.4 Is Here!
> We're thrilled to announce pgshift 2.4, packed with exciting new features and under-the-hood improvements! This release brings enhanced performance, a streamlined configuration experience, and various bug fixes.

Rewrite:

> ## 2.4.0 (2026-09-28)
>
> Breaking: `--config` no longer reads YAML. Convert an existing file with `pgshift config convert pgshift.yaml > pgshift.toml`.
>
> Added
> - `pgshift plan` prints the locks each pending migration will take, without applying it (#212).
>
> Fixed
> - `pgshift status` exited 0 when the database was unreachable; it now exits 2 (#231).
> - Migrations with a `BEGIN` inside a comment were split at the wrong line (#240).

What changed. The version heading uses the version number and an ISO date. The breaking change moved to the top with the exact command to migrate, because that is what decides whether a reader can upgrade today. "Enhanced performance" had no change behind it in the changelog and was dropped. Each entry says what the user will see differently and links the issue.

## Answer to a user

Draft:

> Great question! You're absolutely right to be concerned about this. Connection pooling is a crucial aspect of database performance. There are several factors to consider:
>
> - **Pool size:** It's important to choose the right pool size.
> - **Timeouts:** Timeouts should be configured appropriately.
>
> I hope this helps! Let me know if you have any other questions.

The user had asked why their app gets "too many connections" errors after scaling from 2 to 12 pods, with a pool of 20 per pod and PostgreSQL `max_connections` at 100.

Rewrite:

> 12 pods with 20 connections each can open 240 connections, and the server allows 100 (a few of which are reserved for superusers). With 2 pods you peaked at 40, so the limit never showed. Either lower the per-pod pool so the total stays under about 90 (7 per pod gives 84), or put PgBouncer in transaction mode between the pods and the database and size its server-side pool under the limit. Raising `max_connections` works too, but each PostgreSQL connection is a separate backend process with its own memory, so it trades this error for memory pressure.

What changed. The praise and the offer to help went. The bullet list said only that pool size and timeouts matter; the rewrite does the arithmetic with the user's own numbers, gives two fixes with the rule for choosing, and names the cost of the third. Nothing in the answer is generic: every sentence uses their pods, their pool and their limit.

## Over-corrected draft

Told to make the first draft above "sound human", a model often produces this:

> Here's the deal: caching. Honestly? It's kind of a big win. Latency? Way down. Costs? Probably down too. The catch: stale prices. But hey, nothing's perfect.

It has no em dashes and no "robust", and it is still unmistakably generated: self-answered questions, fragments for rhythm, forced casualness, and the unmeasured cost claim survives as "probably". The plain rewrite in the first section is the target. Plain means the register the reader expects and sentences as long as the thought.
