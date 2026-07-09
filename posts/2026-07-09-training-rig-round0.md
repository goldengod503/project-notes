# Training a smaller pilot, round 0: prove the rig before you chase quality

*2026-07-09*

[Last time](2026-07-07-measuring-small-llm-pilot.md) I measured whether a small
local model (Qwen3-14B) could *pilot* a four-player game of Magic, and the finding
that stuck was Finding 2: small models anchor to a menu's *position*, not its
contents. Turning the model's reasoning on halved the anchoring but only lifted
decision-consistency to ~45% — it killed the bias without buying judgment. And it
cost 5× the wall-clock: ~24 hours per 100 games on one consumer GPU.

That combination — a 14B that's *okay* but *slow* — is the setup for an obvious
question. What if you took a **smaller** model, fine-tuned it on the 14B's own
choices, and got something that plays similarly but runs cheaper? A 4B student
serves in a fraction of the time and leaves VRAM to spare.

This post is **round 0** of trying. And round 0 is deliberately not about whether
the student plays *well*.

## What round 0 is — and isn't

The trap with a new training pipeline is measuring quality before you've proven the
plumbing. So round 0 has exactly one success criterion: **every stage runs and
produces a number.** Take the 14B's recorded decisions, fine-tune a 4B on them,
merge the weights, convert the format, serve it locally, and score it — end to end,
on one machine, without a stage silently faceplanting.

Quality movement was **explicitly not the goal.** If the student happens to improve,
that's a free signal, not the deliverable. The deliverable is a rig I can turn a
crank on.

> **The hardware, again.** Same desktop as last time — one NVIDIA RTX 3080 Ti with
> **12 GB of VRAM**, 8 CPU cores, 32 GB RAM, Pop!_OS. No cloud. The 12 GB ceiling
> adds a wrinkle that shapes the whole rig: **you can't train while you serve.**
> Fine-tuning wants the GPU; the served model already lives there. So the training
> script refuses to start if a model is resident in VRAM — training and serving
> take turns on the same card. Everything below happened on hardware someone
> actually owns, not a rented cluster.

---

## The pipeline

Seven stages, each handing its output to the next:

1. **Format** — turn 134 recorded decisions into training rows. Each row's target is
   just the *number* the 14B picked from that decision's menu. (0 rows dropped.)
2. **Train** — QLoRA fine-tune of the 4B: 4-bit base, small low-rank adapters, 3
   epochs, ~16 minutes.
3. **Merge** — fold the adapters back into a full-size model on the CPU (the GPU is
   busy elsewhere).
4. **Convert** — repackage that model into the quantized format the local runtime
   loads (a 4.3 GB file).
5. **Serve** — register it with the local model server.
6. **Smoke-test** — hand it three real menus and confirm it answers with an
   in-range number, not garbage.
7. **Evaluate** — score it the same way I scored the 14B last time.

All seven ran. That's the round-0 win, full stop. The interesting friction was in
stages 4 and 5, and I'll get to it.

## How the teacher "grades" — and why that's a caveat, not a triumph

There is **no correctness oracle** here. Nobody labeled the right play for each
decision. The student's training target is simply *what the 14B picked* — and to
make that pick less noisy, the 14B was asked five times with the menu reshuffled,
and its **majority vote** became the target. The rows were used unfiltered; no
judge vetted them.

That matters for reading every number below. The student isn't learning to play
*correctly*. It's learning to **imitate the 14B**. So "better" in round 0 can only
mean "more like the teacher" — which is exactly as strong as the teacher, and no
stronger.

---

## The numbers

Same evaluation as last post: take 60 held-out decisions the student never trained
on, ask each five times with the menu shuffled, and look at how it behaves.

