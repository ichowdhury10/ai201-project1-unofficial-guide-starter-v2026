# The Unofficial Guide

Injam Chowdhury — `advice_threads` corpus.

> **This file is your submission.** Fill it in as you go — most sections get
> written during the milestone that produces them, not at the end.
>
> How the starter works, and every command you'll need, is in `RUNNING.md`.
> Leave that file alone.
>
> **Paste everything as text.** No screenshots, no video. A typed table gets
> full credit; a picture of the same table gets none.
>
> Delete these instruction blocks as you replace them. The `<!-- -->` comments
> are notes to you and don't show up when the page renders — you can leave them
> or remove them.

---

# Unit 1

## What This Does

This is a retrieval-augmented Q&A system over `advice_threads`, 23 anonymous
student advice threads on things like laundry timing, parking permits,
laptop specs, and late-work policy — each one a question followed by three
to five replies that often disagree with each other. Ask it something the
threads cover ("how much RAM do I need for CS courses?") and it retrieves
the reply that answers it, checks the match is close enough to trust, and
has Gemini write a short answer naming the thread file it came from. Ask it
something the corpus doesn't cover (oil changes, World Cup trivia) and the
relevance gate refuses instead of guessing.

## Chunking Strategy

**Chunk size:** one reply per chunk (not a character count) — 132 to 281
characters in practice, ~202 on average.
**Overlap:** none.

Every document here is one thread: a `THREAD:` title line, then three to five
`--- reply N (votes) ---` blocks that often disagree with each other. Reading
these in Milestone 1, two things stood out. First, a single reply is
frequently one short sentence that means nothing on its own — "16 is the
answer," "Both true.," "Doesn't roll over between semesters" — and only makes
sense paired with the question it's answering. Second, `fallback_split`'s
800-character window is bigger than almost every thread (23 documents
average 543 characters), so it was mostly turning each *whole thread* into
one chunk — merging three to five disagreeing replies into a single chunk
and asking retrieval to treat them as one idea.

`split_documents` now cuts on the reply boundary instead of a character
count: each chunk is the thread title plus exactly one reply. Repeating the
title into every chunk fixes the first problem (a reply chunk still reads as
a complete thought), and one-reply-per-chunk fixes the second (retrieval
returns a specific claim instead of a whole disputed thread). No overlap,
because replies don't share any text with each other — overlapping would
only duplicate a reply's own sentences into its neighbor. This raised the
chunk count from 26 (fallback) to 75.

## Sample Chunks

**Chunk 1** — source: `thread_bike_commute.txt#0` — produced by: `chunker.py::split_documents`

```
THREAD: Is a bike worth it for a 20 minute walk commute?

--- reply 1 (14 votes) ---
Yeah. Cuts an 18 minute walk to about 6. The thing nobody mentions is storage — covered bike parking exists at three buildings and is full by 9am at all three.
```

**Chunk 2** — source: `thread_first_gen.txt#1` — produced by: `chunker.py::split_documents`

```
THREAD: Anything specific for first-generation students?

--- reply 2 (41 votes) ---
The thing I'd say: the unwritten rules are the hard part, not the coursework. Ask about the unwritten rules explicitly. People are happy to explain them and nobody volunteers them.
```

**Chunk 3** — source: `thread_laptop_specs.txt#2` — produced by: `chunker.py::split_documents`

```
THREAD: How much laptop do I actually need for CS courses?

--- reply 3 (12 votes) ---
I did two years on an 8GB machine and it was fine until the last project, at which point it very much wasn't. 16 is the answer.
```

**Chunk 4** — source: `thread_parking.txt#1` — produced by: `chunker.py::split_documents`

```
THREAD: Worth getting a parking permit?

--- reply 2 (21 votes) ---
Street parking on Verrill is legal and free and unmarked, which is why half the upper years do it.
```

**Chunk 5** — source: `thread_sleep_schedule.txt#1` — produced by: `chunker.py::split_documents`

```
THREAD: Everyone says fix your sleep. Does it actually matter?

--- reply 2 (37 votes) ---
The library being open until 2am is a trap. It's a resource, not a schedule.
```

