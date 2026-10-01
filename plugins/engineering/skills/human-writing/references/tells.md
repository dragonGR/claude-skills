# Machine-prose tells in detail

Read this during an explicit de-AI pass, when reviewing someone else's text, or when the self-check in SKILL.md flags a pattern and you need the repair or need to decide whether a flagged word is the correct technical term.

A single tell proves nothing. One "crucial" in a long design doc is fine. Four of them, a triple in every paragraph and a summary at the end is a voice, and that voice is what readers recognize. Mark the clusters across the whole passage first, then repair from the largest unit down: document, paragraph, sentence, phrase, word.

## Word list with exceptions

The replacement is often the fact the word was standing in for. "A robust retry mechanism" says nothing; "retries three times with exponential backoff, then moves the message to the dead-letter queue" says what robust meant, if the source supports it. When you do not have the fact, use the plain word or delete the adjective.

| Word | Write instead | Keep it when |
| --- | --- | --- |
| delve (into) | look at, examine; or start with the finding | no technical sense |
| tapestry | name the parts ("twelve services owned by three teams") | no technical sense |
| landscape | the market, the options, the threats; or cut the clause | page or screen orientation |
| realm | area, field; usually cut ("in the realm of databases" becomes "in databases") | Kerberos realms, Keycloak realms, other named auth concepts |
| foster | encourage, lead to; better, the mechanism | no technical sense |
| streamline | simplify, remove steps; say which ("one command instead of four") | no technical sense |
| elevate | improve, raise; say by how much | privilege elevation, elevated permissions |
| unlock | let, allow, make possible | real locks: mutexes, `unlock()`, account lockout, device unlock |
| empower | let, allow | no technical sense |
| robust | the property: survives a node loss, validates input, retries with backoff | robust statistics (robust regression, robust estimator) |
| seamless(ly) | without a restart, without code changes, automatically; or cut | seamless tiling or textures in graphics |
| navigate | handle, find, work through | UI and routing ("navigate to Settings", `router.navigate()`) |
| journey | process, steps, onboarding flow | a product's own term ("customer journey" in an analytics tool) |
| crucial, vital | important; or the consequence ("without it, writes are lost") | no technical sense |
| essential | required, needed | "strictly necessary" or "essential" cookies as a consent category |
| pivotal | important, deciding | pivot tables, pivot rows in algorithms ("pivot", not "pivotal") |
| key (adjective) | main, important; or cut | the noun: API key, primary key, key rotation, keyboard key |
| boasts | has | no technical sense |
| nestled | is in, sits in | no technical sense |
| vibrant | active, busy; or cut | no technical sense |
| intricate | complicated, detailed; better, name the complicated part | no technical sense |
| meticulous(ly) | careful(ly); better, say what was checked | no technical sense |
| holistic | whole, end-to-end, covering X and Y | no technical sense |
| leverage | use | financial leverage |
| utilize, utilization | use | CPU or memory utilization as a metric |
| facilitate | help, let, run | no technical sense |
| enhance, enhancement | improve; say what changed | a tracker label ("enhancement") or product tier |
| optimize | make faster, smaller or cheaper; name the metric | compiler flags (`-O2`), mathematical optimization, query planner |
| comprehensive | full, complete; or list what it covers | no technical sense |
| cutting-edge, state-of-the-art | new, current, or the version number | a cited comparison in a paper |
| game-changer, revolutionize | the effect, with numbers if you have them | no technical sense |
| harness (verb) | use | test harness, wiring harness |
| ecosystem | tools, integrations, libraries | package ecosystems (npm, PyPI, crates.io) and advisory databases that use the word |
| paradigm | approach, model | programming paradigms in CS writing |
| synergy | cut | no technical sense |
| testament to | shows | no technical sense |
| underscore, highlight (as "shows importance") | shows; or cut | syntax highlighting, UI highlight, the `_` character |
| showcase | show | a component or page named Showcase |
| myriad, plethora, a wide array of, a diverse range of | many, several, or the number | no technical sense |
| bolster | strengthen, support | no technical sense |
| garner | get, attract | no technical sense |
| interplay | how X affects Y | no technical sense |
| endeavor | work, project, attempt | no technical sense |
| commence | start | legal text that already uses it |
| embark on | start | no technical sense |
| resonate with | matters to, suits | physics and signal processing |
| align with | match, follow, agree with | memory alignment, layout alignment |
| insights | findings, what we learned, the numbers | a product or dashboard named Insights |
| stakeholders | name them: support, finance, the client's CTO | governance documents that define the term |
| actionable | concrete; say what the reader can do | "actionable alert" in on-call writing |
| nuanced | name the nuance | no technical sense |
| intuitive | say why it is easy to use, or cut | no technical sense |
| effortless(ly), simply, just (as reassurance) | cut | "just" meaning "only" or "a moment ago" |
| invaluable | useful | no technical sense |
| groundbreaking, renowned, world-class, best-in-class | cut | no technical sense |
| exciting, thrilled, delighted | cut; "We released X" | no technical sense |
| ever-evolving, fast-paced, rapidly changing | cut the clause | no technical sense |
| supercharge, turbocharge | speed up by the measured amount | no technical sense |
| in order to | to | no technical sense |

