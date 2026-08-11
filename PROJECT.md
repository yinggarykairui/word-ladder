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
   │                  try/catch + a shape check; a `?d=` naming a date other
   │                  than today is a no-op writer (`?d=<today>` is today's
   │                  game, and loads and saves like a plain load — C4).
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
| 1.9 | Phone-width + hit-box sweep at 320×568 and 390×844, whole-box, both directions | done — and shipped as assertions in `tests.html` after cycle 1 (m9) |
| 1.10 | README made true, `screenshot.png` captured from the shipped build, LICENSE (MIT), description, topics | done — README, LICENSE and `screenshot.png` (1280×800 headless capture, two moves into the 2026-08-10 puzzle) are in; the repo description is byte-equal to the README's opening sentence and eight topics are set, both written over the API by the shipper |
| 1.11 | Pages enabled from repo root; live URL confirmed loading the working build | done — Pages enabled from `main` / root at ship. The Pages API reports the site `built` at the shipped sha, and a fetch of `https://yinggarykairui.github.io/word-ladder/` returns the live page. Neither the fixer's nor the critics' sandbox can reach `yinggarykairui.github.io` (403 `host_not_allowed`), so the witness is the conductor's session on a different transport — three networks, three answers (LESSONS 2026-08-04) |

Checklist items 1–7 in the issue comment map to: 1.4/1.5 → item 1; 1.5/1.8 →
item 2; 1.2/1.3/1.8 → item 3; 1.6/1.8 → item 4; 1.7/1.8 → item 5; 1.9 → item 6;
1.10/1.11 → item 7.

## 4. Open threads

- **One screenful when there is room for one; an ordinary scrolling page when
  there is not.** Cycle 1's blocker was the page growing as the ladder grew,
  with the feedback line the first thing off the bottom. Cycle 1's fix then
  froze the document at the viewport height, and cycle 2's blocker was the
  consequence: on any viewport under ~420px tall — every phone in landscape —
  the ladder column computed to 0px with the chain inside it and no scrollbar
  anywhere to recover it, and the target word painted over the status line.
  The shape now: `min-height` (never `height`) on the page so the document can
  grow; wrappers that keep their automatic minimum size, so a box is never
  shorter than what it draws; and a ladder column with a real floor
  (`min-height: 4.8rem`, one legible rung plus its label) and a ceiling
  (`calc(100dvh - var(--chrome))`) that is what lets the column give up height
  at all. Below the floor the page itself takes the overflow. Measured at eight
  viewports x three chain states: no overlaps, the column never below 77px,
  every hidden pixel reachable by wheel and keyboard, and no page scroll at all
  at 320x568, 390x844 and 1280x800. Two consequences to keep in mind: on a short
  screen the chain box shows one rung and the rest scrolls inside it (it is a
  tab stop exactly while it has something to scroll, so the keyboard can reach
  what is hidden and stops nowhere when nothing is — F6), and the START label scrolls away with
  the start word (it lives inside the scroller so it can never sit above a rung
  that is not the start).

- **Two 16px clearances around the input are load-bearing.** Above it (the
  `#inputrow` padding) and below it (the `.hr` top margin), because Chromium's
  touch adjustment retargets a nearby tap into the field. Measured while fixing
  the layout: at 10.2px below the field, 5 of 5 taps on that hairline were
  stolen into the field at 320×568; at 16px, 0 of 5. Neither gap may be scaled
  down for vertical compactness — `tests.html` asserts both.

- **Two tabs, last write wins.** Open the puzzle in two tabs, play a rung in
  each, and reloading the first shows only the second's chain: every write is a
  whole-blob `setItem` and nothing listens for the `storage` event. Accepted for
  v0 — the loss is one puzzle's chain, never the streak, and the storage-event
  merge is a design question (which chain wins?) rather than a bug fix.

- **The ladder stops at the target.** Cycle 2's live-input fix let a legal rung
  be played *past* the target, which stored a chain the loader refuses on read
  (the target must be the last rung) and so emptied the day's progress on the
  next reload while storage still said the day was won. Cycle 3 refuses a submit
  once the last rung is the target. The invariant to keep: **no sequence of UI
  actions may produce a stored chain that `legalChain` rejects** — asserted in
  `tests.html` over three walks, and measured over 75 reload checkpoints
  including 60 mixed actions. Anything that ever appends to the chain again has
  to be checked against it.

- **The win belongs to the date, not to the chain.** Cycle 1 made a solve final
  (Undo disabled) to stop a player un-winning a banked streak; cycle 2 reverted
  that — it contradicted the spec and left a page with nothing focusable on it.
  The build now keeps two separate facts: `solved` (this UTC date has been
  solved, banked once, survives any Undo, persisted as `solved` in the blob) and
  `atTarget` (the chain in front of you ends at the target right now, which is
  what the ladder draws). Undo stays live after a solve, the status
  line reads `solved in N moves · shortest N` while the chain ends at the target
  and `solved today · shortest N` after an Undo — `solved · shortest N` under
  `?d=`, where "today" would be a claim about a date that is not today (C2) —
  and re-reaching the target banks nothing a second time. Two deliberate
  deviations from the spec remain, both smaller than the ones they replace: the
  field is **not** disabled in the solved state (the spec's submit table says it
  is) — the whole input row is hidden instead while the chain ends at the target,
  because an empty field between the last rung and the target breaks a finished
  ladder open and can only take words that will be refused (U1/F9); Submit is
  untouched, Undo keeps its box and its focus, and the first Undo brings the row
  back with the caret in it. And the ordered rule table has one row the spec does not list: a submit
  while the chain already ends at the target is refused (see the thread above).

- **Accented letters are folded, not stripped.** The spec's input rule is a
  literal `replace(/[^A-Za-z]/g, "")`, which turns `sîdé` into `SD` — four
  letters silently becoming two. The build normalises to NFD and drops the
  combining marks first, so `sîdé` is `SIDE`. The deviation came from a
  playtest finding (cycle-1 m6) and is kept on purpose; recorded here so the
  spec and the build stop disagreeing silently.

- **The `--chrome` constant is a measured upper bound, and it is guarded.** The
  ladder column's ceiling is `calc(100dvh - var(--chrome))`, where `--chrome`
  estimates every part of the page that is not the chain. It was measured on the
  rendered page at thirteen viewports (423.7px at 390x420, 511.5px at
  1920x1080 — it grows because the rhythm is vh-scaled) and the expression is
  deliberately 2.6-16px above the measurement everywhere: over-estimating costs
  a few pixels of ladder, under-estimating puts the message line back under the
  fold. `tests.html` asserts the bound at eight viewports, so a change to the
  vertical rhythm cannot drift it silently.

- **The widen and fallback generation branches are unreachable insurance.**
  Every date from 2020 to 2028 (3,000 checked) is generated by the first
  `distance 3-5` phase, so `phase(2, 6)`, the sorted-scan fallback and the
  `if (!PUZ)` guard have never executed on a real date. They stay because the
  word list is editable, but they are uncovered and the code says so.

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
