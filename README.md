# tiny-chess-01 — 4 KB Python chess engine

From-scratch engine for the make.fun tiny-chess-01 bounty. Stdlib only, no
clock, no randomness: every move comes from a fixed node budget. It talks to
the env via `MAKEFUN_ENV_URL` (`/game`, `/move`) and always sends a move
from the returned `legal_moves`, falling back to `legal_moves[0]`.

## Design

- **Board:** 0x88 (0=a8 … 119=h1; off-board if `sq & 0x88`). Moves are packed
  ints (`from | to<<7 | promo<<14`).
- **Movegen:** pseudo-legal for all pieces (pawn pushes, captures, en
  passant, promotions, castling) filtered by make/unmake + king safety.
- **Eval (side-to-move, cp):** material P=100 N=320 B=330 R=500 Q=900 +
  piece-square tables + bishop pair (+10) + passed-pawn bonus (grows with
  the square of the pawn's advance) + tempo (+10). King uses the middlegame
  table while queens are on the board (or non-pawn material > 1300), else
  the endgame table. One unified PST dict builds both colours' tables.
- **Search:** negamax + alpha-beta + quiescence (captures/promotions only;
  all legal replies in check). Iterative deepening to a fixed **50,000
  nodes/move**. Ordering: MVV-LVA, killers, transposition-table move,
  **history heuristic** (butterfly table, capped +512, boosted on beta
  cutoffs). Null-move pruning (R=2, only with non-pawn material and not in
  check), transposition table (Zobrist hash, inline in parse;
  exact/lower/upper), delta pruning in quiescence (margin 200), repetition
  scores 0, mate as `100000 - ply`.

## Verification (vs python-chess)

Perft: startpos d4 = 197,281; Kiwipete d3 = 97,862; en-passant d3 = 2,812;
promotion d3 = 4,627. Mate-in-1: Qxf7# (`h5f7`). 40-ply self-play clean.
Note: the brief lists 62,379 for the promotion perft, but python-chess
gives 4,627 and the engine agrees — the brief's number is not reachable
for that FEN.

## Speed

~20k–27k nodes/s. A move takes ~2–4 s worst case (limit 10 s);
a 6-game trial takes ~13–16 min (limit 1,800 s).

## Results (local kit, 6 games; floor Elo 1,154, sunfish baseline ~1,325)

| Version | Trial 0 | Trial 1 | Trial 2 | Avg |
| --- | --- | --- | --- | --- |
| v3 (no TT/null/delta) | 1191 (2W 1D 3L) | 1191 (2W 1D 3L) | — | 1191 |
| v5 (+TT, null-move, delta) | 1364 (3W 1D 2L) | 1099 (2W 0D 4L) | — | 1232 |
| v6 (v5 + passed pawns) | 1279 (2W 2D 2L) | 1279 (3W 0D 3L) | 1451 (4W 0D 2L) | 1336 |
| **v7 (v6 + history, unified PST, inline Zobrist)** | **1451 (3W 2D 1L)** | **1279 (3W 0D 3L)** | **1545 (4W 1D 1L)** | **1425** |

**vs sunfish (50k nodes/move):** v6 2W 2L — won both as black, lost both as
white. Both engines deterministic, so only two unique games; Elo ≈ 0.

## Files

- `engine.py` — readable source (not committed; perft helper in a separate
  verification copy).
- `main.py` — packed output, **4,056 bytes** (limit 4,096).
