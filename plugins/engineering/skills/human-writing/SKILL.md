---
name: human-writing
description: Write, edit or de-AI prose so it reads like a careful engineer wrote it, in English or Greek. Covers READMEs, commit messages, PR descriptions, incident reports, API docs, release notes, security advisories, client emails and UI copy. Use when drafting any of these, or when asked to make text sound less robotic, less like ChatGPT, or more natural.
license: MIT
metadata:
  author: Alex Tsanis
---

# Human writing

Write the way a senior engineer writes to a colleague they respect: the point first, then the evidence, then the limits, then stop. Fidelity comes before style. A rewrite that sounds human but moves a number, swaps a technical term or turns "mitigated" into "resolved" is worse than the robotic original, because nobody double-checks text that reads well.

## Know the job before touching the text

| Request | Keep | Change |
| --- | --- | --- |
| Draft | Brief, audience, evidence | Everything else |
| Edit | Author's voice and claims | Clarity, flow, errors |
| Rewrite | Meaning, facts, constraints | Structure and phrasing |
| Proofread | Everything but mistakes | Grammar, punctuation, consistency |
| Summarize | Fidelity and stated uncertainty | Length and emphasis |
| De-AI pass | Meaning, claims, recognizable voice | Machine habits at every level |

When the user supplies their own prose, its voice is the style guide. The owner of this plugin is Greek and writes English and Greek; natural non-native phrasing is a voice to keep, and only errors that change meaning or read as mistakes get fixed (`references/greek-and-english.md`).

## Failures that cost the most

These happen while rewriting and cleaning up. They cost more than any stylistic tell because they change what the text says.

- **Fact drift.** A number, unit, version, date, timezone, name or quoted string changes. "Up to 18%" becomes "18%", "p99 of 1,840 ms" becomes "about two seconds", "14:05 UTC" loses its timezone, "PostgreSQL 15" becomes "PostgreSQL". Locale traps too: Greek "1.500" is fifteen hundred and an English reader sees one and a half. Copy numbers and identifiers character for character, and diff them against the source afterwards.
- **Certainty drift.** Hedges that carried real uncertainty get cut along with the filler, or a tentative claim hardens. "We believe the cache caused it" becomes "The cache caused it"; "should" becomes "will"; "mitigated" becomes "resolved". Cut stacked hedges; keep the one that states what is unknown.
- **Term drift.** A term of art is swapped for a friendlier word that means something else: "idempotent" for "safe", "at-least-once" for "reliable", "p99" for "slowest requests", "Keycloak realm" for "area", "deprecated" for "removed". Whatever term the source uses, the rewrite uses too, every time.
- **Invented texture.** Anecdotes, customer quotes, round numbers, dates, names or "users love it" added so the text feels lived-in. Never add a fact the source does not contain. If the text needs a number you do not have, tell the user or write "not measured".
- **Word swap, same skeleton.** "Leverage" becomes "use" while the triple, the trailing "-ing" clause, the matched paragraphs and the closing summary survive. It still reads as generated. Repair structure first, words last.
- **Over-correction.** Removing a flagged word in its technical sense (API key, robust regression, `unlock()`, landscape orientation, privilege elevation), forcing a real list into prose, americanizing a British document, replacing Greek punctuation, or polishing the author's plain English into corporate idiom.
- **Second-generation slop.** The voice a model adopts when told to sound human: "Here's the deal:", "Short version:", "Honestly,", one-word fragments, jokes in an incident report. The target is plain, not casual.
- **Chat wrapper.** "Here's a more natural version:" before the text, three alternatives nobody asked for, a list of every change made. Deliver the text. Add a note only for something the user must act on, such as a contradiction in the source or a fact you could not confirm. Never explain stylistic edits (words removed, heading case changed, adjectives dropped); the user can see them.

## Machine-prose tells

One instance proves nothing. The same move repeated across a passage is a voice, and readers hear it. Mark the clusters, then repair from the document level down.

### Document

- **Announced structure, closing summary.** "In this guide we'll explore X, Y and Z" at the top; "In conclusion" or "Key takeaways" restating the middle at the bottom. Cut both. End on the last useful fact, the decision or the next step.
- **Title Case Headings.** Use sentence case: first word and proper nouns only. Follow the repository's convention if it has a different one.
- **Bold-label bullets.** Every bullet opens with `**Label:**`. Drop the labels, make a table if it is a real mapping, or write prose if the bullets were an argument.
- **Emoji headers and markers.** 🚀 Features, ✅, 💡 Tip, ⚠️. Remove them. Use the doc system's admonition if a warning must stand out.
- **Headings and bullets for everything.** A heading every two sentences, four heading levels in a one-page doc, reasoning chopped into bullets so every "because" disappears. Reasoning goes in paragraphs; lists are for parallel items people scan.
- **Boilerplate sections.** "Why this matters", "Benefits", "Challenges and future outlook", an FAQ that repeats the body. Keep a section only if deleting it loses a fact.
- **Fake specificity.** "Up to 10x faster", "99.9% uptime", "trusted by thousands", "studies show", none of it in the source. Only numbers from the source, or from a measurement you describe.
- **Artifacts.** `[Your Name]`, `[insert link]`, "As of my last update", citation debris (`oaicite`, `contentReference`, `turn0search0`), a stray "```markdown" fence around the whole answer.

