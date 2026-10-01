# Genre guides

Read the section for the genre you are writing or rewriting: READMEs, commit messages, PR descriptions, incident reports, API docs, client emails, release notes, security advisories or UI microcopy. Each gives what the reader needs, the order to put it in, the tells that are typical for that genre, and a short before and after.

The examples use invented projects and numbers. In real work every number, version and name comes from the source or from the user.

## README

The reader wants to know within one paragraph whether this solves their problem, then wants it running. Order:

1. Name and one sentence: what it does, for whom, on what. "pgshift runs PostgreSQL schema migrations from plain SQL files and aborts a migration that cannot acquire its lock within a set timeout."
2. Status if it is not stable ("Pre-1.0: the config format may change between minor versions").
3. Install: the exact command, and the supported runtime or platform versions.
4. The smallest example that works, with its real output.
5. Configuration: a table of name, default and meaning. The meaning adds something the name does not.
6. What it does not do, and known limits. Readers trust a README that says "no MySQL support" more than one that claims everything.
7. Contributing, license.

Tells typical of generated READMEs: "Welcome to X!", "X is a powerful, lightweight, and blazing-fast…", a "🚀 Features" list with a bold label on every bullet, a "Why X?" section of praise, "Whether you're a hobbyist or an enterprise…", and a closing "Happy coding!".

Before:

> ## 🚀 Why Choose Tidy?
> Tidy is a **powerful** and **intuitive** CLI that seamlessly streamlines your log management workflow, empowering teams to unlock actionable insights.

After:

> Tidy reads JSON log lines from stdin, drops fields you list in `tidy.toml`, and writes the rest to stdout. It is meant for stripping tokens and emails from logs before they leave the host.

## Commit messages

Git's own documentation suggests a first line of no more than 50 characters, then a blank line, then the body. Git treats the text up to the first blank line as the title and uses it, for example, as the Subject line of `git format-patch` emails. Many projects accept longer subjects, so follow the repository's history (`git log --oneline -20`) over any general rule.

- Subject: imperative mood, what changed in behavior. "Reject expired invites at accept time", not "Updated invite logic" or "Fixes".
- Body, when the why is not obvious from the diff: the problem, why this fix, what it does not cover, and how it was verified if that is not the test in the diff. Wrap at the width the repository uses.
- Conventional Commits only if the repository already uses it. That spec defines only `feat` and `fix`; other types such as `docs` or `refactor` are allowed but carry no meaning in the spec. A breaking change is marked with `!` after the type or scope, or a `BREAKING CHANGE:` footer.
- No trailers or attribution lines unless the repository or the user asks for them.

Tells: "Refactor code for improved readability and maintainability", "This commit introduces…", a body that lists every file touched, and a last line like "Overall, these changes enhance the user experience."

Before:

```
Enhance invite handling

This commit introduces several improvements to the invite flow,
ensuring a more robust and secure experience. Additionally, it
refactors the validation logic for better maintainability.
```

After:

```
Reject expired invites at accept time

Invites were only checked for expiry when the email was sent, so a
link opened after its 7-day window still created a membership. The
accept handler now compares expires_at with the database clock and
returns 410 for expired invites.
```

## Pull request descriptions

The reviewer needs to know what changed, why, how it was checked and what could go wrong, in about that order.

- What and why in two or three sentences. Link the issue. GitHub closes the issue on merge when the description uses a closing keyword such as `Fixes #123`, `Closes #123` or `Resolves #123`, and only when the pull request targets the default branch.
- How it was verified: the commands you ran and what they showed. Never tick a test-plan box for something you did not run; write "not run" and why.
- Risk and rollout: migrations, flags, backward compatibility, what to watch after deploy, how to roll back.
- Where to look: the file or function that carries the real change, when the diff is large.
- Screenshots or a recording for UI changes, at the widths that changed.

Tells: a "Summary / Changes / Impact / Conclusion" template with bold-label bullets under each, "This PR introduces a comprehensive refactor that significantly improves…", an impact section that praises the change, and a checklist ticked in full.

## Incident reports and postmortems

Readers are engineers, support, leadership and sometimes customers. They need the facts in a form they can quote.

1. Summary: what broke, who was affected, how many, from when to when in UTC, and the current state. Say "mitigated" when the impact stopped but the cause is still present, "resolved" only when the cause is fixed.
2. Impact in numbers: failed requests, affected accounts, duration, money, data. "No data was lost" only if you checked, and say how you checked.
3. Timeline in UTC with timestamps: start of impact, detection, escalation, each mitigation attempt, recovery.
4. Cause: the change or condition that triggered it, the factors that made it possible, the ones that made it worse, and what delayed detection. Write "root cause" only when it is established; otherwise "leading hypothesis" and the evidence for it.
5. What is still unknown.
6. Action items, each with an owner and a ticket, each tied to a factor above. "Continue to prioritize reliability" is not an action item.

Blameless means naming the systems, decisions and missing safeguards instead of a person's failings. It does not mean vague. "A config change was deployed" hides the fact that the deploy pipeline had no validation for that file, which is the useful finding.

Tells: "We take this incident very seriously", "some users may have experienced intermittent issues", "our team worked tirelessly around the clock", "a testament to our team's dedication", "out of an abundance of caution", "going forward, we are committed to…", and a timeline without timestamps ("in the early hours", "shortly after").

Before:

> On Tuesday, some users may have experienced intermittent issues with logging in. Our team worked tirelessly to identify and resolve the issue, which was caused by a configuration change. We sincerely apologize for any inconvenience.

