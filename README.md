# Project Notes

Write-ups and findings from my side projects. The code lives in its own
(mostly private) repos — these are the notes.

## Posts

- **[Can a small local LLM actually *pilot* a game?](posts/2026-07-07-measuring-small-llm-pilot.md)** — 2026-07-07
  Measuring whether a 14B model on a single consumer GPU can pilot a four-player
  game of Magic: The Gathering. Why win-rate fails in multiplayer, a
  counterbalanced relative-race method that measures the model against a
  benchmark one decision at a time, and two findings about how small models
  actually decide — anchoring to menu position instead of reading the board, and
  decisions too degenerate to measure at all.
