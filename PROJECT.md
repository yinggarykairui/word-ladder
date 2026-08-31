# PROJECT.md — word-ladder

Kept per MANUAL.md §4 (every `size:m`+ build). The planner writes and revises
it; the shipper checks off the done-map at ship; any revisit reads it first and
updates it last.

- Hub issue: [factory-hub#9](https://github.com/yinggarykairui/factory-hub/issues/9) — `queued` → `speccing` → `building` → `shipped` (day 017)
- Size: `size:m` · Type: `type:game` · Day 017 (2026-08-10) · manual_version 1.6.1
- Idea source: seeded (warm-start pack, §16-P0)
- Increment 2: [factory-hub#50](https://github.com/yinggarykairui/factory-hub/issues/50) · Day 036 (2026-08-30) · manual_version 1.8.0 — the share card and two of the day-017 residuals

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
  identical in the spec, the code and the harness — a harness writing a
  different key asserts nothing (LESSONS 2026-08-07). (Day 036: this line read
  "and the README" for twenty days and the README has never named the key. The
  claim is corrected rather than the README padded — the rule is that the
  places which *use* the key agree, and the README does not use it.)
- **The word list is written by hand in this repo.** No download, no vendored
  dictionary, no runtime fetch. Provenance stated in the README.
- **The exclusion list is a fence,** not a backlog for today. Chiefly: no share
  card, no hints, no reveal, no archive UI, no dark mode. **Day 036: the share
  card is out of the fence and built** — it was nominated in this file as the
  first follow-up candidate, and it came back through a revisit issue, which is
  the door the fence has. The rest of the list stands.
- **The input rule is a fold, not a strip** (amended day 036, was a literal
  `replace(/[^A-Za-z]/g, "")`): NFD, drop the combining marks, then strip
  non-letters and slice to four. See the open thread below for why the build
  won this argument.

## 2. Architecture sketch

Single file, `index.html` at the repo root (GitHub Pages serves the root).
Plain `<script>`, no modules, no fetches — so it also runs from `file://`.

```
index.html
├─ <style>            paper/ink palette, one accent, system font stack.
│                     Ladder = one column: START · played rungs · input row ·
│                     TARGET behind a rule. Tap targets ≥44×44 CSS px; no text
│                     ever inside another control's hit box (LESSONS 08-09/10).
│                     The share fallback box stands where the input row stands,
│                     which is put away exactly when the box can be open.
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
   │                  try/catch + a shape check, and every write re-reads
   │                  the blob and merges the record before replacing it
   │                  (§4, two tabs); a `?d=` naming a date other
   │                  than today is a no-op writer (`?d=<today>` is today's
   │                  game, and loads and saves like a plain load — C4).
   ├─ the rule table  the spec's ordered rules, inline in commit(); returns
   │                  a reason, never throws.
   ├─ share           shareText() — date, moves, par, streak, demo URL, and
   │                  nothing a player could read a word out of. share() writes
   │                  it to navigator.clipboard; settle() decides whether the
   │                  answer still applies to the board it comes back to;
   │                  shareCopied()/shareFallback() are the two endings, and
   │                  fitShare() sizes the fallback field to the wrapped line.
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

Increment 1 (day 017) is the whole of v0. Increment 2 (day 036) is the share
card and two residuals. States: `todo` · `wip` · `done` · `cut`.

### Increment 1 — v0, the daily ladder

| # | Item | State |
|---|------|-------|
| 1.1 | `index.html` skeleton, paper/ink typographic layout, phone-first column | done |
| 1.2 | Word list: 600–900 hand-written common 4-letter words, sorted, unique | done |
| 1.3 | Wildcard-bucket adjacency graph + largest-component flood fill | done |
| 1.4 | `fnv1a32` + `mulberry32`; UTC `dateStr`; `?d=` parse via `URLSearchParams` (never `decodeURIComponent`) with regex + round-trip validation, bad values falling back to today and saying so on the message line | done |
| 1.5 | `puzzleFor(date)`: seeded start pick, BFS, distance 3–5 candidates, widen 2–6, sorted-scan fallback; returns `par` and BFS parents | done |
| 1.6 | Play loop: input normalisation on `input`, ordered validation table, append/undo, solved state, move count vs `par` | done |
| 1.7 | `word-ladder.v1` storage: progress restore, streak/best arithmetic, lapsed-streak display rule, corrupt-blob tolerance, `?d=<other date>` read-only | done |
| 1.8 | `tests.html`: determinism ×30 dates, solvability walk, list integrity, validation matrix (with positive rows), streak matrix (with a positive row) | done |
| 1.9 | Phone-width + hit-box sweep at 320×568 and 390×844, whole-box, both directions | done — and shipped as assertions in `tests.html` after cycle 1 (m9) |
| 1.10 | README made true, `screenshot.png` captured from the shipped build, LICENSE (MIT), description, topics | done — README, LICENSE and `screenshot.png` are in; the repo description is byte-equal to the README's opening sentence and eight topics are set, both written over the API by the shipper. The image day 017 shipped was a 1280×800 headless capture two moves into the 2026-08-10 puzzle; day 036 re-shot it (row 2.9) and the committed file is a 2560×1600 capture (1280×800 at 2×) of a solved and shared 2026-08-30 board. The old parenthetical was the day-017 caption, deleted from the README on day 036 and left standing in this cell, so the file described its own screenshot two different ways two sections apart — corrected day 037 |
| 1.11 | Pages enabled from repo root; live URL confirmed loading the working build | done — Pages enabled from `main` / root at ship. The Pages API reports the site `built` at the shipped sha, and a fetch of `https://yinggarykairui.github.io/word-ladder/` returns the live page. Neither the fixer's nor the critics' sandbox can reach `yinggarykairui.github.io` (403 `host_not_allowed`), so the witness is the conductor's session on a different transport — three networks, three answers (LESSONS 2026-08-04) |

Checklist items 1–7 in the issue comment map to: 1.4/1.5 → item 1; 1.5/1.8 →
item 2; 1.2/1.3/1.8 → item 3; 1.6/1.8 → item 4; 1.7/1.8 → item 5; 1.9 → item 6;
1.10/1.11 → item 7.

### Increment 2 — day 036, the share card

The one fenced-out v0 feature with real pull, plus the two day-017 residuals
that were decisions rather than instrumentation. Scope and fence:
[factory-hub#50](https://github.com/yinggarykairui/factory-hub/issues/50).

| # | Item | State |
|---|------|-------|
| 2.1 | `shareText()`: date · `solved in N moves · best possible P` · streak, then the demo URL. The status line's exact words, because the README promises the number is named the same way everywhere it appears. No word, no rung, no letter. | done |
| 2.2 | Share control, live only while the chain ends at the target — the rule the status line and the target label already follow. Says `Copied` for 1.6s, because a copy changes nothing else on the page. | done |
| 2.3 | The streak clause is omitted under `?d=`, not printed as 0 | done |
| 2.4 | Clipboard fallback: a read-only field sized to the wrapped line at the width it is read at, pre-selected, re-sized on rotation. Not a graceful extra — on `file://`, the documented way to run this page, `navigator.clipboard` is present in Chromium and the write comes back `NotAllowedError`; both the absent and the refused path end here | done |
| 2.5 | `settle()`: a write that comes back to a board that has moved announces nothing (fix cycle 1) | done |
| 2.6 | Rung `aria-label`s name the changed position; START and an unplayed TARGET stay bare (residual 13) | done |
| 2.7 | `#word:disabled` styling restored for the unreachable no-puzzle guard (residual 10) | done |
| 2.8 | Suite 218 → 279 **at this increment's ship**: three clipboard paths, `settle()`'s three guards told apart, both phone viewports, the labels, the fallback's fit, its re-fit, the clearance its focus ring needs and the size its text is set at. Mutation-tested after each of the three cycles. This cell has now been stale twice — written at 264 while the suite was 274, corrected to 274 while it was 279 — which is what a hand-copied count does. The count has moved again since (day 037's fixes added rows), and rather than chase it a third time the number is fixed here to the one this increment shipped with; the live number is the one `tests.html` prints | done |
| 2.9 | README true of the shipped build; `screenshot.png` re-shot on a solved, shared board; provenance footer in the revisit form | done |

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
  what is hidden and stops nowhere when nothing is — F6, recomputed on resize
  since cycle 2 (C9), because a rotation changes what is hidden without playing
  a word), and the START label scrolls away with
  the start word (it lives inside the scroller so it can never sit above a rung
  that is not the start).

- **Two 16px clearances around the input are load-bearing.** Above it (the
  `#inputrow` padding) and below it (the `.hr` top margin), because Chromium's
  touch adjustment retargets a nearby tap into the field. Measured while fixing
  the layout: at 10.2px below the field, 5 of 5 taps on that hairline were
  stolen into the field at 320×568; at 16px, 0 of 5. Neither gap may be scaled
  down for vertical compactness — `tests.html` asserts both.

- **Two tabs: the chain is last write wins, the record is merged.** The line
  that used to stand here said the loss was "one puzzle's chain, never the
  streak", and that clause was false. It was never measured; it was assumed from
  the shape of the code, and it was the whole justification for leaving the
  thread open. Driven for the first time on day 037: seed a 4-day streak banked
  yesterday, open tab A and leave it on the fresh board, solve the day in tab B
  (`streak 5 · best 5`, `lastSolved` today, `solved: true`, the solved chain in
  storage), then play one legal rung in tab A — and storage came back
  `streak 4 · best 4`, `lastSolved` yesterday, `solved: false`. The streak, the
  `best` and the day's solve were gone, silently and with no way back.
  `save()` wrote `store` — a snapshot taken at load — back as a whole blob with
  no re-read, so any write put that snapshot over everything since; `undo()`
  saves too, so the loss did not even need an accepted rung. The same hole opens
  in a single tab, because `today` is read once at load: a tab left open across
  UTC midnight keeps playing and saving the previous day's board.

  `save()` now re-reads the stored blob immediately before `setItem` and merges
  the fields that are facts about the player rather than about the board in
  front of this tab: `best` is the higher of the two, `lastSolved` and `streak`
  travel as a pair so the later date wins and brings its own streak with it
  (a tie between two writes for the same date keeps the longer streak), and a
  solve of *this* date recorded by anyone is this tab's solve too. A save that
  banked nothing can no longer rewrite a streak it never touched. `readStore()`
  returning junk, `null` or a throw means there is nothing to merge, not a
  broken board.

  What is still last write wins is the chain, and that part of the old entry
  stands: reload the stale tab and you see the other tab's rungs replaced by
  yours. Which chain should win is a design question, not a bug — and losing one
  puzzle's rungs is not losing the record of having played. There is still no
  `storage` listener and no live cross-tab sync; the merge happens on this tab's
  own writes. Asserted in `tests.html` with two boards held open at once, and
  every branch of the merge mutation-tested individually.

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
  line reads `solved in N moves · best possible P` while the chain ends at the
  target and `solved today · best possible N` after an Undo — `solved · best
  possible N` under
  `?d=`, where "today" would be a claim about a date that is not today (C2) —
  and re-reaching the target banks nothing a second time. Two deliberate
  deviations from the spec remain, both smaller than the ones they replace: the
  field is **not** disabled in the solved state (the spec's submit table says it
  is) — the whole input row is hidden instead while the chain ends at the target,
  because an empty field between the last rung and the target breaks a finished
  ladder open and can only take words that will be refused (U1/F9). Cycle 2
  finished the thought: Submit is disabled with the row (C8/U12). Cycle 1 left
  it live, which made it a control that read an empty field and changed nothing
  at all — two clicks on a solved board left the DOM byte-identical — and a
  button that does nothing teaches the player the button is dead. Undo keeps
  its box, its focus and its move, and the first Undo brings the row, the caret
  and Submit back. The ordered rule table still has one row the spec does not
  list — a submit while the chain already ends at the target is refused — but
  no player can reach it now; it is the guard on the state, exercised in
  tests.html by submitting the form directly.

- **Accented letters are folded, not stripped.** The spec's input rule is a
  literal `replace(/[^A-Za-z]/g, "")`, which turns `sîdé` into `SD` — four
  letters silently becoming two. The build normalises to NFD and drops the
  combining marks first, so `sîdé` is `SIDE`. The deviation came from a
  playtest finding (cycle-1 m6) and is kept on purpose; recorded here so the
  spec and the build stop disagreeing silently. **Closed day 036: the rule is
  amended, not the build.** Three review cycles and an evening all agreed the
  folding was the better behaviour and none of them changed the rule it
  deviated from, so the spec and the build disagreed in the player's favour for
  twenty days. The fold is now what §1 says the input rule is.

- **The `--chrome` constant is a measured upper bound, and it is guarded.** The
  ladder column's ceiling is `calc(100dvh - var(--chrome))`, where `--chrome`
  estimates every part of the page that is not the chain. It was measured on the
  rendered page at thirteen viewports (423.7px at 390x420, 511.5px at
  1920x1080 — it grows because the rhythm is vh-scaled) and the expression is
  deliberately 2.6-16px above the measurement everywhere: over-estimating costs
  a few pixels of ladder, under-estimating puts the buttons back under the
  fold. `tests.html` asserts the bound at eight viewports, so a change to the
  vertical rhythm cannot drift it silently.

- **The widen and fallback generation branches are unreachable insurance.**
  Every date from 2020 to 2028 (3,288 checked) is generated by the first
  `distance 3-5` phase, so `phase(2, 6)`, the sorted-scan fallback and the
  `if (!PUZ)` guard have never executed on a real date. They stay because the
  word list is editable, and the code says so. **Day 037: the `if (!PUZ)`
  branch is no longer uncovered.** It ships two controls and a form, and it
  shipped them live — Submit and Undo both enabled and drawn at full ink
  beside a correctly greyed field, and a press on Submit falling through to a
  native form GET that reloaded the page and threw away the line explaining
  why there was no board. `tests.html` now reaches the branch by writing a
  copy of the source with the word list cut to two words that are not
  neighbours, which is the only way in. `phase(2, 6)` and the sorted-scan
  fallback are still uncovered.

- **Share card — built day 036.** Cut from v0 because clipboard permissions, a
  fallback path and a toast were a cycle's worth of edge cases. That estimate
  was right: the whole of fix cycle 1 went on this feature's edges, and both of
  the two things v0 chose not to reason about are what the critics found. The
  clipboard is not a function call, it is a promise that outlives the board it
  was made about — press Share, Undo, take a longer route, re-solve, and an
  unguarded resolve says "result copied" over a seven-move ladder for a
  five-move copy. And the fallback field's height is not a constant: the line
  wraps to three rendered rows at 390px and four at 320px, so `rows="2"` showed
  a player the URL with their own result scrolled out of sight above it, under
  a message telling them to copy the line.
  Three things are now settled and should not be re-litigated without a reason:
  the card follows `atTarget` and not `solved`, because `N moves` counts a
  ladder that reaches the target; the streak clause is omitted rather than
  zeroed under `?d=`; and the line quotes the status line word for word, since
  the README promises this number is named the same way everywhere.
- **Opening the fallback grows the page on a short screen, and that is the
  law working.** Measured with the box open: 360x640, 390x844 and 1280x800 take
  it inside the 64px `.roomier` gives back and the document does not grow by a
  pixel; 320x568 grows 17px; 667x375 and 844x390 grow 97px,
  the column already sitting on its 4.8rem floor. (Re-measured day 037. This
  list read "360x640, 390x844 and 1280x800 … do not grow by a pixel; 360x640
  grows 3px" — 360x640 in both halves of its own sentence — and it read 37px
  at 320x568, which was true until the fallback's message was corrected and
  stopped wrapping to a second line there. The 64px the `!shareOpen` term
  refuses to lend twice is asserted in `tests.html` now, as a bound on the
  growth rather than as a class name.) (Re-measured in cycle 2, after the field went to
  the status line's 0.82rem and gained 8px of clearance under it so its focus
  ring is not drawn past the last scrollable pixel.) One screenful where there is room for one, an
  ordinary scrolling page where there is not. The property that has to hold is
  that the line is fully rendered and fully inside the viewport, which
  `tests.html` asserts at both phone widths; a still page showing two thirds of
  the line would have been the wrong trade. If this is ever revisited, the only
  way to buy the pixels back is to take them off the ladder column, and at
  320x568 there are 36px between it and its floor — enough, and it would leave
  two rungs visible on a solved board.
- **The win moment reflows.** Share arriving divides the button row three ways
  instead of two: at 390x844 Submit and Undo go 173px → 111.3px and `.controls`
  rises 32px, at the instant of the win. Left alone deliberately — the
  alternative is holding a third of the row empty for the whole game — but it
  is the day's largest unfixed UX finding and it belongs in front of whoever
  revisits the solved board.
- **`scrollTop = 0` after `select()` is untested insurance.** A mutant removing
  it survives the suite, because sizing the field to its content leaves nothing
  to scroll. Kept as a second line under `fitShare()` rather than deleted;
  recorded here so it is not mistaken for covered code (cf. the widen/fallback
  branches above). It is the **only** line of the day-036 code with no mutant
  that kills it: the ship-gate pass found two more — the 8px clearance under the
  share box and the field's `0.82rem` — and both are now guarded rather than
  registered here, which is the right way round. A register is where a line goes
  when it cannot be covered, not where it goes to avoid being.
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
