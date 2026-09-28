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

<!-- Your five criteria, three runs each. `python run_eval.py --label before`
     runs the questions, puts the OUT_OF_SCOPE ones through the gate, and
     writes it all into results/ for you. Targets come from criteria.md; the
     verdict column is your call.

     Criterion 3 is measured in one deterministic pass rather than three, so
     the same number goes in all three run columns. That's correct, not lazy.

     Milestone 1. -->

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 |  |  |  |  |
| 2. Every answer names a source | 5 of 5 |  |  |  |  |
| 3. Gate stops out-of-corpus questions | 4 of 5 |  |  |  |  |
| 4. | | | | | |
| 5. | | | | | |

<!-- Underneath, paste the REAL output for each criterion from one of your
     runs — the actual text your system produced, not a description of it.
     Name the file and function that produced it. -->

## Verdicts

<!-- MET or MISSED for each of the five, against the target you wrote last
     unit — not a new one. Plus a sentence on how you decided. That sentence
     matters most where it was close.

     If your target said 4 of 5 and your runs came out 4, 3, 4, that's a MISS.
     The target has to hold, not show up occasionally.

     Milestone 2. -->

| # | Criterion | Verdict | How I decided |
|---|---|---|---|
| 1 |  |  |  |
| 2 |  |  |  |
| 3 |  |  |  |
| 4 |  |  |  |
| 5 |  |  |  |

## Diagnoses

<!-- For each miss: which stage caused it, and how. The stage alone isn't
     enough — you need the mechanism.

     Not a diagnosis: "Question 3 didn't work."
     A diagnosis:     "Question 3 asks about laundry costs. The answer is in
                       one sentence that got split across two chunks, so
                       neither chunk on its own contains it."

     The five stages: loading → chunking → embedding → retrieval → generation.

     Look for a pattern. If three misses all ask about numbers, that's one
     problem, not three.

     Missed nothing? Say so, then say honestly whether your targets were set
     low, and which one you'd tighten and to what.

     Milestone 3. -->

## The Improvement

**What I changed:**

**Why I picked it:**

<!-- Connect it to a specific diagnosis above in one sentence. If you can't,
     you picked a fix because it sounded impressive. -->

### Run Log — After

<!-- Same format, same five criteria, three runs each.
     `python run_eval.py --label after` -->

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 |  |  |  |  |
| 2. Every answer names a source | 5 of 5 |  |  |  |  |
| 3. Gate stops out-of-corpus questions | 4 of 5 |  |  |  |  |
| 4. | | | | | |
| 5. | | | | | |

**Did it help?**

<!-- Say plainly whether it did, and how you know. If it made things worse,
     say that — a change that backfired, honestly reported, earns full credit
     and is more interesting than one that worked. What matters is that you can
     tell.

     Milestone 4. -->

## What's Still Broken

<!-- For each criterion still missed after your fix: what you'd do about it,
     and why you stopped where you did.

     "I ran out of time" is fine if it's true. Pretending nothing is left is
     not.

     Milestone 5. -->

## What I'd Do Differently

<!-- Knowing what you know now — which of your five criteria would you write
     differently, and why?

     Milestone 5. -->