Sentence-opening adverbs that only add weight: notably, importantly, crucially, interestingly, ultimately, essentially, fundamentally, arguably. Cut them. Intensifiers do the same inside a sentence: truly, really, incredibly, extremely, highly, deeply, remarkably, significantly. Keep "significantly" only for a statistical result with its test.

## Phrase bank

Throat-clearing and signposting. Delete the phrase and start with what followed it.

- "It's important to note that", "It's worth noting/mentioning that", "Keep in mind that"
- "In today's fast-paced digital world", "In the ever-evolving landscape of", "In an era where"
- "Let's dive in", "Let's take a closer look at", "Let's explore", "Let's break it down"
- "In this article/guide/section, we will explore", "The following section outlines", "Below is a breakdown of", "Here's what you need to know"
- "As mentioned above", "As we discussed earlier" (link the section if the reader needs it)
- "When it comes to X", "In terms of X", "In the context of X", "With regard to X": make X the subject. "When it comes to caching, Redis is a popular option" becomes "Most of our services cache in Redis" (if that is the fact).

Empty connectives. "Moreover", "Furthermore", "Additionally", "In addition", "What's more" at the start of a sentence usually join two facts that need no joint. If there is a real relation, write it: because, so, but, which means, until. If there is none, start the next sentence with its subject.

Wrap-ups. "In conclusion", "In summary", "To sum up", "All in all", "Overall", "At the end of the day". If the paragraph after the label restates, delete the paragraph. If it holds a decision or next step, keep that sentence without the label.

Copula avoidance. "X serves as the entry point" becomes "X is the entry point". "The library boasts/features/offers a CLI" becomes "The library has a CLI". "This marks/represents a shift" becomes "This is a change", and then usually you can say what changed instead.

Vague attribution. "Experts agree", "Studies show", "Industry reports suggest", "Many developers find", "It is widely considered". Name the source, make the claim in your own name with your evidence, or cut it.

Stacked hedges. "May potentially", "could possibly", "might perhaps", "it could be argued that this may", "to some extent, in some cases". Decide the confidence, then write it once: "may", or better, what is known and what is not ("Failed on 3 of 40 runs; we have not reproduced it outside CI").

Reassurance and absolutes. "Rest assured", "fully secure", "guaranteed", "100% reliable", "zero risk". These are claims. Keep them only if they are true and you can say why; otherwise state the actual property.

Sycophancy and chat residue. "Great question!", "You're absolutely right", "What a thoughtful approach", "Certainly!", "Absolutely!", "I hope this helps", "Let me know if you have any other questions", "Feel free to reach out", "Happy to help further". Start with the answer; end when the content ends. One concrete offer is fine: "If step 3 fails, send me the output of `pg_dump --version`."

Stock email phrases. "I hope this email finds you well", "I wanted to reach out", "I'm writing to inform you", "Please don't hesitate to contact me", "Thank you for your patience and understanding", "We apologize for any inconvenience this may have caused". Open with the reason for writing. Apologize once, for the specific thing, with the specific consequence: "Sorry for the delay; the invoice will go out on Tuesday instead of Friday."

Model self-talk inside a document. "As an AI language model", "As of my last update", "I don't have access to real-time information", "I cannot browse". Remove. If a fact might be stale, name the fact and the date it was true.

## Sentence repairs

Em dashes. Decide what the dash was doing, then use the mark that does that job.

- Aside inside a sentence, use commas: "The worker, which runs every minute, checks the queue."
- The second part explains or lists, use a colon: "One default changed: the request timeout is now 10 seconds."
- An aside the sentence reads fine without, use parentheses: "Set `PGSSLMODE` (the client default is `prefer`) to `verify-full`."
- Two complete thoughts, use two sentences: "The import finished. Two rows were rejected for duplicate emails."
- A dramatic pause ("It worked, until it didn't"), cut the drama and state the event from the source: "It worked until the queue passed 10,000 messages."

