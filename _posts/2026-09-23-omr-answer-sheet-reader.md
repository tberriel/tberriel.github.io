---
layout: distill
title: Reading a 210-question answer sheet from a phone photo
description: An Android app that reads hand-marked exam sheets whose marks change meaning as ink increases, fills a Moodle quiz with a human confirming every answer, and was built by two AI agents.
tags: computer-vision android agents
giscus_comments: false
date: 2026-09-23
featured: false

authors:
  - name: Tomás Berriel Martins
    url: "https://tberriel.github.io/"

toc:
  - name: Why build it
  - name: "The problem: ink that changes meaning"
  - name: From photo to answers
  - name: Human review as the guarantee
  - name: "Scoring: Moodle and local"
  - name: Built by agents
  - name: Lessons learned
---

## Why build it

My partner is preparing for one of the Spanish IR exams (MIR, BIR, RFIR and the rest), which
thousands of students sit every year. Preparation means mock exams: 210 multiple-choice
questions, four and a half hours, answered on a printed sheet. The prep academy scores them
through a Moodle quiz, so after every mock exam the 210 answers had to be typed in again, one
by one, on a phone. That took 20 to 30 minutes, right after four and a half hours of exam.

I built an Android app that removes that step. You photograph the answer sheet, the app reads
all 210 answers, you confirm every one of them on screen, and the app fills the Moodle quiz.
After you submit, it reads Moodle's corrections back and computes the score. For practice
exams outside Moodle, it scores the answers locally against a bundled answer key.

The whole process now takes about 5 minutes, and most of that is the review. The bigger
change is harder to measure: after a long exam, there is one less tedious thing to get
right.

