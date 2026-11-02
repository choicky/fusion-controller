# R53 key matrix — measured result

Produced by `tests/key_matrix.cpp` against the pinned engine, current `main` (R53 stopgap included). One row per (key, state, modifier); the CSV beside this file is the full matrix. `T` is the answer `OnTestKeyDown` gives (the shell's `routes_key` and `engine_has_work_for`, with the hard policy allowed); `K` is the key path's own result; `engine` is what Fcitx answered, either from the batch the dispatch produced or from a direct probe of a second context in the same state.

Total cells: **5220**

| verdict | cells |
|---|---|
| ok | 4818 |
| LOST | 204 |
| MISSED | 60 |
| FALLBACK | 138 |
| RECOVERY | 0 |
| unreached | 0 |

## By state

| | ok | LOST | MISSED | FALLBACK | RECOVERY | unreached |
|---|---|---|---|---|---|---|
| inactive | 866 | 4 | 0 | 0 | 0 | 0 |
| idle | 824 | 20 | 7 | 19 | 0 | 0 |
| composing | 774 | 72 | 23 | 1 | 0 | 0 |
| composing-selection | 778 | 68 | 23 | 1 | 0 | 0 |
| after-commit | 824 | 20 | 7 | 19 | 0 | 0 |
| after-toggle | 752 | 20 | 0 | 98 | 0 | 0 |

## By key class

| | ok | LOST | MISSED | FALLBACK | RECOVERY | unreached |
|---|---|---|---|---|---|---|
| letter | 884 | 0 | 0 | 52 | 0 | 0 |
| digit | 340 | 0 | 4 | 16 | 0 | 0 |
| oem | 380 | 0 | 16 | 36 | 0 | 0 |
| function | 768 | 96 | 0 | 0 | 0 | 0 |
| editing | 148 | 12 | 12 | 8 | 0 | 0 |
| navigation | 316 | 36 | 8 | 0 | 0 | 0 |
| numpad | 502 | 0 | 12 | 26 | 0 | 0 |
| numpad-off | 352 | 36 | 8 | 0 | 0 | 0 |
| lock | 108 | 0 | 0 | 0 | 0 | 0 |
| system | 108 | 0 | 0 | 0 | 0 | 0 |
| media | 648 | 0 | 0 | 0 | 0 | 0 |
| modifier | 264 | 24 | 0 | 0 | 0 | 0 |

## LOST cells (204)

T claimed the key, the engine handed it back, nothing reached the document.

- F1 / composing / none
- F1 / composing / shift
- F1 / composing-selection / none
- F1 / composing-selection / shift
- F2 / composing / none
- F2 / composing / shift
- F2 / composing-selection / none
- F2 / composing-selection / shift
- F3 / composing / none
- F3 / composing / shift
- F3 / composing-selection / none
- F3 / composing-selection / shift
- F4 / composing / none
- F4 / composing / shift
- F4 / composing-selection / none
- F4 / composing-selection / shift
- F5 / composing / none
- F5 / composing / shift
- F5 / composing-selection / none
- F5 / composing-selection / shift
- F6 / composing / none
- F6 / composing / shift
- F6 / composing-selection / none
- F6 / composing-selection / shift
- F7 / composing / none
- F7 / composing / shift
- F7 / composing-selection / none
- F7 / composing-selection / shift
- F8 / composing / none
- F8 / composing / shift
- F8 / composing-selection / none
- F8 / composing-selection / shift
- F9 / composing / none
- F9 / composing / shift
- F9 / composing-selection / none
- F9 / composing-selection / shift
- F10 / composing / none
- F10 / composing / shift
- F10 / composing-selection / none
- F10 / composing-selection / shift
- F11 / composing / none
- F11 / composing / shift
- F11 / composing-selection / none
- F11 / composing-selection / shift
- F12 / composing / none
- F12 / composing / shift
- F12 / composing-selection / none
- F12 / composing-selection / shift
- F13 / composing / none
- F13 / composing / shift
- F13 / composing-selection / none
- F13 / composing-selection / shift
- F14 / composing / none
- F14 / composing / shift
- F14 / composing-selection / none
- F14 / composing-selection / shift
- F15 / composing / none
- F15 / composing / shift
- F15 / composing-selection / none
- F15 / composing-selection / shift
- F16 / composing / none
- F16 / composing / shift
- F16 / composing-selection / none
- F16 / composing-selection / shift
- F17 / composing / none
- F17 / composing / shift
- F17 / composing-selection / none
- F17 / composing-selection / shift
- F18 / composing / none
- F18 / composing / shift
- F18 / composing-selection / none
- F18 / composing-selection / shift
- F19 / composing / none
- F19 / composing / shift
- F19 / composing-selection / none
- F19 / composing-selection / shift
- F20 / composing / none
- F20 / composing / shift
- F20 / composing-selection / none
- F20 / composing-selection / shift
- F21 / composing / none
- F21 / composing / shift
- F21 / composing-selection / none
- F21 / composing-selection / shift
- F22 / composing / none
- F22 / composing / shift
- F22 / composing-selection / none
- F22 / composing-selection / shift
- F23 / composing / none
- F23 / composing / shift
- F23 / composing-selection / none
- F23 / composing-selection / shift
- F24 / composing / none
- F24 / composing / shift
- F24 / composing-selection / none
- F24 / composing-selection / shift
- Escape / idle / none
- Escape / idle / shift
- Escape / after-commit / none
- Escape / after-commit / shift
- Escape / after-toggle / none
- Escape / after-toggle / shift
- Tab / idle / none
- Tab / idle / shift
- Tab / after-commit / none
- Tab / after-commit / shift
- Tab / after-toggle / none
- Tab / after-toggle / shift
- Insert / idle / none
- Insert / idle / shift
- Insert / composing / none
- Insert / composing / shift
- Insert / composing-selection / none
- Insert / composing-selection / shift
- Insert / after-commit / none
- Insert / after-commit / shift
- Insert / after-toggle / none
- Insert / after-toggle / shift
- Home / composing / shift
- Home / composing-selection / shift
- End / composing / shift
- End / composing-selection / shift
- PageUp / idle / none
- PageUp / idle / shift
- PageUp / composing / shift
- PageUp / composing-selection / shift
- PageUp / after-commit / none
- PageUp / after-commit / shift
- PageUp / after-toggle / none
- PageUp / after-toggle / shift
- PageDown / idle / none
- PageDown / idle / shift
- PageDown / composing / shift
- PageDown / composing-selection / shift
- PageDown / after-commit / none
- PageDown / after-commit / shift
- PageDown / after-toggle / none
- PageDown / after-toggle / shift
- Left / composing / shift
- Up / composing / shift
- Up / composing-selection / shift
- Right / composing / shift
- Down / composing / shift
- Down / composing-selection / shift
- Numpad0 (NumLock off) = Insert / idle / none
- Numpad0 (NumLock off) = Insert / idle / shift
- Numpad0 (NumLock off) = Insert / composing / none
- Numpad0 (NumLock off) = Insert / composing / shift
- Numpad0 (NumLock off) = Insert / composing-selection / none
- Numpad0 (NumLock off) = Insert / composing-selection / shift
- Numpad0 (NumLock off) = Insert / after-commit / none
- Numpad0 (NumLock off) = Insert / after-commit / shift
- Numpad0 (NumLock off) = Insert / after-toggle / none
- Numpad0 (NumLock off) = Insert / after-toggle / shift
- Numpad1 (NumLock off) = End / composing / shift
- Numpad1 (NumLock off) = End / composing-selection / shift
- Numpad2 (NumLock off) = Down / composing / shift
- Numpad2 (NumLock off) = Down / composing-selection / shift
- Numpad3 (NumLock off) = PgDn / idle / none
- Numpad3 (NumLock off) = PgDn / idle / shift
- Numpad3 (NumLock off) = PgDn / composing / shift
- Numpad3 (NumLock off) = PgDn / composing-selection / shift
- Numpad3 (NumLock off) = PgDn / after-commit / none
- Numpad3 (NumLock off) = PgDn / after-commit / shift
- Numpad3 (NumLock off) = PgDn / after-toggle / none
- Numpad3 (NumLock off) = PgDn / after-toggle / shift
- Numpad4 (NumLock off) = Left / composing / shift
- Numpad6 (NumLock off) = Right / composing / shift
- Numpad7 (NumLock off) = Home / composing / shift
- Numpad7 (NumLock off) = Home / composing-selection / shift
- Numpad8 (NumLock off) = Up / composing / shift
- Numpad8 (NumLock off) = Up / composing-selection / shift
- Numpad9 (NumLock off) = PgUp / idle / none
- Numpad9 (NumLock off) = PgUp / idle / shift
- Numpad9 (NumLock off) = PgUp / composing / shift
- Numpad9 (NumLock off) = PgUp / composing-selection / shift
- Numpad9 (NumLock off) = PgUp / after-commit / none
- Numpad9 (NumLock off) = PgUp / after-commit / shift
- Numpad9 (NumLock off) = PgUp / after-toggle / none
- Numpad9 (NumLock off) = PgUp / after-toggle / shift
- LShift / inactive / none
- LShift / inactive / shift
- LShift / idle / none
- LShift / idle / shift
- LShift / composing / none
- LShift / composing / shift
- LShift / composing-selection / none
- LShift / composing-selection / shift
- LShift / after-commit / none
- LShift / after-commit / shift
- LShift / after-toggle / none
- LShift / after-toggle / shift
- RShift / inactive / none
- RShift / inactive / shift
- RShift / idle / none
- RShift / idle / shift
- RShift / composing / none
- RShift / composing / shift
- RShift / composing-selection / none
- RShift / composing-selection / shift
- RShift / after-commit / none
- RShift / after-commit / shift
- RShift / after-toggle / none
- RShift / after-toggle / shift

## MISSED cells (60)

T declined a key the engine would consume. A modified chord is the platform policy (the shell never presents one), so an engine binding on it is given up; a character-less key is rule 2, and an editing key is the composition gate.

- 7 / composing / ctrl
- 7 / composing / ctrl+shift
- 7 / composing-selection / ctrl
- 7 / composing-selection / ctrl+shift
- OEM_1 / idle / ctrl
- OEM_1 / idle / ctrl+shift
- OEM_1 / composing / ctrl
- OEM_1 / composing / ctrl+shift
- OEM_1 / composing-selection / ctrl
- OEM_1 / composing-selection / ctrl+shift
- OEM_1 / after-commit / ctrl
- OEM_1 / after-commit / ctrl+shift
- OEM_PERIOD / idle / ctrl
- OEM_PERIOD / idle / ctrl+shift
- OEM_PERIOD / composing / ctrl
- OEM_PERIOD / composing / ctrl+shift
- OEM_PERIOD / composing-selection / ctrl
- OEM_PERIOD / composing-selection / ctrl+shift
- OEM_PERIOD / after-commit / ctrl
- OEM_PERIOD / after-commit / ctrl+shift
- Return / composing / ctrl
- Return / composing / ctrl+shift
- Return / composing-selection / ctrl
- Return / composing-selection / ctrl+shift
- Backspace / composing / ctrl
- Backspace / composing / ctrl+shift
- Backspace / composing-selection / ctrl
- Backspace / composing-selection / ctrl+shift
- Space / idle / ctrl
- Space / composing / ctrl
- Space / composing-selection / ctrl
- Space / after-commit / ctrl
- Delete / composing / ctrl
- Delete / composing / ctrl+shift
- Delete / composing-selection / ctrl
- Delete / composing-selection / ctrl+shift
- Left / composing / ctrl
- Left / composing-selection / ctrl
- Right / composing / ctrl
- Right / composing-selection / ctrl
- Numpad7 / composing / ctrl
- Numpad7 / composing / ctrl+shift
- Numpad7 / composing-selection / ctrl
- Numpad7 / composing-selection / ctrl+shift
- NumpadDecimal / idle / ctrl
- NumpadDecimal / idle / ctrl+shift
- NumpadDecimal / composing / ctrl
- NumpadDecimal / composing / ctrl+shift
- NumpadDecimal / composing-selection / ctrl
- NumpadDecimal / composing-selection / ctrl+shift
- NumpadDecimal / after-commit / ctrl
- NumpadDecimal / after-commit / ctrl+shift
- Numpad4 (NumLock off) = Left / composing / ctrl
- Numpad4 (NumLock off) = Left / composing-selection / ctrl
- Numpad6 (NumLock off) = Right / composing / ctrl
- Numpad6 (NumLock off) = Right / composing-selection / ctrl
- Numpad. (NumLock off) = Delete / composing / ctrl
- Numpad. (NumLock off) = Delete / composing / ctrl+shift
- Numpad. (NumLock off) = Delete / composing-selection / ctrl
- Numpad. (NumLock off) = Delete / composing-selection / ctrl+shift

## RECOVERY cells (0)

The dispatch ended in the cancelled-recovery path.

None.

## Cells that could not be reached (0)

None.

## Modifier coverage

| modifier | cells |
|---|---|
| none | 870 |
| shift | 870 |
| ctrl | 870 |
| alt | 870 |
| ctrl+shift | 870 |
| ctrl+alt | 870 |

## How to read the groups

The 204 LOST cells are not 204 independent defects. They fall into five shapes, and each one is a
different question for the design decision:

| shape | cells | what it is |
|---|---|---|
| the function row while a composition is live | 96 | rule 2 is deliberately gated on "nothing composing", so with a preedit up T still claims F1–F24 and the engine still hands them back. This is the stopgap's own remaining hole, and the largest one. |
| a Shift press | 24 | claimed by design (R39): the engine must see the press for its modifier-only binding, and a press carries no text, so "nothing was delivered" is not a lost character. Listed for completeness, not as a defect. |
| the navigation cluster | 36 | `Insert`, `PageUp` and `PageDown` are in neither `editing_key_needs_composition` nor rule 2's set, so they are claimed and forwarded whenever nothing composes; `Home`, `End` and the arrows with `Shift` while composing are claimed because the composition gate applies, and the engine forwards the modified form. |
| the keypad with NumLock off | 36 | the same eleven keys again, with the keypad's scan code and the extended flag clear — see the section below. The shape is identical because the keys are identical. |
| `Escape` and `Tab` while nothing composes | 12 | `Escape` is the shell's own "abandon" key and `Tab` is in neither list, so both are claimed and forwarded. |

The 60 MISSED cells are all one fact: **Fusion never presents a Ctrl chord, so every Fcitx binding
on one is dead through Fusion.** `Ctrl+Space` (the engine's own trigger), `Ctrl+;` (quickphrase),
`Ctrl+.` (punctuation toggle), `Ctrl+1..9` and `Ctrl+keypad 7` (candidate selection), `Ctrl+Return`,
`Ctrl+Backspace`, `Ctrl+Delete` and `Ctrl+Left/Right` are all consumed by the engine when asked
directly, and all of them are `T = FALSE` in the product. That is the platform policy stated in
`translate_key` — a chord is never presented — and the matrix is the first place its cost is
counted.

`RECOVERY` is zero, and that is a result rather than a gap: the cells the instruction expected
to end in the cancelled-recovery path (PrintScreen, Pause, ScrollLock, Apps while composing)
never reach the Host at all. `routes_key` refuses a `Type::Pass` key with no keysym before any
dispatch, so the Host's `key_name` — which throws for a keysym-less stimulus — is unreachable
from the key path. The matrix asserts exactly that for all four keys (`dispatched=0`, `t=0`).

## The keypad with NumLock off — and a withdrawn finding

An earlier revision of this file modelled these cells as `VK_NUMPAD0..9` (0x60–0x69) and
`VK_DECIMAL`, on the belief that Windows reports the keypad's own virtual key and only the
character changes with the lock. **That is wrong, and it produced a finding that is withdrawn:
the claim that "a keypad digit with NumLock off is typed as the top-row digit" described an event
the TIP never receives.** The keypad is translated in the keyboard layout, not in the application:
with NumLock off Windows delivers `VK_INSERT` (0x2D), `VK_END` (0x23), `VK_DOWN` (0x28),
`VK_NEXT` (0x22), `VK_LEFT` (0x25), `VK_CLEAR` (0x0C), `VK_RIGHT` (0x27), `VK_HOME` (0x24),
`VK_UP` (0x26), `VK_PRIOR` (0x21) and `VK_DELETE` (0x2E), with the keypad's own scan code and the
extended flag clear. The earlier cells handed the shell a virtual key that cannot arrive in that
state, so the digit they produced was an artefact of the harness, not a defect of the product. The
four operators (`/` `*` `-` `+`) do report the same virtual key and the same character in both
states, so they are the same cells and are not duplicated here.

Re-modelled, the eleven keys are 396 cells: **352 `ok`, 36 `LOST`, 8 `MISSED`**, and **each of the
ten that has a dedicated counterpart behaves exactly like it** — across all 360 paired cells every
recorded field is identical (T's answer, route, eaten/forward, operations, fallback, committed
text, the engine's own answer, the verdict). The extended flag is the only bit that differs, and
`translate_key` never reads it: it reaches only `modifier_identity`, which names right-Ctrl and
right-Alt in a diagnostic line. A non-extended navigation key is therefore *not* handled
differently from the dedicated key, and the measurement says so rather than assuming it.

`VK_CLEAR` has no dedicated counterpart and is its own case: `translate_key` gives it no keysym,
so `routes_key` refuses it before any dispatch in all 36 cells (`t=0`, `dispatched=0`) and the
application receives the key. No composition is disturbed and no recovery path is entered.

**What the corrected cells do show** is not a new defect but the reach of an existing one: the
keypad inherits the navigation cluster's gaps. `Numpad0` with NumLock off is `LOST` in the same ten
cells as `Insert` — pressing it does nothing in an application that decides from T alone — and
`Numpad9`/`Numpad3` (PgUp/PgDn) are `LOST` whenever idle, exactly as the dedicated keys are. A
keypad user with NumLock off therefore meets the Insert/PgUp/PgDn gap on the numeric keypad too,
which is worth knowing before the fix list is ordered.

## Delta against the head before the stopgap (`0565f8b`)

The same file was run at `0565f8b` (the merge before `fix/r53-stopgap`) in its own worktree, so the
delta is measured rather than inferred. Same engine, same key set, same states.

| | `0565f8b` | this head | change |
|---|---|---|---|
| `ok` | 4 492 | 4 818 | **+326** |
| `LOST` | 628 | 204 | **−424** |
| `MISSED` | 60 | 60 | 0 |
| `FALLBACK` | 40 | 138 | **+98** |

424 cells changed verdict, and they are exactly the two rules the stopgap added plus the delivery
it added:

- **326 `LOST` → `ok`**: every letter, digit and punctuation key in the engine's off state (rule 1,
  where T now declines and the application gets the key), and the idle function row, Apps,
  PrintScreen, Pause and ScrollLock (rule 2). `F2 / idle / none` is the canonical cell: `LOST` at
  `0565f8b` (`t=1`, engine forwarded, nothing delivered), `ok` here (`t=0`).
- **98 `LOST` → `FALLBACK`**: the first key after the toggle into English, which is now claimed,
  handed back, and delivered by the shell — the Notepad++ shape. `N / after-toggle / none` is the
  canonical cell, and it commits `n` exactly once.
- **`MISSED` did not move at all (60 → 60)**, which is the right answer: the stopgap never claimed
  a key it had been declining, so no engine binding that was dead became alive, and none that was
  alive became dead. The Ctrl-chord cost is unchanged by this round.

The baseline run exits non-zero on its first known-case assertion (`F2 / idle / none` is `LOST`
there), which is the delta stated as a test.

**The numbers in the previous revision of this document are superseded.** It reported 5 364 cells
(4 994 `ok`, 168 `LOST`, 64 `MISSED`) and 458 changed cells; those were computed with the
mis-modelled keypad rows. The arithmetic of the difference is exact: the fifteen rows that were
removed contributed 40 `LOST`→`ok` cells and the eleven corrected rows contribute 6, so
360 − 40 + 6 = 326.
