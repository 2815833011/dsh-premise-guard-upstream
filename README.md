# @deepseek-ai/dsh-premise-guard

[中文](README.zh.md) | English

Post-compaction premise-drift guard. After a `compaction/summary` event, it extracts distinctive literal anchors (file paths, quoted literals, key=value pairs, error codes) from the shadowed span's text, checks whether the committed summary still contains them, and — when a critical anchor vanished — injects a one-shot notice into the next step telling the model what it may have lost and how to recover it from the append-only log.

## Why

Recall-oriented designs (recallable-compaction, session-query tools) make shadowed content reachable **when suspected**. Nothing says "you just dropped a key fact". This guard turns the compaction summary into a checked handoff: the model learns the moment a path, value, or error string it was relying on is no longer in the summarized context.

## How it works

| Hook | Role |
|---|---|
| `session/event` (`compaction/summary`) | Re-derives the shadowed span's text from `shadowedSeqs` (the log keeps every byte), extracts anchors, and compares against `event.data.summary`. |
| `agent/pre-step` | Injects one notice (plugin source) naming up to `maxAnchors` vanished anchors and the shadowed seq range, once per alarm; a new user prompt skips delivery. |

Anchor extraction is deterministic and pure — no LLM calls, no new storage.

## Config

| Field | Default | Meaning |
|---|---|---|
| `maxAnchors` | `5` | Vanished anchors named in one notice. |
| `minAnchorLength` | `6` | Anchors shorter than this are never critical. |
| `maxNoticeChars` | `400` | Notice text cap. |

All three must be integers `>= 1`; misconfiguration throws at plugin load.

## Install

Not on npm yet - install from this repository:

```sh
npm install github:ICCuse/dsh-premise-guard
# or: pnpm add github:ICCuse/dsh-premise-guard
```

Then mount the bundle (declared in package.json 'dsh.bundle'):

```yaml
- id: dsh-premise-guard
  name: 'dsh-premise-guard'
```

Or, once published, 'dsh plugin --profile web add dsh-premise-guard'.

## Install on a local profile (measured)

```sh
cd ~/.dsh/profiles/desktop
pnpm add link:~/Documents/dsh-workspace/link-plugins/dsh-premise-guard
```

Add `dsh-premise-guard` to `dsh.profile.bundles` in the profile's `package.json`.
The bundle ships its own `cordis.patch.yml`, so the row is inserted automatically —
do **not** copy that insert block into the profile's `cordis.patch.yml`, it would
duplicate the entry id and the settings panel would refuse to save.

To override config, add a top-level row instead:

```yaml
- id: dsh-premise-guard
  config:
    maxAnchors: 5
    minAnchorLength: 6
```

## Where it sits next to `dsh-handoff-state`

The two answer different questions and do not overlap:

| | `dsh-premise-guard` | `dsh-handoff-state` |
|---|---|---|
| Trigger | `compaction/summary` | every step |
| Question | "what literal did the summary just drop?" | "where was I when the last session stopped?" |
| Output | one-shot notice per alarm | 4000 B handoff section, re-rendered |
| Cost | zero when no compaction happened | one runtime-context line, cached by text |

`premise-guard` fires only after a compaction; `handoff-state` fires on every step but
renders byte-identical text while nothing changes, so the loop admits no new snapshot.

## Notes from a real install

- A notice is delivered **once per alarm**; a new user prompt skips delivery, so the
  model is never interrupted mid-thought by a stale reminder.
- Anchor extraction is deterministic — no model call, no new storage. The shadowed
  span is re-derived from `shadowedSeqs`, because the append-only log keeps every byte.
- If nothing ever fires, check that compaction actually happened: the guard has no
  timer, and a session that never compacts never triggers it.