The project had two goals. The first was the app. The second was personal: I used it as my
first fully agent-driven project, where AI agents wrote the code and my job was to design,
constrain and verify. This post covers the algorithm, the review process, the Moodle
integration, and how that development process worked. The code is in
[the public repository](#)<!-- TODO: public repo URL -->.

## The problem: ink that changes meaning

Each question has four boxes, one per option. Students mark them by hand, following a
convention set by the exam board:

| Cell state | Ink | Meaning |
|---|---|---|
| blank | none | not selected |
| cross (✕) | low | **selected** |
| filled square | high | cancelled, not selected |
| filled square with a ring around it | highest | **selected** again |

The ink increases from top to bottom, but the meaning alternates. This breaks the standard
optical mark recognition approach. Almost every OMR system decides whether a box is marked by
how dark it is: more ink means more selected. Here, a cancelled answer is the darkest box in
its row, so any intensity comparison picks it as the most confident answer. Intensity
thresholding is not only less accurate on this sheet, it gives inverted results.

So every decision has to be structural. Are there two crossing strokes? Is the box solidly
filled? Is there a ring drawn *outside* the box?

The hardest case is filled versus filled with a ring. Both are high-ink, they mean opposite
things, and they differ only by a thin ring in the space around the box. A crop that is a few
pixels too tight, or a registration error of a fraction of a row, removes that ring from the
image. The cell still looks like a clean, confident "filled", and the error is silent. For
that reason, every cell crop is deliberately wider than the box itself.

The other constraints came from the user:

- **Phone only.** My partner has an Android phone and a tablet, no laptop, and is not
  technical. Anything that needs a terminal or configuration is out.
- **Hand-held photos.** Different phones, resolutions, angles and lighting.
- **One attempt.** Each Moodle quiz allows a single submission, and Moodle autosaves. Any
  fill of the real form is written to the server within about a minute, so there is no safe
  way to try something out on it.
- **Very little data.** I started with 4 labelled sheets.

<!-- SCREENSHOT: the four cell states (blank, X, filled, filled+ring), annotated with meaning -> assets/img/blog/omr/cell_states.png -->
<p><strong>&lt;screenshot: the four cell states (blank, X, filled, filled+ring), annotated with meaning&gt;</strong></p>
{% comment %}
{% include figure.liquid loading="eager" path="assets/img/blog/omr/cell_states.png" class="img-fluid rounded z-depth-1" %}
{% endcomment %}
<div class="caption">The four cell states. Ink increases from left to right; the meaning alternates.</div>

## From photo to answers

The pipeline has four stages, each with a narrow interface:

```
photo → Registrar → GridExtractor → CellClassifier → AnswerResolver → 210 answers
```

Each stage uses only the previous stage's output. That keeps the sample-dependent tuning,
such as classifier thresholds and registration parameters, behind interfaces that can change
without touching the rest.

**Template.** The reference frame comes from a blank scan of the printed sheet. The box
outlines are detected with a contour-based shape filter and fitted to a parametric lattice per
block, with one row pitch and one column pitch. The only hand-written knowledge is which
x-range and which question range belong to each block; no individual cell position is
hard-coded. The printed lattice is very regular: the row pitch measures 50.000 px, consistent
within 0.06% across blocks. If a block's detections don't produce exactly the expected number
of rows, calibration fails loudly instead of shifting the grid.

**Registration.** The photo is aligned to the template in two stages. The first is SIFT
features, a ratio test and RANSAC. I tried ORB first, but it could not bridge the scale gap
between a 300 DPI scan and a phone photo with about a third of that pixel density. SIFT's
coarse match has a subtler problem: the answer grid is hundreds of identical boxes, so the ratio
test correctly discards most of it, and only the page header produces unique matches. The
homography is then fitted on a thin strip at the top and extrapolated over the grid, so the
error grows toward the bottom of the page.

The second stage fixes this by using the grid itself. It warps the photo with the current
estimate, re-detects box outlines across the whole sheet, pairs each detection with its
nearest lattice point, and refits on hundreds of correspondences. This repeats a few times
with a tightening distance gate. The residual drops from 0.19–0.31 of a row pitch to
0.05–0.07. If the refit never finds enough correspondences, registration keeps the coarse
estimate and reports that it did.

**Classification.** The classifier never looks at grayscale intensity. Pen ink on these
sheets is strongly blue, while the printed lines and text are neutral, so each crop is
white-balanced (gray-world) and turned into a blue-ink mask. Three structural questions follow:

- *Is there a mark?* The ink fraction inside the box, after eroding a thin band at the edge
  so the printed border can't pass as ink.
- *Cross or fill?* A cross follows the two diagonals, so the regions far from both diagonals
  stay mostly empty; a fill covers them. A second check on the box corners catches bold
  crosses. A cell is a fill only when both checks agree.
- *Fill or fill with a ring?* The area between the box and the widened crop's edge is split
  into angular wedges around the box centre. A real ring puts ink in a majority of wedges. Ink
  bleeding in from a neighbouring row only reaches one or two, which an earlier single
  ink-density measure could not tell apart.

Confidence is the margin to the nearest decision boundary, scaled by how much the two
neighbouring classes actually overlap there on real data. A confidence of 1 means "outside
the measured overlap", not just "far from zero".

**Resolution.** The `AnswerResolver` turns the four cell states of a question into an answer.
One selected cell gives that option. None gives `0`, which is correct whether the question
was left blank or every mark was cancelled. Two or more, or low confidence, gives `Ambiguous`.
It is a pure function with no image input and only 4⁴ = 256 possible inputs, so it is tested
exhaustively.

**Accuracy.** I split accuracy by the true answer, because `0` is rare: a system that never
outputs `0` still scores around 90% overall.

| Split | `0` correct | `1–4` correct | Ambiguous | Confidently wrong |
|---|---|---|---|---|
| Development (7 sheets) | 100% (110/110) | 91.6% (1142/1247) | 113 | 105 |
| Held-out (4 sheets) | 100% (44/44) | 98.3% (705/717) | 79 | 12 |

Cancellations and blanks are never confused with a real answer. Most of the development-set
errors come from one sheet marked in a pen colour the blue mask cannot see. That is an accepted
limitation, and the review step below exists for cases like it.

**On the phone.** The Python code is the readable reference implementation. The app runs a
Kotlin port on OpenCV's Java bindings, entirely on the device. The two are kept in agreement by
the same 256-case fixture for the resolver and a differential test that runs both pipelines on
real sheets and reports mismatch counts. When they disagree, the Python side is right by
definition, because every tuning decision was measured there.

<!-- SCREENSHOT: pipeline strip: photo -> registered -> grid -> per-cell states -> answers -> assets/img/blog/omr/pipeline.png -->
<p><strong>&lt;screenshot: pipeline strip: photo, registered sheet, grid overlay, per-cell states, answers&gt;</strong></p>
{% comment %}
{% include figure.liquid loading="lazy" path="assets/img/blog/omr/pipeline.png" class="img-fluid rounded z-depth-1" %}
{% endcomment %}
<div class="caption">From a hand-held photo to 210 answers.</div>

## Human review as the guarantee

With a handful of labelled sheets, there is no way to show that the pipeline is 100%
accurate, and in an exam every wrong answer costs points. So the guarantee does not come from
the model. It comes from a person.

The review screen shows all 210 questions in order, each reading next to the image of its own
marks. Saving is enabled only after scrolling through the whole list. There are no confidence
numbers and no technical terms; the screen uses the same vocabulary as the printed sheet. The
user checks the reading against the actual marks instead of trusting a label.

This changes what the classifier should optimize. The person looks at every question anyway,
so the cost is the total number of corrections: confidently wrong answers plus ambiguous ones.
Pushing the confidence threshold up to remove the last confidently wrong answers turns many
correct readings into ambiguous ones. On the corpus, the threshold that brought confident
errors to zero produced about five times more total corrections than the one I shipped.

One rule is absolute: `Ambiguous` never becomes `0`. `0` means "the student left it blank";
`Ambiguous` means "we could not read it". Merging them would fill the form, report 210 out of
210, and submit the least certain answers as blanks. The review screen cannot be left while any
question is still ambiguous.

The review also produces data. When a review is confirmed, the app saves the photo, the
pipeline's original prediction and the confirmed answers on the device, and a Share button
exports them. The difference between prediction and confirmation is a labelled record of
exactly where the pipeline failed, at no extra cost. That is how the corpus grew from 4 to 11
sheets: 7 for tuning and 4 held out, evaluated only once.

<!-- SCREENSHOT: the app's review screen, a few questions with their cell images -> assets/img/blog/omr/review.png -->
<p><strong>&lt;screenshot: the app's review screen, a few questions with their cell images&gt;</strong></p>
{% comment %}
{% include figure.liquid loading="lazy" path="assets/img/blog/omr/review.png" class="img-fluid rounded z-depth-1" %}
{% endcomment %}
<div class="caption">Every question is shown next to the image of its own marks.</div>

## Scoring: Moodle and local

**Filling the quiz.** The academy's Moodle quiz opens inside the app, in a WebView logged into
the user's own account. The code that touches the page is JavaScript injected into it, which
means the same file runs in a desktop browser against saved copies of the page. That is how
the fill logic was tested without a phone or the live exam.

It works in three calls. `probe` checks the page's shape before writing anything: questions
numbered from 1 to 210, each with a "clear my choice" option and a gap-free run of answer
options. `fill` sets the 210 answers and immediately reads every question back. `verify` reads
them back again later, after Moodle's autosave may have changed the page. The result is not a
boolean, because a silent partial fill across 210 questions looks exactly like success. It has
three states: *refused* (the page was not what we expected, and nothing was written),
*partial* (these specific questions did not take; finish them by hand), and *complete*.

Moodle has traps that a single test page would not catch. Each field's name contains an id
that changes on every attempt, so the code reads it from the page every time, and a second
test page that differs only in that number exists to catch a hard-coded value. A blank answer
is not an untouched question; it is an explicit click on Moodle's "clear my choice".

**The app never submits.** No code path submits the form, and a test checks the source to
enforce it. The app has buttons that press Moodle's own Finish and Submit buttons for the
user, but one human press is exactly one click on one named Moodle button, and nothing is
chained or triggered automatically.

**Reading the result.** After submission, Moodle shows each question as correct, wrong or
unanswered. The app reads only those marks, from element ids and class names, and computes the
score the exam uses: +3 per correct answer, −1 per wrong one, 0 per unanswered, over questions
1–200, plus the net score (a third of the raw score).

**Scoring locally.** For practice exams outside Moodle, the answer keys ship with the app. They
were extracted from the answer-key PDFs and checked by hand once, offline. The app applies
the same scoring, shows a correct/wrong breakdown for all 210 questions, and handles questions
annulled by the exam board by counting the reserve questions (201–210) in their place. Nothing
leaves the device.

<!-- SCREENSHOT: the fill/verify result after filling the Moodle quiz -> assets/img/blog/omr/fill_result.png -->
<p><strong>&lt;screenshot: the fill/verify result after filling the Moodle quiz&gt;</strong></p>
{% comment %}
{% include figure.liquid loading="lazy" path="assets/img/blog/omr/fill_result.png" class="img-fluid rounded z-depth-1" %}
{% endcomment %}
<div class="caption">The fill is verified by reading every answer back from the page.</div>

<!-- SCREENSHOT: the score screen (Moodle results or local scoring) -> assets/img/blog/omr/score.png -->
<p><strong>&lt;screenshot: the score screen (Moodle results or local scoring)&gt;</strong></p>
{% comment %}
{% include figure.liquid loading="lazy" path="assets/img/blog/omr/score.png" class="img-fluid rounded z-depth-1" %}
{% endcomment %}
<div class="caption">Score over questions 1–200: +3 correct, −1 wrong, 0 unanswered.</div>

## Built by agents

I saw this project as the right size for my first fully agent-driven build: a real user, a
clear definition of done, and failure modes that are easy to state and expensive to miss.
The agents wrote the code. My work was deciding what to build, and making sure what was
built was correct.

The setup used two Claude Code agents with different roles:

- **A planner, on my machine, working with me.** Together we wrote the design documents, split
  the work into milestones, and maintained a written operating contract for the developer: what
  the problem is, which interfaces are fixed, and which rules cannot be broken.
- **A developer, in a sandbox, fully autonomous.** It took a task, wrote its own plan, and
  implemented it test-first, using the [superpowers](https://github.com/obra/superpowers)
  workflow (brainstorm, written plan, TDD). It worked unsupervised and reported which checks
  it had run and which had failed.

This moved my job from writing code to writing constraints and checks that an unsupervised
agent cannot quietly get around. A few examples:

- **Stable interfaces.** The four pipeline stages and the three Moodle calls were fixed. The
  agent could change anything behind them.
- **A reference that cannot be edited to win.** The Kotlin port is checked against the Python
  code, and when they disagree the Kotlin changes. Otherwise a failing comparison would be
  "fixed" by making both sides wrong in the same way.
- **Human gates.** The sandbox has no phone, so anything that runs on the device or renders a
  screen was checked by me. The agent had to say what it built and what still needed the device,
  never that it worked.
- **Data rules.** The sheets are someone's real exam answers. Logs, reports and commits
  contain counts and question numbers, never answer values.
- **Decisions stay human.** Once, measurements showed that two changes to the Python reference
  had reduced confident errors at the cost of more total corrections. Reverting them was my
  call, made on the numbers, not the agent's.

The sandbox is what made this freedom safe. The agent ran in rootless Docker with outbound
network limited to an allowlist that excludes the academy's server. The one capture of the
logged-in exam page was hidden from the container; the agent only ever saw test pages derived
from it with every piece of real text replaced. It had no credentials and could not push: the
only way out of the sandbox was a `git push` I ran myself.

The project took about seven weeks and 270 commits, from an empty repository to a released
app that my partner uses on real mock exams.

## Lessons learned

1. **Find out what the signal means before choosing features.** Every OMR tutorial measures
   ink. Here the meaning is in the shape, and intensity would have produced confident, inverted
   answers.
2. **With little data, design for human review.** I could not prove 100% accuracy with 11
   sheets, but a person checking every answer can guarantee it. Once a human is in the loop,
   optimize their total effort, not a single error metric.
3. **Make the human step produce labels.** Every confirmed review is a labelled example. The
   review screen became the data pipeline.
4. **Verify by reading back.** "No error was thrown" is not evidence. A partial fill across 210
   fields looks like success until you re-read every one.
5. **With agents, the work is in constraints and verification.** The agent wrote the code
   well. The quality came from fixed interfaces, a reference it could not edit, and gates
   only a human could pass.