After (facts from the incident notes):

> From 09:12 to 09:58 UTC on 6 October, 31% of login attempts failed with HTTP 502. A config deploy at 09:10 set the auth service's upstream timeout to 50 ms instead of 500 ms; the config linter does not check units. We rolled the config back at 09:55 and logins recovered by 09:58. Unit validation for timeout fields is tracked in AUTH-412.

Customer-facing status updates are shorter: what is affected, what users should do, when the next update comes. Do not speculate about cause in a status update.

## API reference

The reader is writing code against the endpoint right now. Every field must answer a question they would otherwise answer by trial and error.

- One line starting with a verb: "Returns the invoices for an organization, newest first."
- Method, path, authentication and the scope or role required.
- Parameters: name, type, required or optional, default, constraints (range, format, maximum length, allowed values), and what it does. "`userId`: The ID of the user" repeats the name; "`userId`: UUID of a user in the caller's organization; other organizations' IDs return 404" is documentation.
- A request example that runs as written, and the full response with every field.
- Errors: status, error code, when it happens, what the caller should do. Include which errors are safe to retry.
- Idempotency: whether a retry can create a duplicate, and the header or key that prevents it.
- Pagination, ordering, rate limits and side effects (emails sent, webhooks fired).

Tells: "This endpoint allows you to seamlessly retrieve…", "Returns a response containing the relevant data", descriptions that restate the parameter name, and examples with `"string"` as every value.

## Client emails

The client reads it on a phone between meetings. Put the news or the ask in the first two lines.

- One purpose per email. Several asks go in a numbered list so the reply can answer "1. yes 2. Thursday".
- Bad news first, with the new plan: "The staging deploy moves from Friday 9 October to Tuesday 13 October because the payment provider has not yet enabled our test account."
- What you need from them, and by when.
- Dates with weekday and date; times with a timezone when the reader is elsewhere.
- Apologize once, for the specific thing. "Sorry for the late notice" beats a paragraph about inconvenience.
- Match the thread: its language, its formality, how they address you. A Greek client who writes "Καλημέρα Αλέξη" does not need "Dear Mr. Papadopoulos, I hope this email finds you well."

Tells: "I hope this email finds you well", "I wanted to reach out regarding", "Thank you for your patience and understanding", "We apologize for any inconvenience this may cause", "Please do not hesitate to contact me", "Rest assured", bold headings or "Key points" inside an email, and a closing paragraph that restates the email.

## Release notes and changelogs

Readers want to know whether to upgrade and what will break.

- Version and date. Keep a Changelog recommends ISO dates (`2026-10-01`) and an `Unreleased` section at the top.
- Breaking changes first, each with the migration step: "`--config` no longer accepts YAML. Convert with `pgshift config convert old.yaml > pgshift.toml`."
- Then the changes, grouped. If the repository follows Keep a Changelog, the groups are Added, Changed, Deprecated, Removed, Fixed and Security.
- Each entry in user terms, with the issue or PR link: "Fixed: `pgshift status` no longer exits 0 when the database is unreachable (#231)."
- Security fixes name the affected versions and the advisory ID.

Tells: "We're thrilled to announce", "This release is packed with exciting new features", "various bug fixes and performance improvements", "under-the-hood enhancements", and an emoji per section.

## Security advisories

Readers are deciding how fast to patch. Give them exactly what they need to decide, and nothing that helps an attacker more than a defender. The content side of the fix belongs to the `security-engineering` skill; this section is about the writing.

- Title: the weakness and the component. "SQL injection in the `sort` parameter of `/api/reports`."
- Affected versions as an exact range, and the first patched version for each supported line.
- Severity with the scoring system named (for example a CVSS score with its vector string), and the CWE.
- What an attacker can do, and the preconditions: authentication needed or not, a non-default setting, network position.
- Workaround for people who cannot upgrade yet, and whether it fully closes the issue.
- How to tell whether you were affected, if you know (log lines, request patterns).
- CVE ID when assigned, credits, timeline if the project publishes one.

Do not minimize ("a minor issue in rare configurations") or inflate. "No evidence of exploitation" only with what was checked: "We found no requests matching the pattern in 90 days of our hosted service's access logs."

Tells: "We take security very seriously", "out of an abundance of caution", "a potential theoretical vulnerability that could possibly", and a description that never says what an attacker gains.

## UI microcopy

People do not read interfaces; they scan for the next action. Every word competes with the one they need.

- Buttons: verb plus object, and the same verb as the menu item or heading that opened the dialog. "Delete project", not "OK" or "Submit".
- Destructive confirmations name the object and the consequence: "Delete 'Q3 report'? The 14 files in it are removed for everyone." Buttons "Delete project" and "Cancel"; never "Yes" and "No".
- Errors: what happened, what to do, in the product's voice. "Card declined. Try another card or contact your bank." Keep what the user typed. No "Oops!", no bare "Something went wrong", no blame ("You entered an invalid date"). Add a reference code if support will ask for one.
- Empty states: what will appear here and how to start ("No invoices yet. Invoices appear here after your first paid order.").
- Sentence case, no exclamation marks, digits for numbers, one term per concept across the whole product.
- Labels above fields; placeholder text is not a label.

Tells: "Unlock your potential", "Let's get started!", "Oops! Something went wrong 😕", "Are you sure?" dialogs, and marketing adjectives in settings screens. For labels, focus and announcements see `accessibility`.
