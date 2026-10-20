# No-change comparison — what was compared and what it showed

The concern: the UI's apply path sends back a whole value tree, so an unmodified load → apply could
silently reset options the user never touched, or rewrite learning data.

## Procedure

On the profile already holding `PageSize = 9` (after the round trip and an orderly restart):

1. Read the configuration through the pipe and dump the **entire value tree** in a canonical,
   order-stable form — every leaf as `path=value`, nested maps flattened with a `0x1f` separator,
   list elements indexed, all lines sorted (566 lines).
2. Fingerprint the redirected profile: SHA-256 of **every file** under the profile directory,
   path-sorted, rendered as one string (covers the saved configuration *and* the learning data).
3. Drive the UI in `--nochange` mode: load, then `save()` with nothing modified, then reload.
4. Repeat steps 1 and 2 and compare.

## Result

- **Value tree: identical.** Both dumps are 566 lines and have the same SHA-256:
  `b87bb45e136741382ae0ac5d7861f7d30bfd0df3b4cc8cb9710ac7c7d55ead1d`.
  (The dumps themselves stay in the recovery workspace as `evidence/spike3-values-before.txt` and
  `-after.txt`; this hash is what makes the equality checkable.)
- **Profile files: identical.** The whole-directory fingerprint before and after the no-change apply
  is byte-identical, so neither the saved configuration nor any learning file was rewritten.
- `PageSize` after the no-change round trip: 9 — unchanged, as expected.
- The run's own checks reported `PASS engine value tree unchanged after a no-change apply: True` and
  `PASS profile files (config + learning) untouched: True` (`ui-roundtrip.log`, last section).

## Limits of this comparison

- It covers one profile, one URI form (`fcitx://config/inputmethod/pinyin`) and one no-change cycle.
- No typing occurred during the run, so the learning files were expected to be quiescent; the test
  proves the apply path did not touch them, not that learning behaves correctly under input.
- Byte equality is a strong but narrow claim: it shows nothing was rewritten, not that every option
  round-trips semantically under deliberate modification.