### Paragraph

- **Symmetric paragraphs.** Every paragraph three or four sentences: claim, support, support, significance. Let content set the length; a one-sentence paragraph is fine when that is the whole thought.
- **Significance closers.** "This makes X an excellent choice for teams of any size." Cut it. A real consequence belongs in the paragraph as a fact with its mechanism.
- **One fact smeared across three sentences.** "Performance matters. That's why we reworked the query. It is now much faster." becomes one sentence carrying the numbers from the source.
- **Heading echo.** "## Installation" followed by "This section explains how to install X." Start with the first step.

### Sentence

- **Em dashes.** Any em dash, and any en dash, " - " or " -- " doing the same job. Replace by function: commas for an aside inside the sentence, a colon when what follows explains or lists, parentheses for an aside the sentence reads fine without, a full stop when both halves are sentences. Swapping the character for a hyphen keeps the tell.
- **Contrast templates.** "Not only X but also Y", "not just X, but Y", "it's not about X, it's about Y", "X isn't just a Y. It's a Z.", "No X. No Y. Just Z." Say what the thing is. A real contrast gets stated once: "The limit is per API key, not per IP."
- **Reflexive triples.** "Fast, reliable and scalable." Three because three sounds finished. Keep the items that carry distinct information; two or four is fine.
- **Participial tails.** ", ensuring…", ", highlighting…", ", underscoring…", ", making it…", ", allowing teams to…", ", paving the way for…". A benefit attached with no mechanism. Cut it, or make it a claim: "keyed by tenant, ensuring isolation" becomes "the cache key includes the tenant ID, so one tenant cannot read another's entries."
- **Copula avoidance.** "Serves as", "stands as", "acts as", "represents", "boasts", "features". Write "is" or "has".
- **Self-answered questions and colon reveals.** "The result? A smaller bundle." "Here's the thing:" "The catch:" State it.
- **Punchline fragments.** "Simple." "That's it." "Enter Redis." A full sentence or nothing.
- **False ranges.** "From startups to enterprises", "whether you're a beginner or an expert". Say who it is for.
- **Synonym rotation.** The tool, the platform, the solution: one thing, three names. In technical text a new noun implies a new thing. Pick one name.
- **Actor hidden by passive.** "An issue was experienced", "the table was dropped". Say what did it: "a cleanup job dropped the table." Passive is fine when the actor is unknown or irrelevant.

### Phrase and word

- **Signposting.** "It's important to note that", "It's worth mentioning", "In today's fast-paced world", "Let's dive in", "When it comes to", "As mentioned above". Delete; start with the fact.
- **Empty connectives.** "Moreover", "Furthermore", "Additionally", "Ultimately", "Overall" opening sentences. Use the real relation (because, so, but) or nothing.
- **Stacked hedges.** "Could potentially", "may possibly", "it could be argued that this might". One hedge, on the claim it qualifies, with the reason if you know it.
- **Intensifiers.** Truly, incredibly, extremely, highly, deeply, remarkably. Cut, or give the number.
- **Sycophancy.** "Great question!", "You're absolutely right", "What a thoughtful approach", praising a draft before editing it. Start with the answer.
- **Stock email lines.** "I hope this email finds you well", "I wanted to reach out", "Please don't hesitate to contact me", "Thank you for your patience and understanding", "We apologize for any inconvenience". Open with the reason for writing; apologize once, for the specific thing.
- **Slop vocabulary.** Replace with the plain word, or better, with the fact the word was standing in for:

| Tell | Write instead |
| --- | --- |
| delve into | look at; or start with the finding |
| tapestry, landscape, realm | name the actual parts, market or field; usually cut |
| foster, streamline, elevate | the mechanism or the measured change ("four steps become one") |
| unlock, empower | let, allow |
| robust | the property: survives a node loss, rejects malformed input |
| seamless(ly) | without a restart, without code changes; or cut |
| navigate (a problem), journey | handle, work through; process, steps |
| crucial, vital, essential, pivotal, key (adjective) | important, required; or the consequence of skipping it |
| boasts, features, offers | has |
| nestled | is in |
| vibrant | active; or cut |
| intricate | complicated; better, the part that is complicated |
| meticulous(ly) | careful(ly); better, what was checked |
| holistic, comprehensive | whole, end-to-end; or list the coverage |
| leverage, utilize | use |
| testament to, underscores, highlights | shows; or cut |
| myriad, plethora, a wide array of | many, or the number |
| cutting-edge, game-changer, revolutionize | the version, or the effect with numbers |

