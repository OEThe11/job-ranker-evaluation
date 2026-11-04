# Building an LLM Job Ranker, and Testing Whether It Beats Keyword Rules

**A scoring layer for a job-board poller**
Osayuki Ehigie · November 2026

---

## What I was building

I have a poller that hits 133 company job boards across three ATS APIs and writes new postings to a CSV. It works. The first run returned 309 rows, and every run after that returns only what's new: 25 rows on the one later day I kept. Each row still has to be checked against the same things: title, location, pay, seniority. I wanted those checks written down and applied the same way every time.

So I built a ranker on top of it. An LLM reads each row and returns APPLY, MAYBE, or SKIP against written criteria.

The criteria file carries everything I'd otherwise hold in my head. Tiers of job titles I want, hard gates that disqualify a posting outright, skip flags that mean a human has to look, and defaults for what happens when none of it applies.

Then the real question. Is the model any good at it, and is it better than the keyword rules the poller already had?

---

## Writing the criteria exposed contradictions in my own judgment

The criteria file went through six versions before the first real run. Almost none of that was polish.

I labeled 20 real postings by hand, before any prompt existed. Labeling is what found the problems.

**Three rows with the same shape got three different verdicts.** Each one was a tier-matching title with no pay posted. I called one MAYBE and two APPLY. I had not decided whether missing pay should override a title I already wanted, and I didn't know that until I saw the three sitting next to each other.

**The same role family got treated three ways.** Forward Deployed Engineer showed up as a tier-two title in one row, as no tier at all in another, and as a MAYBE citing a tier that should have been an automatic apply.

**A rule I added contradicted a rule I already had.** I wrote a skip flag sending senior titles to a human read, and it collided with a hard gate that sent some senior titles straight to SKIP. Both can't win.

**Two rows disagreed about what a word in front of a title means.** "Strategic Implementation Manager" I called no-tier. "Senior Product Operations Manager" I called a tier match with a flag. Both are a title I want with a word in front of it. Either a modifier breaks the match or it doesn't.

None of that turned up from reading the file. All of it turned up from labeling real rows.

---

## The test set can only contain what the pipeline can actually see

I wrote six edge cases early on, and four of them depended on pay.

The poller had no pay field. It had no description field either.

So four of six cases tested the ranker on data it would never receive. The rules that depended on pay could never fire, and the rules that depended on job description text could never fire, because the ranker had no job description.

This is the same constraint that shaped the Subscout test set. There, every piece of evidence had to be something a bank could actually see. Here, every test row can only contain what the poller actually outputs.

Adding those two fields turned up two real bugs in the poller itself.

**Lever descriptions were being truncated.** The code only read the opening blurb and ignored the section where the pay range actually lives.

**An HTML entity was never decoded.** Greenhouse and Ashby write pay ranges with an em dash entity in the middle. The decoder handled six entities and that wasn't one of them, so every one of those ranges was unreadable.

Fixing both took pay detection across the full 9,086-posting corpus from 18% to 32%.

Still only 32%. Most postings don't state pay at all, which turned out to matter more than the fix did. It means the ranker decides most rows on title, location, and description, and the pay gates almost never fire.

---

## The results

Test set: 20 hand-labeled postings pulled from a frozen 274-row corpus. One call per row. gemini-3.8-flash, thinking level medium.

**19 of 20 verdicts correct.**

**20 of 20 parsed cleanly, with forced-JSON mode off.** I left Gemini's structured-output mode deliberately switched off so the baseline would measure whether the prompt alone produces valid JSON. It did, on every row. No code fences, no preamble. Schema enforcement is available and I don't currently need it.

Rows are my label, columns are the model's verdict.

| | APPLY | MAYBE | SKIP |
|---|---|---|---|
| **APPLY** | 9 | 0 | 0 |
| **MAYBE** | 0 | 7 | 0 |
| **SKIP** | 0 | 1 | 3 |