A spaced hyphen, a double hyphen or an en dash doing the same job is the same tell. En dashes between numbers ("pages 10–12") are typography, not a tell, but in plain Markdown prose "10 to 12" is safer.

Contrast templates. State what the thing is. Keep a contrast only when the reader is likely to hold the wrong idea, and state it once.

- "This isn't just a cache, it's a consistency layer" becomes "The cache also invalidates entries when the source row changes."
- "Not only does it reduce latency, but it also cuts costs" becomes "It reduces latency and cost" plus the numbers, if the source has them.
- "It's not about speed. It's about predictability." becomes "The goal is a stable p99, even if the median gets slightly worse."
- "No config. No boilerplate. Just results." becomes "It runs with no configuration file."
- A real contrast, stated once: "The limit is per API key, not per IP address."

Reflexive triples. "Fast, secure, and scalable." "Plan, build, and ship." Three parallel clauses in a row, three bullets under every heading. Count the actual distinct things. If two adjectives mean one thing, keep one. If the third was filler, drop it. Four real items stay four.

Participial tails. A comma and an -ing verb that attaches a benefit to the end of a sentence: ", ensuring…", ", highlighting…", ", underscoring…", ", emphasizing…", ", reflecting…", ", making it easy to…", ", allowing teams to…", ", enabling…", ", paving the way for…", ", contributing to…". The benefit arrives without a mechanism. Either cut it or turn it into a supported claim with its own subject and verb.

- "Requests are signed with HMAC, ensuring security" becomes "Requests carry an HMAC-SHA256 signature over the body and timestamp, so a replayed or altered request fails verification" (only if the source says it covers the timestamp).
- "The team migrated to PostgreSQL 16, highlighting its commitment to modern tooling" becomes "The team migrated to PostgreSQL 16."

Self-answered questions and colon reveals. "The result? A 40% smaller bundle." "Why does this matter? Because…" "Here's the thing:" "The catch:" "The best part?" Write the statement: "The bundle is 40% smaller."

Punchline fragments. "Simple." "That's it." "Game over." "Enter Redis." "No more waiting." Write a full sentence, or nothing.

False ranges. "From solo developers to Fortune 500 teams", "whether you're a beginner or a seasoned pro". Say who it is for and what they need ("for teams running PostgreSQL on a single primary"), or cut it.

Synonym rotation. "The tool", "the platform", "the solution", "the framework" for one thing. In technical writing a new noun implies a new thing, so readers look for the second system. Pick one name and repeat it.

Nominalizations. "Perform an analysis of" becomes "analyze"; "provide support for" becomes "support"; "carry out the implementation of" becomes "implement"; "make a determination" becomes "decide"; "is in alignment with" becomes "matches".

"This" chains. "This ensures… This allows… This means…" as consecutive openers. Name the subject each time ("The lock prevents…", "The retry means…"), or merge the sentences.

Actor hidden by passive. "An issue was experienced", "the table was dropped", "mistakes were made". Name what did it: "A cleanup job dropped the table." In blameless writing, name the role and the system that allowed the action, not the person. Keep the passive when the actor is unknown ("the key was leaked; we don't yet know how") or does not matter ("the package is published to npm").

## Paragraph and document repairs

Symmetric paragraphs. Generated text tends toward paragraphs of three or four sentences, each shaped as claim, support, support, significance. Readers feel the metronome even if they cannot name it. Let content decide length. Merge two thin paragraphs that make one point; split a long one at the point where the subject changes; leave a one-sentence paragraph when that is the whole thought.

Significance closers. The last sentence of a paragraph that says why the paragraph mattered ("This makes it a strong choice for teams of any size", "This highlights the importance of testing"). Delete it. If the consequence is real and not obvious, it belongs in the paragraph as a fact with its mechanism.

Announce, say, repeat. An intro that lists what the document covers, a body, and a conclusion that lists what it covered. Keep the body. If the document is long, one line of scope at the top is fine ("This covers upgrades from 3.x; for new installs see INSTALL.md").

Heading hygiene. Sentence case. No heading that has only one short paragraph under it in a document a reader will not scan. No heading level skipped. No heading that repeats the document title. No "Introduction" heading over the first paragraph.

Lists. Bold-label bullets (`- **Speed:** It's fast.`) for every item turn a list into a slide. Drop the labels if the bullet text already says it; use a table if it is a real mapping of names to values; use prose if the bullets were an argument. Bullets that are all the same length and all start with a gerund ("Improving…", "Enhancing…", "Streamlining…") are a second tell; write them as the specific changes.

Emphasis. Bold used on a phrase in every paragraph dilutes to nothing. Keep bold for the one thing a scanning reader must not miss, or for terms at their definition. Emoji go.

