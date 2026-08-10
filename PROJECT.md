# PROJECT.md — word-ladder

Kept per MANUAL.md §4 (every `size:m`+ build). The planner writes and revises
it; the shipper checks off the done-map at ship; any revisit reads it first and
updates it last.

- Hub issue: [factory-hub#9](https://github.com/yinggarykairui/factory-hub/issues/9) — `queued` → `speccing` → `building`
- Size: `size:m` · Type: `type:game` · Day 017 (2026-08-10) · manual_version 1.6.1
- Idea source: seeded (warm-start pack, §16-P0)

---

## 1. The spec being converged on

A daily four-letter word ladder. One start word and one target word, derived
from the **UTC** date, identical for every player on that date. The player
types words; each must be on an in-repo list of common English four-letter
words and must differ from the word above it in exactly one position. Solving
on consecutive UTC days builds a streak held in `localStorage`.

The full spec is the comment on issue #9. The load-bearing decisions:

- **UTC, not local time.** "Same puzzle for everyone per date" is only true
  across timezones under UTC. The header prints `· UTC` so the roll-over is
  never a surprise. No `getFullYear`/`getMonth`/`getDate` in the file — only
  `getUTC*`.
- **Solvability by construction.** The target is found by a BFS *from* the
  start, inside the graph's largest connected component, so a path of exactly
  `par` moves provably exists in the shipped list. `tests.html` walks the
  reconstructed parent chain rather than trusting the claim.
- **Determinism by seeded PRNG.** `fnv1a32("word-ladder:" + dateStr)` →
  `mulberry32`. `Math.random` must not appear in the file.
- **Storage: one key, `word-ladder.v1`,** one JSON blob. The key string is
  identical in the spec, the code, the README and the harness — a harness
  writing a different key asserts nothing (LESSONS 2026-08-07).
- **The word list is written by hand in this repo.** No download, no vendored
  dictionary, no runtime fetch. Provenance stated in the README.
- **The exclusion list is a fence,** not a backlog for today. Chiefly: no share
  card, no hints, no reveal, no archive UI, no dark mode.

## 2. Architecture sketch

Single file, `index.html` at the repo root (GitHub Pages serves the root).
Plain `<script>`, no modules, no fetches — so it also runs from `file://`.

```
index.html
├─ <style>            paper/ink palette, one accent, system font stack.
│                     Ladder = one column: START · played rungs · input row ·
│                     TARGET behind a rule. Tap targets ≥44×44 CSS px; no text
│                     ever inside another control's hit box (LESSONS 08-09/10).
└─ <script>
   ├─ WORDS           one inlined space-separated string constant, sorted,
   │                  600–900 entries, split at load. The only "data file".
   ├─ graph.js-ish    wildcard buckets (_ATE / C_TE / CA_E / CAT_) → neighbours
   │                  in O(4N); one flood fill → largest connected component.
   ├─ rng             fnv1a32 + mulberry32, integer ops only.
   ├─ puzzleFor(date) seeded pick of start (degree ≥2) → BFS → candidates at
   │                  distance 3–5 (widen 2–6, then a sorted-scan fallback).
   │                  Pure function of (dateStr, WORDS). Returns
   │                  {start, target, par, parents}.
   ├─ store           read/write `word-ladder.v1`; every read and write in
   │                  try/catch + a shape check; `?d=` mode is a no-op writer.
   ├─ validate(word)  the ordered rule table from the spec; returns a reason,
   │                  never throws.
   └─ render          textContent only, never innerHTML with player input;
                      one aria-live="polite" message line, replaced not appended.

tests.html            standalone assertion page, no framework. Loads index.html
                      in an iframe and drives the real page, reading the engine
                      off the window.WL seam, so there is no second copy of the
                      code to drift. Prints N/N. Needs http:// (browsers give
                      file:// pages an opaque origin and block iframe access);
                      it says so on the page when it cannot get in.
```

Data flow is one direction: `dateStr → seed → puzzle → view`, with
`localStorage` as the only side channel and the only mutable state being the
chain array plus the streak blob.

## 3. Done-map

Increment 1 (day 017) is the whole of v0. States: `todo` · `wip` · `done` · `cut`.

### Increment 1 — v0, the daily ladder

| # | Item | State |
|---|------|-------|
| 1.1 | `index.html` skeleton, paper/ink typographic layout, phone-first column | done |
| 1.2 | Word list: 600–900 hand-written common 4-letter words, sorted, unique | done |
| 1.3 | Wildcard-bucket adjacency graph + largest-component flood fill | done |
| 1.4 | `fnv1a32` + `mulberry32`; UTC `dateStr`; `?d=` parse via `URLSearchParams` (never `decodeURIComponent`) with regex + round-trip validation, bad values falling back to today silently | done |
| 1.5 | `puzzleFor(date)`: seeded start pick, BFS, distance 3–5 candidates, widen 2–6, sorted-scan fallback; returns `par` and BFS parents | done |
| 1.6 | Play loop: input normalisation on `input`, ordered validation table, append/undo, solved state, move count vs `par` | done |
| 1.7 | `word-ladder.v1` storage: progress restore, streak/best arithmetic, lapsed-streak display rule, corrupt-blob tolerance, `?d=` read-only | done |
| 1.8 | `tests.html`: determinism ×30 dates, solvability walk, list integrity, validation matrix (with positive rows), streak matrix (with a positive row) | done |
| 1.9 | Phone-width + hit-box sweep at 320×568 and 390×844, whole-box, both directions | done |
| 1.10 | README made true, `screenshot.png` captured from the shipped build, LICENSE (MIT), description, topics | wip — README true and LICENSE in; screenshot/description/topics are the shipper's |
| 1.11 | Pages enabled from repo root; live URL confirmed loading the working build | todo |

Checklist items 1–7 in the issue comment map to: 1.4/1.5 → item 1; 1.5/1.8 →
item 2; 1.2/1.3/1.8 → item 3; 1.6/1.8 → item 4; 1.7/1.8 → item 5; 1.9 → item 6;
1.10/1.11 → item 7.

## 4. Open threads

- **Vertical scroll on short phones.** A solved five-rung puzzle is 667px tall
  at 320×568, so the controls sit below the fold and the page scrolls. That is
  allowed (only horizontal scroll is forbidden) and the tap sweep confirms the
  controls work once scrolled to, but the margins that keep taps off the
  controls are load-bearing (16px above the input; see the fix commit) and must
  not be traded away for vertical compactness.

- **Share card.** The one excluded feature with real pull (TASTE: "make things
  people share"): a copyable result line like `word-ladder 2026-08-10 · 5 moves
  (shortest 4) · streak 6`. Cut from v0 because clipboard permissions, a
  fallback path and a toast are a cycle's worth of edge cases, and two
  consecutive evenings were lost to polish regressions. **First follow-up
  candidate** — file it as a revisit issue, do not build it today.
- **The word list is load-bearing on order and contents.** Puzzles are a
  function of the date *and* of the exact list. Any later edit reshuffles every
  past and future puzzle. v0 freezes the list at ship. If the list is ever
  extended, either accept the reshuffle (nothing but streak counts is stored,
  so nothing breaks) or bump to a `word-ladder.v2` scheme that pins the list —
  decide at revisit, not now.
- **Difficulty drift.** `par` 3–5 was chosen unmeasured. Measured over 60
  consecutive dates around ship: par 3 ×11, par 4 ×22, par 5 ×27, and the
  distance-3–5 branch fired on all 60 — the widen and fallback branches have
  never run on a real date, so they are guarded but unexercised in the wild.
  The skew is toward the long end, so narrowing to 4–5 would cut the easiest
  fifth of days; worth a revisit decision, not a v0 change.
- **Component size — answered.** The shipped list is **874 words**; the largest
  connected component is **821 of them (93.9%)**, and 772 of those have degree
  ≥ 2. The ≥60% tripwire passes with room. Curating toward dense families
  (`-ate`, `-ail`, `-ell`, `-ill`, `-ore`, `b_ll`, `c_re`) is what bought it;
  a few dozen isolated abstract words were dropped rather than the threshold.
- **No archive.** `?d=` exists for verification and is documented honestly, but
  nothing in the page links to it. A real archive (and whether replaying old
  puzzles should ever count toward a streak — currently: never) is a separate
  design question.
- **Timezone honesty.** UTC is the right call for "same puzzle for everyone",
  but a player at UTC+13 rolls over at 1pm local. If that ever reads as a bug
  rather than a rule, the alternative is a local-date puzzle with a stated
  loss of global sameness. Not a v0 question.
