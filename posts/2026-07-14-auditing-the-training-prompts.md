# Garbage in: auditing the training prompts before round 1

*2026-07-14*

[Round 0](2026-07-09-training-rig-round0.md) proved the training rig runs end to
end on one consumer GPU — fine-tune a small student on the big model's choices,
merge, convert, serve, score. Its closing line was that round 1 is where *quality*
is finally allowed to be the point.

So I started building the round-1 training set: take thousands of real in-game
decisions, replay each one to the strong teacher model, and record its pick as the
label the student will learn. Before spending a day of GPU time training on those
labels, I did the boring thing and **read a hundred of them.**

That was the whole round. The training never started. The prompts were lying.

## The teacher was fine. The questions were broken.

Here's the shape of the problem. To ask the teacher "what would you do here?", the
game has to be turned into text — a written description of the board, your hand,
your mana, and the menu of legal plays. The teacher reads that description and picks
an option. Its pick becomes the answer the student is trained to imitate.

I graded 100 of these prompts by hand against the actual game they came from.
**58.4% of them described a board state that didn't match the game.** Not subtly, in
several cases — attackers that had already died were still listed as attacking; the
same option appeared eleven times; internal placeholder tokens showed up in the
text raw.

The important part: **none of this was the teacher's fault, and none of it was a
rules bug.** The underlying game engine simulated everything correctly. The defect
was entirely in the layer that *describes* the game to the model — the equivalent of
handing a chess grandmaster a photo of the board with three pieces in the wrong
squares. Whatever they answer, you've learned nothing about chess. Train on enough
of those and you teach the student to reason confidently about positions that never
existed.

## What the audit found

Every defect fell into a handful of buckets, and each one was a display bug I could
point at a line of code and fix.

![Horizontal bar chart titled "A 100-row audit: the training prompts described games that never happened." Three bars show the share of round-1 training prompts carrying each defect: stale or phantom combat 55.7%, duplicate menu options 13.6%, and sentinel and predicate leaks 8.8%. A dashed vertical line marks that 58.4% of prompts had at least one defect; a footnote notes the categories overlap, so they sum past 58.4%.](assets/prompt-fidelity-audit.svg)

- **Stale / phantom combat (55.7%)** — by far the worst. Combat from a *previous*
  turn was still being described as if it were happening now: creatures shown
  attacking that had already dealt their damage and gone. One turn's fight bled into
  every later prompt. The engine cleared combat correctly between turns; the text
  layer was reading from a stale copy it never reset.
- **Duplicate menu options (13.6%)** — the same play listed several times as separate,
  identical choices. More on this one below, because the fix has a trap in it.
- **Sentinel & predicate leaks (8.8%)** — internal shorthand escaping into the prompt.
  Where the text should have said "creatures with power 2 or less," it printed the
  raw internal token `creatures_with_power_2_or_less`; where a spell on the stack had
  no special effect to describe, it was labelled "unknown effect" instead of, say,
  "a creature spell."

These overlap — one prompt often had several — which is why they add up to more than
the 58.4% of prompts with *any* defect.

## Two fixes worth explaining

Most of these were mechanical once found. Two are worth walking through, because the
obvious fix would have quietly made the training data *worse*.

### Duplicate options: enrich before you collapse

Picture the model being asked which token to sacrifice, and the menu lists the same
thing eleven times — every line reads `Blood (1/1)`. Whatever it picks, I record the
slot number as the "right answer." But slot 7 and slot 11 are identical, so that
answer is pure noise. Multiply across the corpus and the student is being trained on
a coin-flip dressed up as a decision.

The obvious fix: merge lines that look the same. But merge too eagerly and you delete
a *real* choice. If one of those tokens has a +1/+1 counter on it, sacrificing that
one is genuinely different from sacrificing a vanilla one — and by silently collapsing
them, I'd be making the decision *for* the model instead of fixing the prompt. That's
the cardinal sin here: never quietly pre-decide something the model is supposed to
weigh.

So the fix does it in two steps. First **enrich** every line so it fully describes the
token — power and toughness, counters, anything attached, whether it's tapped. Then
**collapse** only the lines that are now, after enrichment, genuinely identical. Two
tokens merge if and only if they truly are interchangeable; the buffed one keeps its
own line. Eleven identical `Blood (1/1)` lines become one, `×11` — but a
`Blood (2/2)` with a counter stays separate, exactly as it should.

### Mana: eight mana isn't always eight mana

The prompt told the model "you have 8 mana available." True — and useless. Six of
that came from Treasures, and spending a Treasure's mana *destroys the Treasure*.
Eight mana where six of it burns down part of your own board is a completely
different hand from eight lands you keep. The number was correct and told the model
nothing about the cost of using it.

Now the same line reads: `8 total; 2 recurring + 6 one-shot from sacrificing 6
Treasures`. Same total — I didn't invent a second, competing mana count that could
disagree with what the game actually lets you cast — just split into the part that
comes back next turn and the part you're setting fire to. Same idea for the board
itself: six separate `Treasure` bullet points now collapse to `Treasure ×6`, so the
real permanents aren't buried under a pile of identical tokens.

## The rule underneath all of them

Every one of these fixes obeys the same line, learned the hard way on this project:
**a prompt states facts; it never gives advice.**

I know it's the hard way because I once crossed it. An earlier change added a helpful
little nudge to the prompt — language suggesting the model *should spend* its
expendable tokens. It backfired badly: on one class of decision, the model's sensible
"yes" rate collapsed from 46% to 19%. Telling the model what to do made it play
worse, and the change was reverted. The lesson stuck. So the mana fix shows the split
and stops — it does not say "you should hold these" or "sacrifice these now." That
judgment is the model's whole job. My job is to make sure the board it's judging is
the real one, described completely, and then get out of the way.

## What this costs, and the guardrail that comes out of it

The honest tally: round 1 doesn't get to be about quality yet. A day of GPU time and
a spending window of teacher calls were about to go into imitating labels drawn from
lying prompts. Catching it cost an afternoon of reading; not catching it would have
cost the entire round and left me puzzling over why a "successful" training run
produced a worse player.

The durable win is that "read a hundred by hand" becomes a permanent, automatic
check. Each of these defects is mechanical to detect — stale combat outside the
combat phase, a menu with identical lines, a raw internal token in the text — so
they fold into a cheap deterministic linter that refuses to bless a training corpus
until the correctness-level defects read zero. No model, no cost, run before every
build. The audit that saved this round becomes the gate that guards every future one.

Then round 1 can finally be about whether the student plays *well* — on data that,
this time, describes the game that was actually played.

---

*Setup: a strong hosted model as the teacher and a small local model as the student,
scored through a from-scratch Magic rules engine on a single consumer GPU; the same
four decks as before —
[Arabella](https://archidekt.com/decks/16059609/dolls_kill),
[Benton](https://archidekt.com/decks/7599772/sergeant_john_instant_combat_tricks_benton),
[Ghyrson](https://archidekt.com/decks/6396937/ghyrson_starn_slingin_pingin), and
[Vihaan](https://archidekt.com/decks/16574615/treasures_attack_golden_sacrifice_edition).*