Unrequested sections. "Why this matters", "Benefits", "Key features" written as praise, "Challenges and future outlook", "Best practices" of generic advice, an FAQ restating the body, "Conclusion". Keep a section only if deleting it loses a fact.

Fake specificity. Round numbers, "up to 10x", "99.9%", "trusted by thousands", "saves hours every week", named customers or testimonials, dates, that are not in the source. They read as specific and are invented. Remove them, or ask the user for the real figure.

Artifacts. `[Your Name]`, `[Company]`, `[insert link]`, lorem ipsum, a leftover "```markdown" fence around the whole answer, citation debris (`oaicite`, `contentReference`, `turn0search0`, `[cite: 1]`), and quote marks that switch between curly and straight in a file that used one style.

Style shift. A generated section pasted into a human document stands out because it is smoother, longer and more abstract than what surrounds it. Match the surrounding sentence length, vocabulary and level of detail.

## Second-generation slop

Told to sound human, a model often swaps one set of habits for another set that is just as recognizable.

- Forced casualness: "Here's the deal", "Let's be real", "Honestly,", "Spoiler:", "kinda", "a bit of a", "TL;DR" at the top of a three-line note, jokes in technical or incident writing.
- Fragments for rhythm: "Short version:", "Simple.", "Worth it.", "Big difference."
- Colon reveals: "The kicker:", "The fix:", "Bottom line:".
- Announcing the style: "No fluff.", "In plain English:", "Without the jargon:".
- Uniformly short sentences, over-correcting from long ones, so the text reads like a telegram.
- Contractions sprinkled into formal text that had none, or lowercase-only styling.

Aim for plain: the register the audience expects, sentences as long as the thought.

## Grep patterns

When the draft is in a file, run each pattern with the Grep tool (`output_mode: content`). Every hit is a candidate for review. Technical senses from the word list above stay, and proper nouns in headings are fine.

| Check | Pattern |
| --- | --- |
| Dashes | `[\x{2014}\x{2013}]\|[^\s>] -{1,2} \S` |
| Vocabulary | `(?i)\b(delv\|tapestr\|landscape\|realm\|foster\|streamlin\|elevat\|unlock\|empower\|robust\|seamless\|navigat\|journey\|crucial\|pivotal\|vital\|essential\|boast\|nestled\|vibrant\|intricat\|meticulous\|holistic\|leverag\|utiliz\|facilitat\|comprehensive\|cutting-edge\|game-chang\|testament\|underscor\|showcas\|myriad\|plethora\|bolster\|garner\|interplay\|synerg\|paradigm\|embark\|ever-evolving\|fast-paced)\w*` |
| Signposts and connectives | `(?i)\b(moreover\|furthermore\|additionally\|in conclusion\|in summary\|to sum up\|ultimately\|notably\|importantly\|when it comes to\|at the end of the day\|in today['’]?s\|let['’]?s (dive\|explore\|take a)\|as mentioned (above\|earlier)\|it['’]?s (important\|worth) (to note\|noting\|mentioning))\b` |
| Contrast templates | `(?i)\bnot only\b\|\bnot just\b\|\bit['’]?s not about\b\|\bisn['’]?t just\b` |
| Participial tails | `, (ensuring\|highlighting\|underscoring\|emphasizing\|showcasing\|reflecting\|demonstrating\|making it\|allowing\|enabling\|paving\|fostering\|contributing)\b` |
| Copula avoidance | `(?i)\b(serves\|stands\|acts\|functions) as\b\|\bboasts\b` |
| Stacked hedges | `(?i)\b(may\|might\|could\|can) (potentially\|possibly\|perhaps)\b\|\bit could be argued\b` |
| Reveals | `(?i)\b(the (result\|catch\|kicker\|best part)\|here['’]?s the (thing\|deal\|catch))[?:]` |
| Chat residue | `(?i)great question\|absolutely right\|i hope this (helps\|email finds)\|feel free to\|don['’]?t hesitate\|rest assured\|any inconvenience\|patience and understanding\|as of my last` |
| Bold-label bullets | `^[\s>]*([-*+]\|\d+\.) \*\*[^*]+\*\*` |
| Capitalized words in headings | `^#{1,6} +\S+.* [A-Z][a-z]` |
| Emoji | `\p{Extended_Pictographic}` |
| Artifacts | `(?i)oaicite\|contentreference\|turn\d+search\d+\|\[cite: ?\d+\]\|\[(your\|insert) [^\]]*\]` |

The table escapes `|` as `\|` for Markdown; pass the pattern to Grep with plain `|`.