| Metric | base 4B | student r0 | want |
|---|--:|--:|:--:|
| **Order-consistency** (same pick after a reshuffle) | 0.526 | **0.801** | ↑ |
| Teacher-agreement (matches the 14B's pick) | 0.333 | **0.533** | ↑ |
| First-slot rate (how often it grabs slot 1) | 0.383 | **0.268** | ↓ |
| Fallback rate (unparseable / out-of-range) | 0.130 | **0.093** | ↓ |
| Latency per decision (ms) | 10785 | **9621** | ↓ |

![Grouped bar chart comparing base qwen3:4b to the round-0 student on two 0-to-1 metrics. Order-consistency rises from 0.526 to 0.801 with a dashed round-1 gate line at 0.851 above it; teacher-agreement rises from 0.333 to 0.533 and is labelled circular, read with care.](assets/round0-student-vs-base.svg)

The student improved on **all five** — more consistent under reshuffling, less
slot-1 anchoring, fewer garbage answers, and faster. Encouraging. But remember what
round 0 was for, and read these honestly:

- **Order-consistency (0.526 → 0.801) is the one to trust.** No training target
  teaches it — the student is never told "pick the same thing when the menu moves."
  It's a pure by-product, so a jump here is a real behavioral change, not memorized
  imitation. This is the direct descendant of last post's Finding 2, and it's the
  metric the round-1 gate is built on.
- **Teacher-agreement (0.333 → 0.533) is partly circular.** The student was *trained*
  to match the 14B, then *measured* against the 14B. Of course it went up. I keep it
  as a descriptive number, not a quality claim, until I have real labels to score
  against.
- The other three (slot-1, fallback, latency) all moved the right way, but they're
  secondary — a well-behaved *format*, not necessarily a good *player*.

---

## What round 0 was actually for: the friction

The real yield of a round-0 run is the list of things that break against your
specific environment. Two were worth writing down.

- **The model server runs in a container, and it can't see your files.** The convert
  step writes a model file on the host; the server lives in a container with the host
  directory unmounted. So "just point the server at the file" doesn't work — the file
  has to be copied *into* the container first, and the registration recipe has to
  reference the in-container path. Obvious in hindsight; a dead stop the first time.
- **A missing tokenizer library aborted the conversion in a confusing way.** The
  converter tries to load one style of tokenizer, and when that library isn't
  installed it raises the *wrong kind* of error — one the converter's fallback path
  doesn't catch — so instead of quietly falling back it just dies. The fix was one
  added dependency, but the failure mode (a "file not found" fallback defeated by a
  "module not found" error) is the sort of thing you only find by running it.

Neither is profound. That's the point: round 0 exists to surface exactly this class
of paper-cut before you've invested in quality work that the plumbing can't deliver.

## The gate for round 1

Round 0 ends by setting the bar the *next* round has to clear, so "did it help?"
has a pre-registered answer instead of a post-hoc story:

- **Primary — order-consistency ≥ 0.851.** That's the round-0 student's 0.801 plus a
  fixed step. It's a **moving bar**: each round's gate is the previous student's score
  plus the same step, so a round only counts as progress if it beats the last one on
  the honest signal — not the base, and not by the circular metric.
- **Guard — fallback rate must not regress** past the round-0 student's 0.093. A round
  can't "win" by getting sloppier at simply producing a valid answer.
- **Teacher-agreement stays descriptive** until there's a real correctness label to
  score against. When that exists, it becomes the primary gate and the circularity
  goes away.

---

## What I'm *not* claiming

- **Not** that the student is a good Magic player. It imitates a 14B that itself only
  *matches a heuristic* on the surfaces measured so far. Imitating parity is, at best,
  parity.
- **Not** that the metrics prove quality. Round 0 proves a *pipeline*. Four of the
  five numbers could move for reasons that have nothing to do with playing better.
- **Not** a fair test of the student's real behavior. It was trained with reasoning
  off but evaluated with reasoning on (the same setting for both models, so the
  comparison is fair — but the student wasn't trained on the distribution it's served
  in). Round 1's job is partly to fix that.

## What this is

A training rig that runs end to end on one consumer GPU — fine-tune, merge, convert,
serve, score — proven on a throwaway round whose only job was to run. It surfaced two
real environment paper-cuts, produced a full set of baseline numbers, and left behind
a pre-registered, self-raising bar for whether the *next* round actually helps.

The next round is where quality is allowed to be the point. This one just had to
prove I could turn the crank.

---

*Setup: Qwen3-14B as the teacher and a QLoRA-fine-tuned Qwen3-4B as the student,
both served locally through Ollama on a single consumer GPU; a from-scratch rules
engine; the same four decks as
[last time](2026-07-07-measuring-small-llm-pilot.md) —
[Arabella](https://archidekt.com/decks/16059609/dolls_kill),
[Benton](https://archidekt.com/decks/7599772/sergeant_john_instant_combat_tricks_benton),
[Ghyrson](https://archidekt.com/decks/6396937/ghyrson_starn_slingin_pingin), and
[Vihaan](https://archidekt.com/decks/16574615/treasures_attack_golden_sacrifice_edition).*