Reading these back: all five read as one complete thought — a question plus
one full reply, no sentence cut in half at either end — which is what
criterion 4 checks for.

## Sample Answer

**Question:** How much RAM do students recommend for CS courses?

**Answer:**

```
Students recommend 16GB of RAM for CS courses, noting that while an 8GB
machine can work initially, it falls short on later projects.

Source: thread_laptop_specs.txt

Sources retrieved: thread_laptop_specs.txt, thread_pass_fail.txt
```

(best distance 0.199, cutoff 0.65 — via `python app.py ask "How much RAM do
students recommend for CS courses?"`)

**My relevance cutoff:**

I set `THRESHOLD = 0.65` in `config.py`. I ran my five test questions and the
five in `OUT_OF_SCOPE` through `python app.py retrieve` and recorded the best
distance for each. In-scope questions came back between 0.199 and 0.567;
out-of-scope came back between 0.807 and 0.896 — a clean gap with nothing
from either group inside it. I put 0.65 near the in-scope end of that gap
rather than in the middle, so a borderline-relevant question is more likely
to get an answer than a refusal, while still leaving 0.157 of headroom below
the closest out-of-scope distance I saw.

| Question | In corpus? | Best distance |
|---|---|---|
| When do applications open for summer internships? | Yes | 0.204 |
| Which campus has parking passes available? | Yes | 0.536 |
| Which mornings do students recommend for finding available laundry machines? | Yes | 0.394 |
| How much RAM do students recommend for CS courses? | Yes | 0.199 |
| Which office handles documented illness affecting assignment deadlines? | Yes | 0.567 |
| What is the capital of Mongolia? | No | 0.893 |
| How do I change the oil in a diesel engine? | No | 0.896 |
| Who won the 1994 World Cup? | No | 0.893 |
| What is the recommended dosage of ibuprofen for a headache? | No | 0.807 |
| How do I write a for loop in Rust? | No | 0.835 |

## How I Used AI

**1.** I asked Claude to write Milestone 3's `split_documents` from what I'd
noticed reading the threads: that `fallback_split`'s 800-character window
was bigger than almost every whole thread, so it was merging three to five
disagreeing replies into one chunk, and that a single reply often means
nothing without the question above it. It came back with a version that
splits on the `--- reply N ---` markers and repeats the `THREAD:` line into
every chunk, with a fallback to the old chunker for any document that isn't
in that shape. I checked the output of `python app.py chunks -n 5` myself to
confirm none of the five samples cut a reply in half before trusting it.

**2.** I asked Claude to actually run retrieval for my five test questions
and the five `OUT_OF_SCOPE` ones and report the best distance for each,
rather than have me copy numbers out of the terminal by hand for the
threshold table. It came back with the two groups (0.199–0.567 in-scope,
0.807–0.896 out-of-scope) and a suggested cutoff at 0.65; I picked where in
that gap to put it myself — close to the in-scope side rather than the
midpoint — and wrote the reasoning in `criteria.md` in my own words.

**3. (Unit 2, Milestone 2)** After my own first pass through all five
criteria came back all-MET, I had Claude run the exact check Milestone 2
suggests — two independent reviews, each handed only the real questions,
retrieved chunks, and generated answers (no view of my own reasoning),
instructed to argue for the opposite verdict as strongly as possible on my
two closest calls, then give an honest final recommendation. One review
argued that my "parking" verdict should flip; the other argued something I
hadn't considered at all — that my *internships* question also doesn't hold
up, because it asks when applications "open" while the corpus only ever
states when they "close" or "hire," and separately that my own `expects`
field for parking ("West Campus") reads the source sentence backwards
("West lots sell out... East lot never sells out" is better evidence for
East). I didn't take either claim on faith — I re-read both source
documents myself and confirmed both held up on a plain reading. I accepted
the internships and parking findings for criterion 1 (flipping it from MET
to MISSED, 3/5), but rejected the same reviewer's attempt to also fail
criterion 5 over internships, since the model's actual answer there
truthfully reports what the source says ("close," not "open") — nothing
false was claimed, so criterion 5 isn't about the same defect. Without this
check I would have submitted five MET verdicts built partly on my own too-
generous reading of two questions I'd written myself.

