# Dependency Update Risk Map

Offline, read-only ranking of changes between two **exported, normalized** dependency lock snapshots. No package manager, advisory service or network is contacted. The caller supplies explicit use/rollback factors, advisory status and API compatibility notes for every changed dependency; absent context is unknown, never an assumed clean bill.

## Run

Node.js 22+, zero dependencies. From this repository:

```sh
node bin/dependency-update-risk-map.mjs --root examples --input passing.json
node bin/dependency-update-risk-map.mjs --root examples --input failing.json
npm run check
```

`--root` confines reads by real path; `--input` is a relative path within it. Optional `--human` prints one summary to stderr. Stdout is only the JSON report; nothing is written. Invalid options/configuration: exit 2, empty stdout. Unreadable, undecodable, malformed, incomplete or unsupported evidence: incomplete report, exit 2. High-risk evaluated update: fail, exit 1. Fully evaluated updates below the review threshold: pass, exit 0.

## Normalized input

Top-level `schemaVersion:"1"`, `complete:true`, `before`, `after`, `factors`, `advisories`, `apiNotes`. Before/after have `complete:true` and `dependencies:[{name,version}]`, with exact numeric `major.minor.patch` versions. The three context exports each have `complete:true` and `items`. Factor item: `{name,directUse,usage,rollback}` with usage `none|low|high`, rollback `ready|manual|none`, optional ignored `note`. Advisory item: `{name,severity}` with `none|low|moderate|high|critical`. API note: `{name,breaking}` boolean. Each changed dependency requires a row in all three context exports, including an explicit `severity:"none"` when advisory review found none. Duplicate names, incomplete exports, and unsupported shapes are incomplete. Only supplied data is ranked; no advisory lookup or API analysis is inferred.

## Transparent score

| Factor | Points |
| --- | ---: |
| major / minor / patch / downgrade / added / removed | 4 / 2 / 1 / 5 / 2 / 3 |
| direct use | 2 |
| usage none / low / high | 0 / 1 / 3 |
| rollback ready / manual / none | 0 / 1 / 3 |
| breaking API note | 4 |
| advisory none / low / moderate / high / critical | 0 / 1 / 2 / 3 / 4 |

`score >= 8` is high and emits `high-risk-update` (exit 1); 5–7 moderate; below 5 low. Ranked updates sort by descending score, then source ordinal. Each output row lists the exact named factors and points. A heavily used major update outranks an unused patch under equal remaining factors. Missing factors, advisory or API notes emit separate `factors-missing`, `advisory-missing`, or `api-note-missing` findings (incomplete 2). No updates is incomplete rather than a vacuous pass. `@export` is the fixed logical source role for `--input`; pointers and source ordinals identify evidence without publishing dependency names, values or host paths. Findings sort by `(location.file, location.pointer, ruleId)` using UTF-16 code-unit order. This score is a triage policy, not a vulnerability prediction.

## Limits and non-goals

1,048,576 UTF-8 bytes; 100 records **per collection** (each lock snapshot and each context export); JSON depth 4 from root depth 0; 5,000 ms injectable library clock. Exact N accepted; N+1 incomplete. Strict UTF-8 and duplicate JSON key rejection (including escaped keys) prevent ambiguous evidence. CLI read has a 5-second abort. No package resolution, network, install, upgrade, remediation, or runtime compatibility test.