The longer list, with the technical senses in which each word is correct and should stay, is in `references/tells.md`.

## Detail without padding

Thorough means every sentence gives the reader something to act on or check. Padding is anything they would not miss.

- Detail is facts: numbers with units, versions, system names, commands, file paths, error strings, timestamps with timezone, causes, conditions, limits. Adjectives are not detail. "Fast" is padding; "p50 of 12 ms on the 40 GB table" is detail.
- Run the deletion test on each sentence. Remove it and ask what the reader lost. If nothing, it goes. If one fact, keep the fact and drop the words around it.
- Say each fact once, where the reader needs it. An intro, body and summary that repeat each other triple the length and add nothing.
- Put a limit next to the claim it limits: "Works on PostgreSQL 13 and later; the `MERGE` path needs 15." A "Limitations" section three screens away gets skipped.
- Length follows content. Twelve real migration steps need twelve steps. A one-line fix gets a one-line commit message.
- Put depth where readers can skip it: tables, a details section, an appendix. A reader who stops after the first paragraph should still have the answer.
- When the source is thin, do not fill it. Ask for the number, or say it was not measured.

## Decision rules

- Use a list when the items are parallel and independent and the reader will scan them; number it when order matters. Use prose when one point depends on another, because a list drops the "because" and the "so".
- Add a heading when the reader will scan or come back to that part. A document shorter than a screen rarely needs one.
- Hedge once, on the claim it qualifies, and say what is unknown: "Failed on 3 of 40 runs; not reproduced outside CI."
- Prefer the active voice when the actor matters (incidents, decisions, who does what next). Use the passive when the actor is unknown or irrelevant.
- Keep a flagged word when it is the literal term in the product, API, standard or source (API key, realm, `unlock()`, CPU utilization). Replace it when it is praise.
- Match the register to the reader and the thread, not a house style. Incident reports and advisories: neutral and exact, no warmth added. Client email: polite, direct, one apology at most. UI: short, verb first.
- When asked to edit, change sentences that need it and leave the rest. Rewrite wholesale only when asked to rewrite.
- When the source contains a claim you cannot support or two facts that contradict, keep the text faithful and tell the user. Do not smooth it over or silently delete it.
- Follow the repository's conventions (commit format, changelog categories, heading case, spelling) over anything here.

## Self-check on every draft

Run this on every piece of prose you produce, including replies to the user, and before handing back any rewrite.

1. Read the first sentence. Does it give the answer, the change or the ask? If it announces the topic, delete it and check the new first sentence.
2. For rewrites, list every number, unit, version, date, time, timezone, name, identifier, quoted string and hedge in the source, and find each one unchanged in the draft. Check that no claim got stronger or weaker and that nothing new appeared.
3. Structure: sentence-case headings only where needed, no bold-label bullets, no emoji, no closing summary, paragraph lengths that follow the content.
4. Sentences: find every em dash and dash substitute, "not only" and "not just", triples, trailing ", -ing" clauses, "serves as", self-answered questions and fragments. Repair each by function.
5. Words: check each word from the vocabulary table. Replace it, or confirm it is the technical term in context.
6. Read top to bottom as the intended reader. Every sentence passes the deletion test, and sentence lengths vary with the thought.
7. Check the repairs did not add second-generation slop or a chat wrapper.
8. If the draft is in a file, run the Grep patterns at the end of `references/tells.md` and review each hit.

## Review checklist

- Does the first sentence carry the point?
- Is every number, version, name, term, date and timezone from the source present and unchanged?
- Is every hedge that expressed real uncertainty still there, and every stacked hedge gone?
- Is there no fact, example, quote or number the source does not contain?
- Are there zero em dashes, and no hyphen or en dash doing an em dash's job?
- Are "not only…but also", "not just X but Y", reflexive triples and trailing -ing clauses gone?
- Are headings in sentence case, bullets free of bold labels, and the text free of emoji?
- Does the text end on a fact, decision or next step rather than a summary or an offer to help?
- Does each flagged word that remains have its technical meaning in context?
- Is the author's voice still recognizable, including non-native phrasing that reads correctly?
- Does the text contain what its genre requires (see `references/genres.md`)?
- Is the delivered text free of a preamble, alternatives and change lists nobody asked for?

This is a quality pass, not an authorship guarantee. Never claim text will pass an AI detector or is "human-written".

## References

- `references/tells.md`: read for an explicit de-AI pass or a review of someone else's text; the full word list with technical exceptions, a phrase bank, sentence repairs, second-generation slop and Grep patterns for files.
- `references/genres.md`: read when writing or rewriting a README, commit message, PR description, incident report, API reference, client email, release notes, security advisory or UI copy.
- `references/rewrites.md`: read when you want a full worked rewrite to calibrate against, including an over-corrected draft.
- `references/greek-and-english.md`: read when editing English by a Greek speaker, writing to Greek clients, or writing Greek.
- Related skills: `security-engineering` for the substance of an advisory, `accessibility` for UI copy in context.