<!-- ── Stretch features ─────────────────────────────────────────────────────
     Doing one? Say so here BEFORE you start. A feature this README never
     claims earns nothing.
     ───────────────────────────────────────────────────────────────────────── -->

---

# Unit 2

<!-- These sections get ADDED to what's already above. Don't delete or rewrite
     unit 1 — the point is that someone can see what you said before you knew
     how it went. -->

## Run Log — Before

Produced by `run_eval.py::main` (retrieval evidence from `store.py::search`
over chunks from `chunker.py::split_documents`; gate evidence from
`gate.py::check` via `run_eval.py::check_out_of_scope`). Full file:
[`results/run_2026-09-29_2206_before.md`](results/run_2026-09-29_2206_before.md).
Corpus `advice_threads`, top-k 5, cutoff 0.65, 3 runs per question, caching off.

Criteria 1, 3, and 4 don't vary between runs — retrieval and the gate are
deterministic (same question, same embeddings, same fixed cutoff every time),
and criterion 4 is a one-time check of the five samples already in this
README, not something `run_eval.py` re-measures per run. Only criteria 2 and 5
depend on what the model actually generates, which is why those are the ones
worth running three times.

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 | 3/5 | 3/5 | 3/5 | MISSED |
| 2. Every answer names a source | 5 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 3. Gate stops out-of-corpus questions | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 4. Sample chunks read as thread title + one complete reply | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 5. Every factual claim supported by its cited source | 4 of 5 | 4/5 | 4/5 | 4/5 | MET |

> Criterion 1 first came back 5/5 in my own initial read. It's 3/5 here
> because I ran the adversarial check Milestone 2 itself suggests — "argue
> the opposite verdict as strongly as you can" — before locking these in,
> and it caught two questions I'd graded too generously. See **Verdicts**
> below for what changed and why.

**Real output — criterion 1, a clean pass** (retrieval, `store.py::search`,
top result for "How much RAM do students recommend for CS courses?",
distance 0.199):

```
THREAD: How much laptop do I actually need for CS courses?

--- reply 1 (31 votes) ---
Less than the recommended spec page says. 16GB of RAM is the one number worth paying for; everything else you'll never notice.
```

**Real output — criterion 1, the two questions that don't actually pass**
(same retrieval, full source documents):

```
THREAD: When should I start looking for a summer internship?

--- reply 1 (30 votes) ---
Earlier than feels reasonable. Large employers close applications in October and November for the following summer.
```

My question asks *when applications open*; the only date in the corpus is
when large employers *close* theirs. `expects: "October"` matched the text,
but not because the corpus answers the question I asked.

```
THREAD: Worth getting a parking permit?

--- reply 1 (15 votes) ---
West lots sell out in about three days in August. East lot never sells out but it's a 12 minute walk, at which point you might as well have parked on the street.
```

