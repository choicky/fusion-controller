# Transport probe — what was tested and what it returned

A standalone Qt program (`$SPIKE/src/probe`, binary `seam-probe.exe`) answering three questions
before any adapter was written: does a synthesized `QDBusPendingCall` work with **no session bus**,
does its watcher deliver `finished()`, and do the custom types `GetConfig` needs survive it.

The probe registers a stand-in for `FcitxQtConfigType` (a structure of `QString` + `QStringList`,
declared with `Q_DECLARE_METATYPE` and a `QDBusArgument` `operator<<`) and runs four strategies
against locally built `QDBusMessage` replies:

| Strategy | What it does | Result |
| --- | --- | --- |
| S1 | `fromCompletedCall` + `QDBusPendingCallWatcher` + `QDBusPendingReply<QVariant>` | call created **PASS**; `finished()` delivered **PASS**; `QDBusPendingReply` **FAIL** |
| S3 | same, with the type list marshalled explicitly via `QDBusMetaType::marshall` | identical to S1 — **FAIL** at `QDBusPendingReply` |
| S5 | `watcher->reply().arguments()` read directly, no `QDBusPendingReply` | **PASS** — two arguments; `QDBusVariant` → `QVariantMap` with the expected value; `QList<ProbeConfigType>` with both fields intact |
| S4 | a `void` reply (`QDBusPendingReply<>`) | **PASS** — `finished()` and a valid pending reply |

Session bus at run time: **none** (the probe reports `session bus connected: no`).

Decisive failure, verbatim:

```
reply.error = Unexpected reply signature: got "", expected "v"
```

`QDBusMessage::signature()` returns an empty string for a locally built reply: QtDBus computes the
signature only while marshalling to the wire, and exposes no setter. A synthesized call can
therefore carry arguments that read back perfectly, but any `QDBusPendingReply<T>` over it is
invalid.

**Consequence for the design:** the approach of "wrap a finished message so the upstream call sites
compile unchanged" is disproven — it is retained in the feasibility document as historical
experimental evidence, not as the plan. The current approach is the fallback that document already
named: a small local asynchronous interface at the affected call sites (`ControllerProxy` returning
a local `PendingCall` that emits `finished()`), which is what the UI round trip in this directory
exercised.

**Record-keeping note:** the probe's console output was read during the run but not captured to a
file at the time; the table above is the recorded result, and the exact error string is quoted from
that run. The probe source and binary remain in the recovery workspace
(`$SPIKE/src/probe`, `$SPIKE/build/probe`), so the run is repeatable there.
