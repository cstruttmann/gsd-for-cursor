# GSD Date Handling

**Purpose:** Ensure all dates in GSD documents are correct. AI models often hallucinate or miscalculate dates. This reference mandates programmatic date resolution.

---

## Rule: Never Guess the Date

When writing any GSD document that includes a date (Last updated, Defined, Gathered, Analysis Date, etc.):

1. **Run the date command first** — Do not infer, estimate, or use training data.
2. **Use the exact output** — Copy the command output verbatim into the document.
3. **Pass to subagents** — Subagents do not receive user_info; include the current date in their prompt.

---

## Commands to Use

| Format | Command | Example Output | Use For |
|--------|---------|----------------|---------|
| ISO (YYYY-MM-DD) | `date +%Y-%m-%d` | 2025-02-19 | Traceability, Defined, footers |
| Human-readable | `date "+%B %d, %Y"` | February 19, 2025 | Last updated, Gathered |
| ISO timestamp | `date -u +%Y-%m-%dT%H:%M:%SZ` | 2025-02-19T14:30:00Z | UAT frontmatter, timestamps |

**Run before writing.** Example:

```bash
date +%Y-%m-%d
# Output: 2025-02-19
```

Then use `2025-02-19` exactly in the document.

---

## Where Dates Appear

| Document | Field | Format |
|----------|-------|--------|
| PROJECT.md | Last updated | `*Last updated: [date] after [trigger]*` |
| REQUIREMENTS.md | Defined, Last updated | `**Defined:** [date]`, `*Last updated: [date]*` |
| ROADMAP.md | (if any) | Same as above |
| STATE.md | Project Reference | `updated [date]` |
| CONTEXT.md | Gathered | `**Gathered:** [date]` |
| Research files | (if any) | ISO or human-readable |
| UAT.md | frontmatter.updated | ISO timestamp |
| Codebase map | Analysis Date | `**Analysis Date:** YYYY-MM-DD` |

---

## For Orchestrators (Spawning Subagents)

Subagents do **not** receive `user_info` (including "Today's date"). You must pass the date explicitly.

**Before spawning any subagent that will write documents:**

1. Run: `date +%Y-%m-%d` (or `date "+%B %d, %Y"` for human format)
2. Capture the output
3. Include in the Task prompt:

```
<current_date>
Current date: 2025-02-19
Use this exact date for all [date] placeholders, Last updated, Defined, Gathered, etc. Never guess or infer.
</current_date>
```

Replace `2025-02-19` with the actual output from step 1.

---

## Checklist

Before writing any document with date fields:

- [ ] Ran `date +%Y-%m-%d` (or appropriate format)
- [ ] Used the exact output — no rounding, no "approximately"
- [ ] If spawning subagents: included `<current_date>` block in their prompt
