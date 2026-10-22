# R40 Task 3 — Multi-row candidate matrix analysis

> **Status: NOT IMPLEMENTED.** This is a separate future task; no implementation work has started.
> 
> The candidate window currently renders a single row (horizontal or vertical list). A multi-row matrix layout — showing candidates in a grid — is a deferred capability, not a current requirement.

## Scope

A multi-row candidate matrix would change how candidates are arranged and navigated:
- **Rendering:** more than one row or column of candidates in the popup
- **Navigation:** arrow keys move across rows as well as along them
- **Selection:** the highlighted index maps to a two-dimensional position

## Key findings from preliminary analysis

The following three functions are the primary integration points where a matrix layout would need to coordinate with the existing candidate system:

1. **`toBulk()`** — The bulk candidate access path that the current presentation layer uses to retrieve the full candidate list from the engine. A matrix layout would need to consume this bulk data and partition it into rows.

2. **`candidateFromAll()`** — The resolution that turns a raw engine candidate index into the display candidate used by the projection layer. Row/column mapping would need to happen after this resolution so that the displayed order matches the engine's semantic order.

3. **`setGlobalCursorIndex()`** — The global cursor index that tracks which candidate is currently highlighted. In a matrix, this single index must map to a two-dimensional (row, column) position, and navigation keys must update it accordingly while preserving the engine's own paging boundaries.

## Deferred decision

Whether a multi-row matrix is ever implemented depends on:
- Owner decision that the single-row layout is insufficient
- A separate design task that defines the exact matrix dimensions, navigation model, and engine coordination
- Evidence that the three integration points above can be adapted without duplicating candidate semantics in the TSF DLL

## Authority

This analysis is research input, not an Accepted Decision. The current product boundary is the single-row horizontal/vertical layout preference (D088, fusion-windows PR #33, merged); the candidate-window geometry, font and DPI fixes are R39 (fusion-windows PR #36).