My `expects` field says "West Campus." Read plainly, this sentence says the
opposite: West sells out (i.e. isn't available most of the year) and East
never does. If "available" means "you can still get one," this chunk is
better evidence for East than for West.

**Real output — criterion 2** (generation, `generate.py::answer_from_chunks`,
run 1 of the same question):

```
Students recommend 16GB of RAM for CS courses, noting that 8GB can become insufficient for later projects.

Source: thread_laptop_specs.txt
```

**Real output — criterion 3** (`gate.py::check` via
`run_eval.py::check_out_of_scope`):

```
refused  (best distance 0.807)  What is the recommended dosage of ibuprofen for a headache?
```

**Real output — criterion 4** — the five samples already pasted under
[Sample Chunks](#sample-chunks) above, produced by `chunker.py::split_documents`.

**Real output — criterion 5**, the one question that actually breaks this
one (`generate.py::answer_from_chunks`, "Which campus has parking passes
available?", run 1 — repeats in runs 2 and 3 with different wording, same
claim):

```
Based on the provided documents, there is no mention of which specific campus has parking passes available.

Source: `thread_parking.txt`
```

The chunk this was generated from (`store.py::search`, ranked #4 of 5,
distance 0.586) actually says: *"West lots sell out in about three days in
August. East lot never sells out but it's a 12 minute walk..."* — there is
relevant information right there, and (see Diagnoses) it's genuinely
ambiguous which side it actually supports. Either way, this is a refusal on
a question I declared answerable in criteria.md, which is what fails
criterion 5, not a claim of outright fabrication.

## Verdicts

My first pass through this table gave all five a MET. Milestone 2's own
advice is that this is exactly where people get it wrong, so before locking
anything in I did what it also suggests — asked for the opposite verdict,
argued as strongly as possible, for my two closest calls (criterion 1 and
criterion 5). I ran that as two independent reviews with no visibility into
each other or into my reasoning, each handed the real questions, the real
retrieved chunks, and the real generated answers, and told to argue for
MISSED as hard as it could, then give its own honest final call. The
criterion 1 review changed my mind; the criterion 5 one didn't, and said so.

| # | Criterion | Verdict | How I decided |
|---|---|---|---|
| 1 | Retrieved chunk contains the answer | **MISSED** | 3/5, not 5/5. This flipped after the adversarial review. On internships, my question asks when applications *open*; the only date the corpus gives is when large employers *close* theirs (Oct/Nov) and when smaller ones *hire* (Feb/March) — `expects: "October"` matched the chunk's text without the chunk actually answering what I asked. On parking, my `expects` field says "West Campus," but the chunk states "West lots sell out in about three days... East lot never sells out" — read plainly, that's evidence for East being the one with availability, not West. Both of these are questions I wrote from memory of the corpus rather than from its actual wording, and a plain reading doesn't let either one count as "the retrieved chunk contains the answer." That leaves laundry, RAM, and dean-of-students — 3 of 5, one short of the target. |
| 2 | Every answer names a source | MET | 5/5 in all three runs (15/15 total) — every one of the 15 generated answers, read individually, names a real filename, including the three "parking" answers that otherwise fail criterion 5. This one held even where the content was wrong. |
| 3 | Gate stops out-of-corpus questions | MET | 5/5, deterministic single pass. All five `OUT_OF_SCOPE` questions came back with best distance ≥ 0.807, comfortably over the 0.65 cutoff. |
| 4 | Sample chunks read as thread title + one complete reply | MET | 5/5 of the samples in this README's Sample Chunks section. I re-read all five against the target after writing the chunker, not just when I first pasted them, since it would have been easy to let a stale sample from the old chunker slip through — I had exactly that stale sample in an earlier draft and replaced it. |
| 5 | Every factual claim supported by its cited source | MET | 4/5 in every one of 3 runs, same question failing every time. The adversarial review pushed on this one too but didn't change the verdict, and I agree with why not: on internships, the model's answer says the source's applications *close* in Oct/Nov — that's exactly what the source says, so nothing false was claimed, even though my *question* asked about opening. That's a flaw in criterion 1, not this one. Parking is the real failure: given my own criterion's rule that a refusal counts as a failure on these five stipulated-answerable questions, "there is no mention of which campus has parking passes available" is a refusal, regardless of how genuinely ambiguous the underlying source turned out to be. 4/5 meets "at least 4 of 5" — MET — but see Diagnoses for why I don't think that's the end of the story. |

## Diagnoses

### Criterion 1 — MISSED (3/5)

Two questions fail, and it's the same mechanism both times, not two separate
ones: **internships** and **parking**.

- **Loading** — not the cause. Both source documents load clean, every
  reply intact.
- **Chunking** — my first guess, tested and ruled out. I hypothesized that
  `split_documents` putting the parking thread's two "west" mentions in
  separate, non-adjacent chunks (ranks #1 and #4 of 5, two unrelated chunks
  between them) was why the model couldn't use them. I tested this directly
  rather than assuming it — see **The Improvement** below — by re-chunking
  the whole corpus so every reply of a thread sits in one chunk together,
  and re-running the full test against that index. Both questions failed in
  exactly the same way, under exactly the same reasoning. Ruled out by
  measurement.
- **Embedding / Retrieval** — also not the cause, under both chunkings.
  Retrieval surfaces the right thread every time (internship timing for the
  internship question, parking for the parking question) at low distance
  (0.204–0.586). The relevant sentences are right there in what comes back.
- **Generation** — not the cause either, and this is the part that
  surprised me. On internships, the model doesn't hallucinate an opening
  date — every run correctly says applications *close* in October and
  November, which is exactly what the source says. On parking, even handed
  a *single* chunk with every "west" mention merged into one place (the
  whole-thread variant), the model still says there's no campus named,
  rather than guessing. In both cases the model is being accurate to what
  the documents actually contain.

The real mechanism sits upstream of all five stages: I wrote these two test
questions in `questions.py` (Unit 1, Milestone 2) from what I remembered or
assumed the corpus said, not from its actual wording, and didn't check either
one against real retrieval before finalizing them. "When do applications
open" has no answer in a corpus that only states *close* and *hire* dates.
"Which campus has parking passes available" asks for a noun ("campus") the
corpus never uses, and my own `expects: "West Campus"` turns out to read the
one relevant sentence backwards — "West lots sell out in about three days...
East lot never sells out" more directly supports East as the one with
availability. No amount of re-chunking, re-embedding, or re-prompting fixes
a question asking for something the source material doesn't contain, or
misreads what it does contain.

I only caught the parking half of this on my own. I found the internships
half by running the "argue the opposite verdict" check Milestone 2 itself
recommends — an independent review, blind to my reasoning, argued for
MISSED on both questions and I checked its claims against the source text
myself rather than taking them on faith. It held up on both. That's the
main lesson of this milestone: my own first pass at these two questions
looked fine to me on a casual read, and only came apart once something
independent was actively trying to break it.

### Criterion 5 — a MET I don't fully trust

Criterion 5 stayed at 4/5 (MET) through the same adversarial review — the
argument that internships' "close" vs. "open" mismatch should count against
it here too didn't hold up, because the model's claim (*"close in October
and November"*) really is what the source says; nothing false was asserted.
But the parking failure is still real, and criterion 5's "4 of 5" was loose
enough to absorb it without ever surfacing as a MISS.

**Are my targets set low?** For criterion 5, a little — it was written
before any evidence existed, and happened to be exactly loose enough to hide
a reproducible failure on the very first real measurement (same question,
all 3 runs, not noise). If I were tightening one criterion for the next
unit, it would be this one, to 5 of 5: these are five questions I already
claimed were answerable from my own corpus, so a single free miss isn't
headroom for a genuinely hard question, it's headroom for not noticing a
bug. Per the rule at the top of this file I'm not rewriting criteria.md to
say that — a target I met stays where it is — but I'm saying so here rather
than letting the MET verdict stand as the whole story.

## The Improvement

**What I changed:** Built a second index of the same corpus using
`chunker.py::fallback_split` — the original whole-thread, 800-character
chunker — stored under index variant `wholethread`, alongside the default
`split_documents` index rather than replacing it. Then ran the full test
again against that variant: `python run_eval.py --label after --variant
wholethread`. The committed default chunker, config, and all five README
Sample Chunks are untouched.

**Why I picked it:** My diagnosis hypothesized that `split_documents` putting
the parking thread's two relevant replies into separate, non-adjacent chunks
was why the model couldn't connect them into an answer. Whole-thread
chunking is the direct test of that hypothesis: it puts both "west" mentions
back into the same chunk, back to back, exactly as they were before I
changed the chunker in Unit 1. (I only later confirmed, via the adversarial
verdict check above, that chunking was never going to be the whole story —
but I'd already formed and was testing this hypothesis before that check
ran, so I'm reporting the experiment as I actually ran it.)

### Run Log — After

Produced the same way as Run Log — Before, index variant `wholethread`. Full
file: [`results/run_2026-09-29_2214_after.md`](results/run_2026-09-29_2214_after.md).

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 | 3/5 | 3/5 | 3/5 | MISSED |
| 2. Every answer names a source | 5 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 3. Gate stops out-of-corpus questions | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 4. Sample chunks read as thread title + one complete reply | 4 of 5 | 5/5 | 5/5 | 5/5 | MET (unaffected — this variant is a side experiment; the committed chunker and samples didn't change) |
| 5. Every factual claim supported by its cited source | 4 of 5 | 4/5 | 4/5 | 4/5 | MET |

**Real output — after** (`generate.py::answer_from_chunks`, "Which campus has
parking passes available?", run 2, whole-thread variant — the retrieved
chunk this time is the *entire* `thread_parking.txt`, all three replies
together, not split apart):

```
I don't have enough information to determine which campus has parking passes available, as the documents only mention West lots and East lots selling out in August without naming a specific campus (thread_parking.txt).
```

**Did it help?** No — and not ambiguously. Every criterion came back at
exactly the same count as before, question for question, run for run,
including criterion 1's 3/5 (the internships and parking questions are still
the two that don't hold up, for the same reasons — no chunking strategy
changes what a source document does or doesn't say). Distances shifted a
little (retrieval sees 26 bigger chunks instead of 75 small ones, so the
numbers move — RAM went from 0.199 to 0.246, for instance), but nothing
crossed the 0.65 gate either way, and the actual answers barely changed in
substance. The parking answer, under whole-thread chunking, is now *more*
explicit that the model saw the west/east lot content and still declined to
treat it as naming a "campus." That's a clean disproof of my chunking
hypothesis, not an ambiguous one: the evidence was structurally identical —
one chunk instead of two — and the model's behavior didn't move at all. The
real cause (two test questions that don't match what the corpus actually
says, diagnosed above) is outside what a different chunking strategy can
fix, and I'd rather report that plainly than dress this up as a win.

## What's Still Broken

**Criterion 1 (MISSED, 3/5)** — internships and parking. Both survive the
improvement unchanged, because the improvement was a pipeline change and
this isn't a pipeline problem:

- **What's wrong:** "When do applications open for summer internships?" asks
  for a date the corpus never states (it gives *close* and *hire* dates
  instead). "Which campus has parking passes available?" asks for a noun
  ("campus") the corpus never uses, and my own `expects` field ("West
  Campus") reads the one relevant sentence backwards relative to a plain
  reading of "sell out" vs. "never sells out."
- **What I'd do about it:** not touch the pipeline again — I tested the one
  pipeline-side hypothesis I had (chunking granularity) and disproved it
  directly. The actual fix is on the test side: reword each question and its
  `expects` field to match the corpus's own wording (ask when applications
  *close*, not *open*; ask which lot is *harder to get a permit for* rather
  than which "campus" has passes "available," and fix `expects` to "east" or
  reword to ask for the scarce one, "west"), or add documents that actually
  support the original framing if I want to keep testing it as originally
  written.
- **Why I stopped here:** rewriting `questions.py` now, after seeing both
  fail, would be indistinguishable from quietly moving the goalposts — the
  whole point of writing the criteria and questions in Unit 1 before any
  results existed was so a fix like that has to be visible as a fix, not
  folded invisibly into "how I always tested it." I'm leaving both broken
  and named, which is what this milestone asks for instead.

**Criterion 5 (MET, but see Diagnoses)** — the parking question is also this
criterion's one failure, for a related but distinct reason (a refusal on a
question I declared answerable, not the wrong retrieval target). Same
answer: I'd fix the question, not the pipeline, and for the same reason I'm
not doing it now.

## What I'd Do Differently

I'd write criterion 1 and criterion 5 both differently, but for different
reasons. Criterion 5's "4 of 5" was loose enough that a real, reproducible
bug never had to surface as a MISS — I'd tighten it to 5 of 5, since these
are five questions I already claimed were answerable from my own corpus.
Criterion 1 didn't need a different number; the problem there wasn't the
target, it was that I graded my own first pass too generously and it took an
adversarial second opinion to catch two flawed questions a casual read had
missed — mine, both in Unit 1 when I wrote them and again in Unit 2 when I
first scored them.

More than either number, I'd change how I *write* the five test questions in
the first place, not just how I score them after the fact. I wrote both
flawed questions in Unit 1 from what I remembered or assumed the corpus
said, not from its actual wording, and never ran `python app.py retrieve` on
either one until Unit 2 forced me to look. If I'd run that one command
before locking each question in, I'd have caught "open" vs. "close" and
"campus" vs. "lot" in Milestone 2 of Unit 1, instead of diagnosing them
after the fact in Unit 2 — and I'd build the "argue the opposite verdict"
check into that same step next time, rather than saving it for Milestone 2
of the *following* unit.
