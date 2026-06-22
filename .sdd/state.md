# SDD State — append-only verdict & audit log

> **Append-only.** Gatekeepers append one record per verdict (format below).
> Per-entity lifecycle `status` does **not** live here — it lives in the index
> rows (see `.sdd/conventions.md` §5). This file records *why* a decision was
> made; the indexes record *where each entity stands*. The driving slash command
> reads the **latest** record for a scope to advance status / re-invoke / escalate.

Record format (see `.sdd/conventions.md` §6):

```
## <ISO-8601 timestamp> — <gate-agent> — <PASS|REJECT>
- scope: <ids reviewed>
- phase: <analysis|code|test>
- iteration: <n>/<budget>
- verdict: PASS | REJECT
- reasons:
  - <blocking reason, citing the spec id / ACn / branch-arm it concerns>
- routing: <none | spec-writer | reuse-analyst | code-implementer | test-writer>
```

<!-- Verdict records are appended below this line. -->
