# R40 Task 3 — 自动卷轴 (automatic scrolling of the candidate list)

> **Status: implemented and verified in isolation (fusion-windows PR #37, head `43a832f`), packaged; not installed, not
> checked in an application, not accepted.** This file is the research record behind that implementation plus the owner's
> decision. It is not an Accepted Decision: the product boundary remains D088's presentation preference and R39's candidate
> window.

## Confirmed interaction requirements

The owner's decisions, unchanged by the implementation:

1. A new candidate session starts on **one line**. The list expands only after the first **next-page**, and only when the
   candidates exceed one line.
2. The per-row count follows the engine's own **`PageSize`**. The last row may be short, columns stay aligned, and the
   cell width stays stable.
3. Cells are drawn **single-line with an ellipsis**; the full text reuses R39's full-text box (no second mechanism).
4. **At most five rows.** On a work area that cannot hold five, fewer rows are shown. The font is never shrunk.
5. **Next-page** moves down one row; from the last row of what is on screen it moves to the first row of the next
   screenful. **Previous-page** is symmetric. Positions are clamped; there is no wrap-around.
6. **Number keys act on the row the engine is on**, and every visible row is selectable by the mouse.
7. The arrangement is **rebuilt or invalidated** on new input, partial selection, filtering, scheme change and connection
   change.
8. Normal horizontal output stays a **single row** and normal vertical output stays a **single column**.

## What the research got right, and what it had to correct

The first version of this file assumed a window of several pages would need a **paging walk**: set the page, read, put the
page back. That was never necessary, and doing it would have left the engine's page and its `usedNextBefore` state moved
after merely *looking* at candidates. The pinned source offers a direct read, and the implementation uses only that:

- `CandidateList::toBulk()` → `BulkCandidateList::candidateFromAll(int)` / `totalSize()`. `CommonCandidateList` registers
  itself as its own bulk source (`candidatelist.cpp:465-473`), and `candidateFromAll` is `candidateAt(global)`, i.e. the
  same word the engine would hand out for that page (`candidatelist.cpp`, `candidateFromAll`/`candidateAt`).
- `CommonCandidateList::pageSize()`, `currentPage()`, `totalPages()`, `usedNextBefore()`. Together with the bulk read these
  say which pages are on screen, where the engine is, and whether anything has been turned yet (**necessary, and — as the
  second pass below found — not sufficient**: they describe the engine, and know nothing about how many rows the window can
  actually draw, nor whether the window was read in full).
- `BulkCursorCandidateList` (`setGlobalCursorIndex`) exists but is a *cursor* interface: it does not move the page, and it
  is therefore not a way to select a candidate on another page. Selection goes through the word itself.
- A `CandidateWord::select` implementation depends on the word and the engine's context, not on the page — measured by
  reading the pinned pinyin candidates (`pinyincandidate.cpp`): `commitString`, `context_.select(...)`,
  `engine_->updateUI(...)`.

Two things the research had listed as open were then settled by measurement rather than by argument:

- **`usedNextBefore()` is per list.** A new input produces a new candidate list, so it is false again, which is exactly the
  "a new session starts on one line" requirement. That is why the window needs no session state on the Windows side.
- **`label()` is page-local.** Pinyin returns `1.`…`7.` for its current page and nothing beyond it, so numbers can only
  honestly be drawn on the row the engine is on. Drawing them on every row would name keys that select other candidates.

## What the second pass corrected

The first implementation of this record's plan was wrong in four ways, each found by review and each now measured:

1. **"The four numbers decide the window" was too strong.** They decide what the engine is doing; they say nothing about the
   window. Two facts live only on the Windows side and had to travel to the host: how many complete rows the popup can draw
   (`CandidatePanelLayout::row_capacity`, carried per key in `BackendStimulus::window_rows`, protocol 6 → 7), and — on the
   engine side, but not visible from the four numbers — whether the window was read *completely*. Without the first, the host
   grouped five rows while a short screen drew three, and the last visible row was a mid-page row; without the second, a
   window could be published with the engine's own page missing from it while its number keys still selected from that page.
2. **Paging does not keep the column.** Pinned pinyin sets `CursorPositionAfterPaging::ResetToFirst` on its list
   (`pinyin.cpp:562`, `1677`), so every page turn put the cursor on the first column — the opposite of the confirmed
   behaviour. No second cursor was needed: the list's own public policy setter is set to `SameAsLast`, whose implementation
   (`fixCursorAfterPaging`) is exactly "same index, or the last candidate when the page is shorter". The engine keeps owning
   the cursor; both the configured paging keys and the popup's arrow reach it through the engine's own `setPage`.
3. **A truncated window is worse than no window.** The fallback rule is now a decision the projection makes before publishing:
   contiguous, and containing every candidate of the engine's page, or the ordinary page projection is sent instead. A short
   last page is not a failure — it is where the candidates end — so the window is bounded by the list's real total size.
4. **The wire checks were weaker than they looked.** `current_page == total_pages` passed, and the test that claimed to prove
   "an ordinary page carries no scrolling fields" also changed the page size, so it would have passed with the relation
   unchecked. Both are fixed, and the arithmetic is now done in 64-bit after the bounds are checked.

## What the implementation is

The product side is described in `docs/windows-architecture.md` §7.2 and in fusion-windows PR #37. In short: the page
carries the engine's paging facts; `plan_scroll_window` turns them into a window with no other input; the window is read
through `toBulk()` in one projection; the TSF lays the entries out as a grid whose columns are `PageSize`; a cell of another
row is selected by global index; and a list that cannot supply any of that keeps the arrangement that shipped before.

## Verification

- `tests/candidate_auto_scroll.cpp` — real pinned engine, Pinyin and Shuangpin: one row → first next expands → row steps
  follow the engine page **and keep the column** (3 → 3, 1 → 1) → the next screenful's first row → a short last page clamps
  the cursor → a global selection equal to paging there and selecting the same cell → number keys equal to the row's cells →
  a capacity of 3 (and of 2) groups and steps correctly → new input back to one row → vertical/horizontal unchanged.
- `tests/candidate_window_layout.cpp` — the real popup: column alignment, a page size of seven, a short last row, a work
  area too short for five rows, the reported capacity equal to the rows actually drawn (five budgets, and a 60 px font),
  a long cell's full text.
- `tests/fcitx_candidate_filter.cpp` — the fallbacks the real engine cannot provoke: a byte budget that cannot hold the
  window, a bulk read that refuses an index mid-window, a list that is not paged, and a short last page that stays a window.
- Six discriminating failures across the two passes: no window, labels on every row, a global index read as page-local, no
  paging policy (the column is lost), the capacity ignored (wrong grouping), and no completeness decision (a hole is
  published).
- Settings: 634 checks + 8 after an orderly Host restart, run against the **packaged** Host.

## Not executed

Any in-application check (needs an authorized install), a real dual-monitor visual check, and an observed engine cursor on a
later row — the isolated Pinyin engine reports `cursorIndex() == 0`, so "the window follows the highlighted row" is covered
by the projection arithmetic and the layout anchor rather than by a cursor the engine moved.

## Authority

Research input plus the owner's decision, not an Accepted Decision. The product boundary remains the presentation
preference of D088 (fusion-windows PR #33, merged) and the candidate-window work of R39 (fusion-windows PR #36, merged).
