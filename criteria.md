# Acceptance criteria — The Unofficial Guide

Five criteria that say what "working" means for this system, written in unit 1
**before** any results existed.

An acceptance criterion names a target: a number, a count, a rate, or something
a person could plainly observe. *"Retrieval works"* is an opinion. *"For at
least 4 of my 5 test questions, the top results include a chunk containing the
answer"* is a criterion.

Under each one, write a sentence or two on **why that target** and not a
stricter or looser one. A reason that says something about your corpus or your
pipeline earns credit; *"80% seemed reasonable"* does not.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

---

## 1. Retrieved chunks contain the answer

For at least 4 of my 5 test questions, the retrieved chunks include one that
contains the answer.

**Why this target:**

My "parking passes" question is the one I expect to miss sometimes. The
answer ("west campus") isn't stated once — it's spread across three replies
in `thread_parking.txt` that partly disagree with each other, and when I ran
retrieval on it the best distance (0.536) was the worst of my five in-scope
questions, right up against the others in the ranking. Four of five leaves
room for that one to come back without the deciding reply while still
holding the other four, which are each answered in a single reply, to a
higher bar.

---

## 2. Every answer names a source

Every answer the system produces names at least one source document.

**Why this target:**

This one isn't down to the model remembering to behave — it's built into the
code path. `generate.py::build_prompt` labels every chunk in the prompt with
`[from <filename>]`, and `app.py::ask_pipeline` sets `outcome["sources"]` from
the retrieved chunks' own filenames, not from parsing the model's text. As
long as a question passes the gate at all, there are retrieved chunks with
real filenames attached, so a source is always available to report. All five
and not four, because the only way this fails is the gate passing on zero
results, which `gate.check` already treats as a refusal.

---

## 3. The relevance gate stops out-of-corpus questions

When I ask a question my documents clearly don't cover, the relevance gate
stops it and the system returns "I don't have enough information about that" —
in at least 4 of 5 tries.

<!-- The five questions are the ones in `OUT_OF_SCOPE` at the bottom of
     `questions.py`, and `run_eval.py` puts them through the gate and writes
     what happened into your run log. Swap them for your own if you'd rather —
     just keep five of them, or the "4 of 5" above has nothing to be 4 of. -->

**Why this target:**

I ran my five in-scope questions and the five `OUT_OF_SCOPE` ones through
`python app.py retrieve` and recorded the best distance for each. In-scope
came back 0.199–0.567; out-of-scope came back 0.807–0.896. That's a clean
gap with nothing from either group inside it, so I didn't need to split the
difference — I put the cutoff at 0.65, close to the in-scope side rather
than the middle (0.687) of the gap. I picked the low end on purpose: a
question that's borderline-relevant should still get an answer instead of a
refusal, and the closest out-of-scope distance I saw (0.807) still leaves
0.157 of headroom above 0.65, so I'm not trading away real protection against
out-of-corpus questions to get that margin.

---

## 4. Something about your chunks

<!-- YOU WRITE THIS ONE.

     How would you know if your chunks were the right size? Name something
     countable or observable.

     Examples of the right shape — don't copy these, they should come from
     what you actually saw in Milestone 3:
       - "At least 4 of 5 sampled chunks read as a complete thought, with no
          sentence cut in half at either end."
       - "No chunk is shorter than 200 characters, since anything below that
          in my corpus turned out to be a heading with no content under it." -->

Out of the five samplechinks recorded in my README, 4 out of 5 must inlvude their threwad title and at least one complete reply



**Why this target:**
some replies are cut off and only make sense when paired with the threads question


---

## 5. Your choice

<!-- YOU WRITE THIS ONE TOO.

     Pick something you actually care about getting right. It could be about
     speed, about refusals, about a particular kind of question your corpus
     handles badly, about source attribution being correct rather than merely
     present — anything, as long as it names a number or an observable
     outcome. -->

For at least 4 of my 5 test questions, the system must produce an answer in which every factual claim is supported by the source document cited for that claim. A refusal counts as a failure for this criterion because these five questions are answerable from my corpus.



**Why this target:**

I want to check that answers preserve the meaning of the sources instead of adding unsupported details. Requiring 4 of 5 allows one imperfect answer while demanding supported answers for most questions.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 2 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath it, like this:

         ## 1. Retrieved chunks contain the answer

         For at least 4 of my 5 test questions, the retrieved chunks include
         one that contains the answer.

         **Why this target:** ...

         > **Revised in unit 2:** For at least 4 of 5 questions, the top three
         > results contain the answer.
         >
         > **Why revised:** I couldn't judge "the chunks include one that
         > contains the answer" the same way twice — I scored two questions
         > differently on Monday than on Wednesday. The new version is
         > something I can actually check.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "I said 4 of 5 but got 2 of 5, so 2 of 5 is more realistic."

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.

     The whole reason the originals stay visible is so someone can see what you
     said before you knew the answer.
     ───────────────────────────────────────────────────────────────────────── -->