I ran the same 20 rows three times: October 7 and twice on November 3, with the same model, prompt, and criteria. All three runs scored 19 of 20 and parsed 20 of 20, and no verdict changed on any row. The reasoning did move: row 259 cited V1 and V2 in two runs and V2 alone in the third, with the same verdict each time. I compared verdicts on all 20 rows and reasoning only on that one. Three runs is a small sample, and these rows mostly resolve on a title plus a gate or two, so a harder set may move more.

---

## The keyword baseline: 9 of 20

This is the number that makes the other one mean something.

The poller already had its own sorting logic. Keyword flags for things to avoid, and a positive-match column for things to prefer. I scored those flags against the same 20 labels.

**9 of 20.**

| | APPLY | MAYBE | SKIP |
|---|---|---|---|
| **APPLY** | 6 | 0 | 3 |
| **MAYBE** | 4 | 0 | 3 |
| **SKIP** | 1 | 0 | 3 |

Look at the MAYBE column. It's empty.

The keyword rules never produce a middle verdict, because every row in the set matched the positive keyword "ai." The seven rows I wanted a human to look at got split between apply and skip, and none of them got flagged for review.

The other two failures are more specific.

**A data-tooling flag skipped three jobs I'd apply to.** The keyword matched somewhere in the description, the row got flagged, and the flag outranked a title I actively want.

**Gate failures are invisible to it.** One row pays below my hourly floor. It carries no flag at all, because nothing in its text looks bad. The keyword rules read it as a clean apply.

That's the honest case for the LLM here. Keyword matching has no way to signify that a row needs human oversight, no way to weigh a flag against a title, and no way to check a number against a floor.

### Checking that I didn't rig the baseline

I picked one reading of the flags: any flag means SKIP. A fair objection is that I chose the reading that makes keyword matching look worst.

So I ran the obvious alternative. Nine rows carried a flag, labeled 3 APPLY, 3 MAYBE, 3 SKIP. Eleven rows were clean, labeled 6 APPLY, 4 MAYBE, 1 SKIP. Under my mapping, flagged rows go SKIP and 3 land correct, clean rows go APPLY and 6 land correct. Under the alternative, where any flag means a human review, flagged rows go MAYBE and 3 land correct, clean rows still give 6.

Both mappings score 9. The result doesn't depend on which one I chose.

There's a blunter version. My 20 labels break down as 9 APPLY, 7 MAYBE, 4 SKIP. A system that answers APPLY to everything, with no logic at all, scores 9 of 20.

**The keyword flags perform exactly at the level of guessing the most common answer.** On this test set they carry no information. I'd want more than 20 rows before claiming that holds generally, and it is the number I have.

---

## The one mismatch was a bug in my rule

One row disagreed. A Trust and Safety role at OpenAI, San Francisco, no pay posted. I labeled it SKIP because San Francisco is not a city I can work in. The model said MAYBE.

My geography gate reads: "requires onsite or hybrid outside of NYC or JC commuting range."

A posting that just says "San Francisco" never states that it requires onsite attendance. So the gate couldn't fire. The model fell through to the defaults for an unknown title with no pay, landed on MAYBE, and was right to.

I wrote a rule that described how a job is worded instead of describing where it is. Most postings list a city and say nothing explicit about onsite, so this is not a one-row problem.

**When the model and the test set disagree, sometimes the rule is what's wrong.** Same lesson as the Babbel case in the Subscout evaluation, arriving from a different direction.

---

## What 19 of 20 does not mean

It doesn't mean the ranker is finished.

My own rule from the last evaluation says a baseline above 90% usually means the test cases were too easy. That applies here, with one qualification: the keyword comparison at 9 of 20 proves the task isn't trivial. Something in these 20 rows is hard enough that simple rules fail on half of them.

But 19 of 20 leaves no headroom. The whole point of a test set is to find a failure category, make one change aimed at it, and re-run everything to see if the number moved. With one error left, and that error being a rule bug rather than a model error, there is nothing to iterate on.

