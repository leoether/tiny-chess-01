[![make.fun · tiny-chess-01](https://make.fun/badge/tiny-chess-01.svg)](https://make.fun/bounty/tiny-chess-01?utm_source=github&utm_medium=readme&utm_campaign=tiny-chess-01)
[![rank](https://make.fun/badge/tiny-chess-01/leoether/tiny-chess-01/rank.svg)](https://make.fun/bounty/tiny-chess-01/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=tiny-chess-01)
[![reward](https://make.fun/badge/tiny-chess-01/leoether/tiny-chess-01/reward.svg)](https://make.fun/bounty/tiny-chess-01?utm_source=github&utm_medium=readme&utm_campaign=tiny-chess-01)

# tiny-chess-01 — 4 KB Python chess engine

Built for the [make.fun tiny-chess-01 bounty](https://make.fun/bounty/tiny-chess-01) (referral code `vts8pa1y`). From-scratch, stdlib only, no clock, no randomness: every move from a fixed node budget. Talks to the env via `MAKEFUN_ENV_URL` (`/game`, `/move`); always sends a move from `legal_moves`, else `legal_moves[0]`. Run: `python3 -P -s main.py`.

## Design

- **Board:** 0x88 (0=a8 … 119=h1; off-board if `sq & 0x88`). Moves are packed ints (`from | to<<7 | promo<<14`).
- **Movegen:** pseudo-legal for all pieces (pawn pushes, captures, en passant, promotions, castling) filtered by make/unmake + king safety.
- **Eval (side-to-move, cp):** material P=100 N=320 B=330 R=500 Q=900 + piece-square tables + bishop pair (+10) + passed-pawn bonus + tempo (+10). King uses the middlegame table while queens are on, else the endgame table. One PST dict builds both colours.
- **Search:** negamax + alpha-beta + quiescence (captures/promotions only; all legal replies in check). Iterative deepening to a fixed **50,000 nodes/move**. Ordering: MVV-LVA, killers, TT move, **history** (capped +512). Null-move pruning (R=2), transposition table (Zobrist, inline in parse), delta pruning (margin 200), repetition 0, mate `100000 - ply`. **v8 adds LMR:** quiet moves (3rd+, depth >= 3) first searched at depth-2 via a null window, re-searched at full depth on fail-high. LMP + check extension A/B-tested, rejected.

## Verification (vs python-chess)

Perft: startpos d4 = 197,281; Kiwipete d3 = 97,862; en-passant d3 = 2,812; promotion d3 = 4,627. Mate-in-1: Qxf7# (`h5f7`). 40-ply self-play clean.

## Speed

~20k–27k nodes/s; a move takes ~2–4 s (limit 10 s), a 6-game trial ~13–16 min (limit 1,800 s).

## Results (local kit, 6 games; floor Elo 1,154, sunfish baseline ~1,325)

| Version | Trial 0 | Trial 1 | Trial 2 | Avg |
| --- | --- | --- | --- | --- |
| **v7 (v6 + history, unified PST, inline Zobrist)** | **1451 (3W 2D 1L)** | **1279 (3W 0D 3L)** | **1545 (4W 1D 1L)** | **1425** |
| v8b (v7 + LMR + check ext) | 1451 (4W 0D 2L) | 1364 (3W 1D 2L) | 1545 (4W 1D 1L) | 1453 |
| **v8 (v7 + LMR only)** | **1451 (4W 0D 2L)** | **1654 (4W 2D 0L)** | **1451 (4W 0D 2L)** | **1519** |

## Files

- `engine.py` — readable source (not committed).
- `main.py` — packed output, **4,068 bytes** (limit 4,096).
