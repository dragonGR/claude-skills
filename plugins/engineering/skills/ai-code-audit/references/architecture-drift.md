# Architectural drift

Read this for the drift pass of an audit, or before writing code in a repository you have not worked in yet.

Drift is code that works but does not belong: a second way of doing something the codebase already does, a layer skipped, a decision moved to the wrong place, a fact copied instead of referenced. AI agents produce it constantly because they write from their training data, not from the repository, and because each change looks reasonable on its own. No single instance breaks anything. Together they leave a codebase with three HTTP clients, two error formats and authority checks in places nobody remembers, and every later change gets harder and riskier.

## Build the convention map first

Before judging the change, spend a few minutes finding how the repository already handles each concern the change touches. Read existing code, not documentation alone; documentation drifts too.

| Concern | How to find the existing way |
| --- | --- |
| Outbound HTTP | search for the existing client wrapper or `fetch`/`axios`/`httpx`/`reqwest` usage and where timeouts and retries are configured |
| Data access | find how three existing handlers read and write: repository, service, ORM, raw SQL, D1 batch |
| Validation | find the schema library and where parsing happens (route edge, service, both) |
| Errors | find the error types, how they map to responses, and the response shape clients rely on |
| Logging and metrics | find the logger import, the field names, the correlation id |
| Configuration | find where config is loaded and validated, and how code reads it |
| Auth and authorization | find where the principal comes from and where permission checks live |
| State on the frontend | find the server-state library, the store, and how mutations invalidate data |
| Styling and components | find the design tokens, the component library and the existing primitives |
| Contracts and chain access | find the shared deployment config, ABI source, client setup and how finality is handled |
| Tests | find the test utilities, fixtures, factories and how the database and network are handled |

Write the map down in the audit notes. It turns "this feels different" into "the repository does X in these 14 places; the change does Y".

## What counts as drift

**Parallel mechanism.** The change introduces its own way of doing a mapped concern: a new client, logger, validator, error type, config reader, date library, store, or component that duplicates an existing primitive. Ask whether the existing mechanism could have done the job. If yes, it is a finding.

**Skipped layer.** The change reaches past the layer every other caller goes through: a handler using the database client directly, a component calling `fetch` instead of the data hooks, a script reading contract addresses from a literal instead of the deployment record. Layers usually exist because they carry a rule (authorization, tenancy, caching, retries); skipping them drops the rule silently.

**Authority moved.** A decision the server, database or contract owned is now made by the client, the frontend or an off-chain script, or a value the server derived is now read from the request. Treat this as a security finding as well as drift.

**Second source of truth.** A constant, enum, ABI, schema, address or type copied into the change instead of imported from where it is defined; a generated file edited by hand instead of regenerated. The copies drift apart on the next change and nobody notices until production disagrees with itself.

**Different contract on the wire.** A new endpoint whose errors, pagination, field naming, dates or money representation differ from the existing ones. Clients now need two parsers.

**Local style in a shared codebase.** Naming, file layout, module boundaries, async style or test structure that matches the model's habits instead of the files next to it. Lower severity than the others, but it compounds.

**Unrequested structure.** Interfaces with one implementation, factories with one product, configuration for things nobody asked to configure, generic helpers with one caller. They look like good engineering and add surface without a reason.

**New dependency for a solved problem.** A package added for something the standard library or an existing dependency already does.

## When a deviation is acceptable

A change may deviate from the convention when the deviation is deliberate and complete:

- the existing way is broken or unsafe, the change says so, and it migrates the other callers or records the decision with a plan to do so;
- the task explicitly asked for the new approach;
- the existing mechanism genuinely cannot do the job, and the change explains why.

A deviation that is undocumented, partial or accidental is drift. "The new way is better" is not enough when the result is two ways.

## Reporting drift

For each finding: the concern, the existing convention with two or three file references, what the change does instead, the concrete cost (a rule skipped, a second parser, a copy that will go stale), and the fix, which is usually to use the existing mechanism. Drift that moves authority or skips a security-bearing layer is reported as a defect, not a style note.
