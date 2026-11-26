# Formats

Each reviewer ends its final message with one fenced ```json block. The orchestrator saves it as
`main.json`, `debate.json`, or `final.json` in the run directory and renders `final.json` into
`report.md`. Nothing is ever posted to GitHub.

## 1. `debate-review.findings.v1` (main pass)

```json
{
  "schema": "debate-review.findings.v1",
  "head": "<head sha>",
  "context_check": "which issues, LEP, related PRs were read; what was missing",
  "issue_alignment": "fully-aligned | partially-aligned | not-aligned | unable-to-verify",
  "alignment_note": "the issue requirement and the code that meets or misses it",
  "testing_status": "what the tests pin, what is untested",
  "verdict": "approve | needs-attention",
  "summary": "ship/no-ship read, under 250 words",
  "findings": [
    {
      "id": "F1",
      "commit": "<sha7 of the commit that introduced it, if one does>",
      "file": "controller/volume_controller.go",
      "line_start": 412,
      "line_end": 415,
      "severity": "critical | major | minor",
      "axis": "data-integrity | concurrency | compatibility | correctness | error-handling | resource | security | spec | scope | standards | tests | docs",
      "claim": "what is wrong, one sentence",
      "evidence": "trigger and wrong result, with the quoted line or grep output",
      "recommendation": "concrete change",
      "confidence": 0.8
    }
  ]
}
```

`file` and lines refer to the file **at the head**, even when the finding is about one commit.
`critical` and `major` are blocking.

## 2. `debate-review.debate.v1` (debate pass)

```json
{
  "schema": "debate-review.debate.v1",
  "head": "<head sha>",
  "verdicts": [
    { "id": "F1", "verdict": "confirm | refute | downgrade", "reason": "one sentence", "evidence": "file:line or quoted code" }
  ],
  "new_findings": [ ]
}
```

One verdict per `F*`. `new_findings` use the finding shape with ids `D1`, `D2`, and only `critical`
or `major`. Zero new findings is normal.

## 3. `debate-review.final.v1` (final pass)

The same top-level fields as section 1 except `verdict`, re-decided after the debate, with
`"schema": "debate-review.final.v1"`. Each finding keeps its id and fields and adds:

```json
{ "status": "agreed | contested | withdrawn", "debate_note": "one sentence: what the challenge said and why the finding stands, moved, or was dropped" }
```

`agreed`: both passes stand behind it. `contested`: the debate refuted it and the main reviewer holds
it with evidence. `withdrawn`: listed only in the report's appendix. A `D*` is only `agreed` or `withdrawn`. Every `F*` and
`D*` must come back.

## 4. Report (`report.md`)

| Severity | Level |
| --- | --- |
| critical | P0 |
| major | P1 |
| minor | P2 |

```
# debate-review: <owner>/<repo>#<n> (or local branch <name>)

Head `<sha7>`, base `<sha7>`. Checkout: `<checkout path>`. Main `<model>`, debate `<model>`, final `<model>`.
P0 <a>, P1 <b>, P2 <c>, contested <m>, withdrawn <w>.

## 1. Context check
<context_check>

## 2. Issue alignment
<Fully aligned | Partially aligned | Not aligned | Unable to verify>. <alignment_note>

## 3. Key findings

### P0 Critical, <axis>: <claim>
`<file>:<line_start>-<line_end>` at head, from commit `<sha7>` (agreed | contested; outside the PR's files)

<evidence>

Suggested: <recommendation>

_<debate_note>_

### P1 ...   (P0 first, then P1, then P2; "None." if empty)

## 4. Assessment
<summary>

## 5. Testing status
<testing_status>

## Appendix: withdrawn during the debate
- `<file>:<line>` <claim>. <debate_note>
```

For a contested finding, add under its heading: "The second reviewer disagreed; the main reviewer
holds it, reasons below."
