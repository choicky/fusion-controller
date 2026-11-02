# R53 key matrix — measured result

Produced by `tests/key_matrix.cpp` against the pinned engine, current `main` (R53 stopgap included). One row per (key, state, modifier); the CSV beside this file is the full matrix. `T` is the answer `OnTestKeyDown` gives (the shell's `routes_key` and `engine_has_work_for`, with the hard policy allowed); `K` is the key path's own result; `engine` is what Fcitx answered, either from the batch the dispatch produced or from a direct probe of a second context in the same state.

Total cells: **5364**

| verdict | cells |
|---|---|
| ok | 4994 |
| LOST | 168 |
| MISSED | 64 |
| FALLBACK | 138 |
| RECOVERY | 0 |
| unreached | 0 |

## By state

| | ok | LOST | MISSED | FALLBACK | RECOVERY | unreached |
|---|---|---|---|---|---|---|
| inactive | 890 | 4 | 0 | 0 | 0 | 0 |
| idle | 854 | 14 | 7 | 19 | 0 | 0 |
| composing | 810 | 62 | 21 | 1 | 0 | 0 |
| composing-selection | 804 | 60 | 29 | 1 | 0 | 0 |
| after-commit | 854 | 14 | 7 | 19 | 0 | 0 |
| after-toggle | 782 | 14 | 0 | 98 | 0 | 0 |

## By key class

| | ok | LOST | MISSED | FALLBACK | RECOVERY | unreached |
|---|---|---|---|---|---|---|
| letter | 884 | 0 | 0 | 52 | 0 | 0 |
| digit | 340 | 0 | 4 | 16 | 0 | 0 |
| oem | 380 | 0 | 16 | 36 | 0 | 0 |
| function | 768 | 96 | 0 | 0 | 0 | 0 |
| editing | 148 | 12 | 12 | 8 | 0 | 0 |
| navigation | 316 | 36 | 8 | 0 | 0 | 0 |
| numpad | 1030 | 0 | 24 | 26 | 0 | 0 |
| lock | 108 | 0 | 0 | 0 | 0 | 0 |
| system | 108 | 0 | 0 | 0 | 0 | 0 |
| media | 648 | 0 | 0 | 0 | 0 | 0 |
| modifier | 264 | 24 | 0 | 0 | 0 | 0 |

## LOST cells (168)

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

## MISSED cells (64)

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
- NumpadMultiply-NumLockOff / composing-selection / none
- NumpadMultiply-NumLockOff / composing-selection / shift
- NumpadAdd-NumLockOff / composing-selection / none
- NumpadAdd-NumLockOff / composing-selection / shift
- NumpadSubtract-NumLockOff / composing-selection / none
- NumpadSubtract-NumLockOff / composing-selection / shift
- NumpadDecimal-NumLockOff / composing-selection / none
- NumpadDecimal-NumLockOff / composing-selection / shift
- NumpadDivide-NumLockOff / composing / none
- NumpadDivide-NumLockOff / composing / shift
- NumpadDivide-NumLockOff / composing-selection / none
- NumpadDivide-NumLockOff / composing-selection / shift

## RECOVERY cells (0)

The dispatch ended in the cancelled-recovery path.

None.

## Cells that could not be reached (0)

None.

## Modifier coverage

| modifier | cells |
|---|---|
| none | 894 |
| shift | 894 |
| ctrl | 894 |
| alt | 894 |
| ctrl+shift | 894 |
| ctrl+alt | 894 |

## How to read the groups

The 168 LOST cells are not 168 independent defects. They fall into these shapes, and each one
is a different question for the design decision:

| shape | cells | what it is |
|---|---|---|
| the function row while a composition is live | 96 | rule 2 is deliberately gated on "nothing composing", so with a preedit up T still claims F1–F24 and the engine still hands them back. This is the stopgap's own remaining hole, and the largest one. |
| a Shift press | 24 | claimed by design (R39): the engine must see the press for its modifier-only binding, and a press carries no text, so "nothing was delivered" is not a lost character. Listed for completeness, not as a defect. |
| PageUp / PageDown while nothing composes | 16 | not in `editing_key_needs_composition` and not in rule 2's set, so they are claimed and forwarded. |
| Escape and Tab while nothing composes | 12 | the same shape one class further out: `Escape` is the shell's own "abandon" key and `Tab` is in neither list. |
| Insert | 10 | in neither list, so it is claimed in every state and forwarded in all of them. |
| a navigation key with Shift while composing | 10 | Home/End/arrows with Shift are claimed while composing (the composition gate applies) and the engine forwards the modified form. |

The 64 MISSED cells are the mirror image, in two groups:

- **52 are Ctrl chords** (`Ctrl` and `Ctrl+Shift`): `Ctrl+Space` (the engine's own trigger),
  `Ctrl+;` (quickphrase), `Ctrl+.` (punctuation toggle), `Ctrl+1..9` and `Ctrl+keypad 7`
  (candidate selection), `Ctrl+Return`, `Ctrl+Backspace`, `Ctrl+Delete`, `Ctrl+Left/Right` and
  the keypad operators. Fusion never presents a chord — that is the platform policy stated in
  `translate_key` — so every engine binding on one is dead through Fusion. This is the first
  place the cost is counted.
- **12 are the five keypad operators with NumLock off, while composing**: the shell has no keysym
  for them (Windows returns no character and `translate_key` has no entry), so it never presents
  them, and the engine would have used them to move the candidate window.

**A new defect the matrix found: keypad digits with NumLock off are typed as top-row digits.**
`classify_key` maps `VK_NUMPAD0..9` to a digit whatever the NumLock state, and with NumLock off
the keysym is empty — so the shell presents the key as a character with no keysym, and the Host
falls back to the stimulus's symbolic name, which is that digit. Measured in every state where the
engine is idle: `Numpad0-NumLockOff / idle / none` is claimed in T and commits `0`. The
application would have seen `End`. 64 of the 360 keypad-with-NumLock-off cells type a digit; the
operators are not affected. This is pre-existing and outside this round's scope to fix, but it is
the kind of cell the matrix exists to find, and it is recorded here rather than left for the
next surprise.

`RECOVERY` is zero, and that is a result rather than a gap: the cells the instruction expected
to end in the cancelled-recovery path (PrintScreen, Pause, ScrollLock, Apps while composing)
never reach the Host at all. `routes_key` refuses a `Type::Pass` key with no keysym before any
dispatch, so the Host's `key_name` — which throws for a keysym-less stimulus — is unreachable
from the key path. The matrix asserts exactly that for all four keys (`dispatched=0`, `t=0`).

## Delta against the head before the stopgap (`0565f8b`)

The same file was run at `0565f8b` (the merge before `fix/r53-stopgap`) in its own worktree, so the
delta is measured rather than inferred. Same engine, same key set, same states.

| | `0565f8b` | this head | change |
|---|---|---|---|
| `ok` | 4 634 | 4 994 | **+360** |
| `LOST` | 626 | 168 | **−458** |
| `MISSED` | 64 | 64 | 0 |
| `FALLBACK` | 40 | 138 | **+98** |

458 cells changed verdict, and they are exactly the two rules the stopgap added plus the delivery
it added:

- **360 `LOST` → `ok`**: every letter, digit and punctuation key in the engine's off state (rule 1,
  where T now declines and the application gets the key), and the idle function row, Apps,
  PrintScreen, Pause and ScrollLock (rule 2). `F2 / idle / none` is the canonical cell: `LOST` at
  `0565f8b` (`t=1`, engine forwarded, nothing delivered), `ok` here (`t=0`).
- **98 `LOST` → `FALLBACK`**: the first key after the toggle into English, which is now claimed,
  handed back, and delivered by the shell — the Notepad++ shape. `N / after-toggle / none` is the
  canonical cell, and it commits `n` exactly once.
- **`MISSED` did not move at all (64 → 64)**, which is the right answer: the stopgap never claimed
  a key it had been declining, so no engine binding that was dead became alive, and none that was
  alive became dead. The Ctrl-chord cost is unchanged by this round.

The baseline run exits non-zero on its first known-case assertion (`F2 / idle / none` is `LOST`
there), which is the delta stated as a test.
