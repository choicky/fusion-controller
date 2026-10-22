# R40 Task 3 — 自动卷轴 (automatic scrolling of the candidate list)

> **Status: owner-confirmed, implementation authorized, not yet complete.** The owner has decided that the candidate
> list should be able to grow into a scrolling multi-row arrangement and has authorized that implementation as its own
> task. Until it is implemented and verified, nothing here describes shipped behaviour: what ships today is the
> single-row horizontal / single-column vertical layout preference (D088) plus the candidate-window geometry, font and
> DPI work with the bounded single-column viewport (R39, fusion-windows PR #36).

## Confirmed interaction requirements

These are the owner's decisions and are not open questions for the implementation:

1. A new candidate session starts on **one line**. The list expands only after the first **next-page**, and only when
   the candidates exceed one line.
2. The per-row count follows the engine's own **`PageSize`**. The last row may be short, columns stay aligned, and the
   cell width stays stable.
3. Cells are drawn **single-line with an ellipsis**; the full text reuses R39's full-text box (no second mechanism).
4. **At most five rows.** On a work area that cannot hold five, fewer rows are shown. The font is never shrunk to make
   room.
5. **Next-page** first moves the scrolled window down **one row**; from the last row of the current scroll page it moves
   to the first row of the next scroll page. **Previous-page** is symmetric. Positions are clamped to the candidates that
   exist — there is no wrap-around.
6. **Number-key selection and the mouse act on the highlighted row**, and every visible row is selectable by the mouse.
7. The arrangement is **rebuilt or invalidated** on new input, partial selection, filtering, scheme change and connection
   change.
8. Normal horizontal output stays a **single row** and normal vertical output stays a **single column**; a long word must
   not turn either into an automatic matrix.

## Confirmed boundaries

- The engine's **layout-hint precedence is preserved** (`resolve_candidate_layout`): a `CandidateLayoutHint` from the
  engine still wins over the stored preference, so D088's semantics are not silently changed.
- The number shown on screen must remain the engine's own candidate index. Display order may be rearranged into rows; the
  numbering, the candidate identity and the committed string may not be redefined.
- No second candidate generation, no second sort, no second hotkey configuration: the list is read from the engine that
  already produced the page.
- A bulk read is treated as **possibly unsupported**. When it is, the layout falls back to the exact arrangement that
  ships today rather than degrading into a partial or guessed list.

## Integration points measured so far

1. **`CandidateList::toBulk()`** → `BulkCandidateList` — the bounded access path for candidates outside the current page.
   `totalSize()` gives the size, `candidateFromAll(int)` resolves a global index. A row layout needs a **bounded window**
   of that list (enough for the rows on screen plus the next scroll page), not a back-and-forth paging walk and never an
   iteration over every candidate.
2. **`candidateFromAll()` / the projection** — where a global index becomes the displayed candidate. Row/column mapping
   belongs after this resolution, so the display order matches the engine's semantic order.
3. **`CommonCandidateList`** — `setPageSize`/`pageSize`, `setGlobalCursorIndex`/`globalCursorIndex`,
   `setCursorIndex`/`cursorIndex`, `next()`/`prev()`, `hasNext()`/`hasPrev()` and `usedNextBefore()`. The scroll window's
   row step has to stay consistent with these, and with the paging keys the user has configured, instead of introducing a
   second notion of "page".
4. **List types after the usual transformations** — Pinyin (and Shuangpin) use `CommonCandidateList`; filtering, auxiliary
   codes and partial selection change what the page contains. The row layout therefore has to be derived from the page it
   is given each time, and invalidated when that page changes.
5. **Event loop** — the reads and the selections happen on the Fcitx event loop, like the existing projection, and a
   refreshed page invalidates the previous snapshot rather than keeping a stale reference.

## Not yet decided by this document

The exact row height and scroll-page size, the protocol increment needed to carry a row-layout hint to the Windows client
(and its sync with the settings client and the tests), and the verification plan. Those belong to the implementation task
and are not implied by the confirmations above.

## Authority

This is research input plus the owner's decision, not an Accepted Decision. The product boundary remains the layout
preference of D088 (fusion-windows PR #33, merged) and the candidate-window work of R39 (fusion-windows PR #36). The task
stays OPEN until the implementation is verified.