Two reasons the set is easy for the LLM specifically.

**The labels and the prompt came from the same source.** I wrote the criteria, then labeled rows against the criteria, then gave the model the criteria. That measures whether a model can follow written rules. It doesn't measure judgment.

**I never built the adversarial rows.** They were in the plan and I skipped them. A title I want buried inside a longer title. A banned word sitting next to a wanted one. A pay range straddling the floor. A seniority word on a gated title versus a wanted one. Those are the rows that would separate a model that reads carefully from one that pattern-matches.

So the next version of this is a harder test set. I expect the number to drop. A lower score on a harder set is a more useful artifact than a high score on an easy one.

---

## Caveats

**The keyword flags were built to sort but not to classify.** Reading them as verdicts was my choice, and I checked the obvious alternative reading above. Both score 9, as does answering APPLY to everything. What I have not ruled out is some third mapping of those same flags that does better.

**35 postings are missing from the corpus.** The test set is frozen from a cohort of 309 postings, but 33 had been taken down by the time I exported it and 2 were retitled. Postings that disappear fast probably skew toward roles that filled fast, so the frozen set isn't a neutral sample.

**20 rows is small.** One case is five percentage points. I'm reporting raw counts rather than percentages for that reason.

**The pay parser has at least one known false negative.** I spot-checked three rows against their live postings, and one row showing no pay in the CSV had a salary visible on the page. So the 32% detection rate is a floor.

---

## The part I'm least comfortable with

I wrote the criteria. I labeled the rows against the criteria. Then I handed the model the criteria and scored whether it matched my labels.

That setup can only measure one thing. Can a model apply written rules. It says nothing about whether the rules are any good.

A ranker that flawlessly implements bad criteria scores 20 of 20 here. If my hourly floor is set wrong, if a tier is missing a title I should want, if a gate is screening out the exact jobs that would have replied, this evaluation reports a perfect score and I learn nothing. The one thing it did catch, the geography gate, surfaced only because the model disagreed with me, and that was luck rather than design.

Breaking that circle takes outcome data. Which APPLY rows turned into replies, and which turned into nothing. Whether a SKIP row was one I should have sent. That would test the criteria instead of testing the model against the criteria.

None of that exists yet. The honest description of what I have is a ranker that reproduces my stated judgment at 19 of 20, on a judgment nothing has validated.

## What I'd tell someone building something similar

**Label real rows before you write the prompt.** Every contradiction in my criteria came from labeling, and none of it came from reading the file. Writing judgment down as rules makes it look consistent. Applying it to twenty real cases is what tests that.

**Build the baseline you have to beat.** I almost skipped this. A model scoring 19 of 20 sounds good until you ask what a dumb version scores, and 9 of 20 is the number that turns the result into evidence.

**The test set can only hold data the pipeline produces.** Four of my first six cases were unusable because they tested fields that didn't exist.

**Check the error before blaming the model.** My single mismatch was a rule that described wording when it should have described location.

**Hide the existing signals from the model.** The poller's own flags stayed out of what the ranker sees. If the model can see them, you can't tell whether it reasoned or copied, and you lose the comparison.

---

## Summary

| | |
|---|---|
| Source | 133 job boards, three ATS APIs, 309 rows on the first run, then new rows only |
| Test set | 20 hand-labeled postings from a frozen 274-row corpus |
| Criteria versions before first run | 6 |
| LLM verdicts correct | 19 of 20 |
| Keyword baseline correct | 9 of 20 |
| Parse rate, forced-JSON mode off | 20 of 20 |
| Runs | 3, no verdict changed on any row |
| Corpus pay detection, before and after | 18% to 32% |

Model: gemini-3.8-flash, thinking level medium. Harness in Node. The poller and criteria are a private repository. This document covers the evaluation approach and findings.
