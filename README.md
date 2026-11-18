# eucp-anchor-log

Public transparency log for the EU Multi-Framework Compliance Platform.

A hash stored beside the data in our own database proves nothing — an
administrator could recompute it. This repository holds each batch of
compliance-check evidence hashes **outside** our infrastructure, committed to
GitHub, timestamped and signed by GitHub, and independently fetchable by any
auditor.

## Format

```
anchors/<YYYY-MM-DD>/<HHMMSS>-<batch8>.json
```

Each manifest is a JSON object:

| Field | Meaning |
| --- | --- |
| `format` | Always `"eucp-anchor/1"` |
| `org_id` | The tenant the batch belongs to |
| `generated_at` | UTC timestamp when the manifest was built |
| `batch_sha256` | SHA-256 over `{format, org_id, entries}` — content only |
| `entries[]` | `{check_id, control_code, evidence_sha256, checked_at, status}` |

Entries are sorted by `check_id` before hashing, so the same set of checks
always produces the same `batch_sha256`.

## Verifying a batch

The hash covers `{format, org_id, entries}` only — it excludes
`generated_at` and the hash field itself, so anyone can recompute it:

```js
import { createHash } from "node:crypto";

const m = await (await fetch(
  "https://raw.githubusercontent.com/3mmanson/eucp-anchor-log/main/anchors/2026-11-18/070828-80e1e0d7.json",
)).json();

const recomputed = createHash("sha256")
  .update(JSON.stringify({ format: m.format, org_id: m.org_id, entries: m.entries }))
  .digest("hex");

console.log(recomputed === m.batch_sha256); // true
```

Each entry's `evidence_sha256` is the SHA-256 of that check's raw evidence as
stored by the platform; the platform's export bundle ships the same value, so
an auditor can cross-check bundle against log.

## Append-only policy

**This log is never rewritten.** Entries are added, never edited or removed —
that immutability is the whole point of the log.

**Early entries include demo data.** The platform seeds demonstration
compliance checks, and batches committed before fixture exclusion was
introduced contain hashes of that demo data. Those entries are retained
rather than rewritten: deleting or amending a commit would break the
append-only guarantee every later entry depends on. Anchoring now excludes
seeded fixtures (`raw_evidence.__fixture = true`), so batches from that point
forward contain live scan results only. An auditor should treat early batches
accordingly — the platform's export marks the affected checks as
`"fixture": true`.

## What an entry proves

A manifest proves that, at the commit's timestamp, the platform held evidence
whose SHA-256 appears in the batch for the stated `org_id` and control. It
does not certify that the underlying evidence was correct — only that it
existed and has not changed since it was logged.
