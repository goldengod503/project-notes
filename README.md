# Project Notes

Write-ups and findings from my side projects. The code lives in its own
(mostly private) repos — these are the notes.

## Posts

- **[Garbage in: auditing the training prompts before round 1](posts/2026-07-14-auditing-the-training-prompts.md)** — 2026-07-14
  Before training round 1 on the teacher's choices, I read a hundred of the prompts
  by hand — and 58.4% described a board that didn't match the game. Not a rules bug:
  the engine was right, the layer that *describes* the game to the model was wrong.
  The buckets (a whole prior turn's combat bleeding forward, identical menu options,
  eight mana that's really six-that-burn-your-board), why two of the fixes had a trap
  in them, the one rule underneath all of them — a prompt states facts, never advice —
  and how a hand audit becomes a permanent gate.

- **[Training a smaller pilot, round 0: prove the rig before you chase quality](posts/2026-07-09-training-rig-round0.md)** — 2026-07-09
  Round 0 of fine-tuning a small 4B student to imitate the 14B teacher from the
  post below. Not about quality — about proving the whole rig runs end to end on
  one consumer GPU: QLoRA fine-tune, merge, convert, serve, score. The one honest
  metric (order-consistency) jumped 0.53→0.80, the toolchain paper-cuts it
  surfaced, and a pre-registered moving bar for whether round 1 actually helps.

- **[Can a small local LLM actually *pilot* a game?](posts/2026-07-07-measuring-small-llm-pilot.md)** — 2026-07-07
  Measuring whether a 14B model on a single consumer GPU can pilot a four-player
  game of Magic: The Gathering. Why win-rate fails in multiplayer, a
  counterbalanced relative-race method that measures the model against a
  benchmark one decision at a time, and two findings about how small models
  actually decide — anchoring to menu position instead of reading the board, and
  decisions too degenerate to measure at all.
