---
name: 1901-resolve-production-source
description: Identifies the exact approved artwork file a 1901 design should use as its render source; read-only.
---

# 1901 Resolve Production Source

Answers one question for one 1901 Main Street design: "What exact approved
artwork file should this design use as `render_source_path`?" It reads the
live Idea Queue, the governing documentation, and the relevant Drive files, and
returns one JSON object naming the file, the candidates, or the reason no file
can be named. It never writes `render_source_path`, never changes a record,
and never returns a readiness verdict.

Core rule: **VERIFY, DON'T ASSUME.** A source is resolved only when exactly one
file can be tied to the requested design, to the human-approved artwork, and to
the current governing production-source rule. Never choose a file because it
looks likely, has "final" or "PRINT" or "keeper" in its name, is the largest,
is the newest, sits alone in a folder, or looks similar. If the evidence is
ambiguous, return ambiguity.

## When to Use

Trigger on requests such as:

- "Which file should 1901-003 render from?"
- "Resolve the production source for <design_id>"
- "What is the approved master for this design?"
- "Is art_path the right render source for 1901-017?"
- Before anyone proposes a `render_source_path` value for a design.

Do not use it to judge readiness (`1901-validate-readiness`), to fetch a queue
row on its own (`1901-read-idea-queue`), or to look up several designs. Those
skills stay separate; this one may use live queue data and governing docs as
evidence, nothing more.

## Authoritative Sources

| Source | Identifier |
|---|---|
| Idea Queue spreadsheet | `1UxnZsA9aWlxZHqMAt17_7HAe86w_cHcMik3xpXQrfq0` |
| Idea Queue worksheet | `Idea Queue`, sheet id `1283408381` |
| Governing docs entry point | `00 Foundations (CURRENT)`, Drive folder `1Jvxd4MwiBfpeKYkM9e8U_xEcmjNt1z1N` |
| README / Index | Doc `1ukOidU0NnUJJ7IjRSF9XFgVgQsZt0jqCUhtHpL5lZuA` |

Relevant governing docs include `00 — 1901 Concept & Prompt Standard
(CURRENT)`, `01 — 1901 Listing Render System (CURRENT)`, and the current
Pipeline README. Start from the entry point and follow what it identifies as
governing. Proposed, draft, unverified, stale, or superseded documents never
override a current governing document.

Read everything with the read-only Sheets and Drive access this bot actually
has. Live values only. Memory, chat history, and prior runs are not evidence.
If a required source cannot be read, the result is `SOURCE_UNAVAILABLE`; do
not substitute anything.

## The Governing Source Rule

There is a known unresolved production-source and upscale question in the
1901 documentation. Before judging any file, determine from the current
governing docs which kind of file the render stage consumes: the approved
artwork master, an upscaled print file, a derived production export, or
something else. Record the answer in `source_rule`:

| `source_rule.status` | meaning |
|---|---|
| `VERIFIED` | A current governing document states the rule; name it and summarise it |
| `UNRESOLVED` | The governing docs do not settle it, or conflict without a clear governing authority |
| `UNAVAILABLE` | The governing docs could not be read, or were not reached |

`UNRESOLVED` ends the run with `DOCUMENTATION_CONFLICT`. Do not guess the rule,
and do not judge files without it.

## Input

Exactly one `design_id`, for example `1901-003`. Trim surrounding whitespace
from the user's input, and nothing else. Match the id against the queue by
exact string equality: no case folding, no fuzzy match, no closest id, no
normalising `1901-3` into `1901-003`. No usable single id: `INVALID_RECORD`.

## Procedure

Results are checked in this order; the first that applies is returned.

1. **Queue.** Read the exact live Idea Queue row: header row first, whole `id`
   column, exact match. Running `1901-read-idea-queue` in this run is an
   acceptable way to do it. Queue unreadable: `SOURCE_UNAVAILABLE`. No row,
   more than one row, or a materially malformed row: `INVALID_RECORD`.
2. **Queue evidence.** Read `art_path`, `render_source_path`, `idea_notes`,
   `notes`, `status`, `human_decision`, and any other production-source
   field present. Put them in `queue_evidence` and `evidence`. Do not
   interpret `status` or `human_decision`; they are context only.
3. **Rule.** Read the current governing documentation that defines the
   render-stage source (previous section). Docs unreadable:
   `SOURCE_UNAVAILABLE`. Rule not confidently determined:
   `DOCUMENTATION_CONFLICT`.
4. **Follow `art_path`.** When present, open what it names. A file: inspect
   it and, when needed, its containing folder and clearly related sibling
   files. A folder: inspect its contents. Drive location unreadable:
   `SOURCE_UNAVAILABLE`. Do not scan unrelated Drive areas unless the live
   record or a governing document points there.
5. **Weigh each candidate.** File names, Drive ids, folders, metadata, and
   recorded human-approval references are supporting evidence only. For each
   file decide: is it plausible for this design, is it tied to the
   human-approved artwork by something recorded, and is it the kind of file
   the governing rule requires.
6. **Decide.**
   - Exactly one file is tied to the approval and matches the rule:
     `RESOLVED`. Return that file only.
   - Two or more plausible files remain and nothing recorded distinguishes
     one: `AMBIGUOUS`. Return every materially plausible candidate; do not
     rank unless evidence actually distinguishes.
   - Exactly one plausible file, but nothing proves it is the approved
     source: `UNVERIFIED`.
   - The evidence says what should exist and no qualifying file can be found:
     `NOT_FOUND`.
7. **Return** the JSON object. Change nothing anywhere.

## Important Distinctions

A queue `art_path` is evidence, not automatically the answer. Never assume
`art_path == render_source_path`; verify that the `art_path` file is the exact
approved source the governing rule requires. Likewise:

- "keeper" does not mean production master
- "PRINT" in a filename does not make a file authoritative
- highest resolution does not win
- newest file does not win
- a lone file in a folder is not thereby approved

## Output

Return exactly one JSON object:

```json
{
  "design_id": "1901-003",
  "result": "RESOLVED | AMBIGUOUS | NOT_FOUND | UNVERIFIED | DOCUMENTATION_CONFLICT | INVALID_RECORD | SOURCE_UNAVAILABLE",
  "source_rule": { "status": "VERIFIED | UNRESOLVED | UNAVAILABLE", "governing_document": "", "rule_summary": "" },
  "queue_evidence": { "row_number": null, "art_path": "", "render_source_path": "", "idea_notes": "", "notes": "" },
  "resolved_file": null,
  "candidates": [],
  "evidence": [],
  "human_action_required": null
}
```

Shape rules:

- `queue_evidence` holds live values: `""` for a blank cell, `null` for a
  column absent from the live schema, and all `null` when no row was read.
- `resolved_file` is present only for `RESOLVED`, as
  `{drive_file_id, name, url, folder, mime_type, reason}`; `candidates` is
  then empty.
- `candidates` is non-empty only for `AMBIGUOUS`, each as
  `{drive_file_id, name, url, reason}` where `reason` says why it remains
  plausible.
- `evidence` lists only facts actually read in this run, each as
  `{source, fact}`, for example `{"source": "Idea Queue row 4", "fact":
  "art_path points to file id ..."}` or `{"source": "01 — 1901 Listing Render
  System (CURRENT)", "fact": "render source must be ..."}`.
- `human_action_required`: normally `null` for `RESOLVED`. For `AMBIGUOUS`,
  the human decision or record clarification needed. For `UNVERIFIED`, the
  approval linkage or governing evidence that is missing. For
  `DOCUMENTATION_CONFLICT`, that Jody must resolve the governing
  production-source rule before this design can be resolved. Never tell
  Walter to write `render_source_path`.

## Results

| result | meaning |
|---|---|
| `RESOLVED` | Exactly one Drive file is tied to the design, the human-approved artwork, and the governing rule |
| `AMBIGUOUS` | Two or more plausible files remain; evidence does not uniquely identify one; none chosen |
| `NOT_FOUND` | The evidence identifies what should exist, but no qualifying file can be found |
| `UNVERIFIED` | A likely file exists, but nothing proves it is the approved source |
| `DOCUMENTATION_CONFLICT` | The governing render-stage source rule cannot be confidently determined |
| `INVALID_RECORD` | The design record is missing, duplicated, or materially malformed |
| `SOURCE_UNAVAILABLE` | Required live Sheets, Drive, or governing docs cannot be read; nothing substituted |

## Never Do

This skill is read-only. Never:

- write to Google Sheets, write `render_source_path`, or change
  `human_decision`, `status`, or `render_status`
- write to Google Drive: no upload, move, rename, copy, or delete
- generate a new production file, upscale, resize, convert, or edit
  transparency
- render mockups, call Printify, call Etsy, or publish anything
- infer human approval
- resolve ambiguous candidates by preference
- use chat history or memory as evidence
- modify governing documentation
- invoke any mutation workflow, or return a readiness verdict

## Pitfalls

- **`art_path` names a file, so that is the answer.** It is evidence. Confirm
  it is the approved artwork and the kind of file the rule requires.
- **The note says "Redraw C stamp remaster" and two Redraw C files exist.**
  `AMBIGUOUS` unless something recorded ties the approval to one of them. If
  only one file plausibly matches but nothing links it to the approval,
  `UNVERIFIED`.
- **`redraw-c-stamp-PRINT.png` is the biggest and newest.** Size and date are
  not approval. Whether a print export can be the render source is decided by
  the governing rule, not the filename.
- **One file in the folder, name looks right.** Alone is not approved.
  `UNVERIFIED` without a recorded linkage.
- **The docs disagree on master versus print file.** Do not pick the sensible
  one. `DOCUMENTATION_CONFLICT`, and say Jody must resolve it.
- **You remember which file Jody approved last week.** Memory is not
  evidence. Only what is recorded and read in this run counts.

## Known 1901 Edge Case

Design `1901-003` may have several related files, such as `original-stamp`,
`redraw-a-circular`, `redraw-b-stamp`, `keeper-square`, `redraw-c-stamp`, and
`redraw-c-stamp-PRINT`. Nothing here says which is correct; the live record and
the governing evidence decide. If the evidence only says "Redraw C stamp
remaster" and more than one Redraw C file remains plausible, return
`AMBIGUOUS`, or `UNVERIFIED` when only one plausibly matches but is not tied to
the approval.

## Examples

Values are fixtures showing shape and logic. File ids, folder ids, and names
are illustrative. Live output always carries what was actually read.

### 1. RESOLVED

`art_path` names one file, the queue note and approval record identify it, and the governing rule excludes the sibling print export.

```json
{
 "design_id": "1901-003",
 "result": "RESOLVED",
 "source_rule": {
  "status": "VERIFIED",
  "governing_document": "01 — 1901 Listing Render System (CURRENT)",
  "rule_summary": "The render stage consumes the human-approved artwork master file recorded for the design; derived print exports are not the render source."
 },
 "queue_evidence": {
  "row_number": 4,
  "art_path": "https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view",
  "render_source_path": "",
  "idea_notes": "Stamp concept, circular and square explorations",
  "notes": "Approved: Redraw C stamp remaster"
 },
 "resolved_file": {
  "drive_file_id": "1FILEc3stamp000000000000000000001",
  "name": "1901-003_redraw-c-stamp.png",
  "url": "https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view",
  "folder": "1901-003 Artwork (folder id 1AbCdEfGhIjKlMnOpQrStUvWxYz012345)",
  "mime_type": "image/png",
  "reason": "named by art_path; queue notes and the approval record identify Redraw C stamp remaster as the approved artwork; it is the master, not a derived export"
 },
 "candidates": [],
 "evidence": [
  {
   "source": "Idea Queue row 4",
   "fact": "art_path = 'https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view'; render_source_path = ''; notes = 'Approved: Redraw C stamp remaster'"
  },
  {
   "source": "01 — 1901 Listing Render System (CURRENT)",
   "fact": "The render stage consumes the human-approved artwork master file recorded for the design; derived print exports are not the render source."
  },
  {
   "source": "Drive folder 1901-003 Artwork (folder id 1AbCdEfGhIjKlMnOpQrStUvWxYz012345)",
   "fact": "1901-003_redraw-c-stamp.png (id 1FILEc3stamp000000000000000000001): named by art_path; queue notes and the approval record identify Redraw C stamp remaster as the approved artwork; it is the master, not a derived export"
  },
  {
   "source": "Drive folder 1901-003 Artwork (folder id 1AbCdEfGhIjKlMnOpQrStUvWxYz012345)",
   "fact": "1901-003_redraw-c-stamp-PRINT.png (id 1FILEc3print000000000000000000002): sibling print export; governing rule excludes derived print exports from the render source"
  }
 ],
 "human_action_required": null
}
```

### 2. AMBIGUOUS: two plausible Redraw C files

`art_path` names the folder. The approval note says "Redraw C stamp remaster" and two files match parts of it. Nothing recorded picks one.

```json
{
 "design_id": "1901-003",
 "result": "AMBIGUOUS",
 "source_rule": {
  "status": "VERIFIED",
  "governing_document": "01 — 1901 Listing Render System (CURRENT)",
  "rule_summary": "The render stage consumes the human-approved artwork master file recorded for the design; derived print exports are not the render source."
 },
 "queue_evidence": {
  "row_number": 4,
  "art_path": "https://drive.google.com/drive/folders/1AbCdEfGhIjKlMnOpQrStUvWxYz012345",
  "render_source_path": "",
  "idea_notes": "Stamp concept, circular and square explorations",
  "notes": "Approved: Redraw C stamp remaster"
 },
 "resolved_file": null,
 "candidates": [
  {
   "drive_file_id": "1FILEc3stamp000000000000000000001",
   "name": "1901-003_redraw-c-stamp.png",
   "url": "https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view",
   "reason": "matches 'Redraw C stamp' in the approval note; nothing recorded ties the approval to this file rather than the remaster"
  },
  {
   "drive_file_id": "1FILEc3remas000000000000000000003",
   "name": "1901-003_redraw-c-stamp-remaster.png",
   "url": "https://drive.google.com/file/d/1FILEc3remas000000000000000000003/view",
   "reason": "matches 'remaster' in the approval note; nothing recorded ties the approval to this file rather than the earlier redraw"
  }
 ],
 "evidence": [
  {
   "source": "Idea Queue row 4",
   "fact": "art_path = 'https://drive.google.com/drive/folders/1AbCdEfGhIjKlMnOpQrStUvWxYz012345'; render_source_path = ''; notes = 'Approved: Redraw C stamp remaster'"
  },
  {
   "source": "01 — 1901 Listing Render System (CURRENT)",
   "fact": "The render stage consumes the human-approved artwork master file recorded for the design; derived print exports are not the render source."
  },
  {
   "source": "Drive folder 1901-003 Artwork (folder id 1AbCdEfGhIjKlMnOpQrStUvWxYz012345)",
   "fact": "1901-003_redraw-c-stamp.png (id 1FILEc3stamp000000000000000000001): matches 'Redraw C stamp' in the approval note; nothing recorded ties the approval to this file rather than the remaster"
  },
  {
   "source": "Drive folder 1901-003 Artwork (folder id 1AbCdEfGhIjKlMnOpQrStUvWxYz012345)",
   "fact": "1901-003_redraw-c-stamp-remaster.png (id 1FILEc3remas000000000000000000003): matches 'remaster' in the approval note; nothing recorded ties the approval to this file rather than the earlier redraw"
  },
  {
   "source": "Drive folder 1901-003 Artwork (folder id 1AbCdEfGhIjKlMnOpQrStUvWxYz012345)",
   "fact": "1901-003_redraw-c-stamp-PRINT.png (id 1FILEc3print000000000000000000002): print export; excluded by the governing rule"
  }
 ],
 "human_action_required": "Jody or Ame: record which of the listed files is the approved artwork for this design (in the queue notes or the governing approval record), then re-run."
}
```

### 3. UNVERIFIED: one likely file, no approval linkage

The folder holds one artwork file. Nothing ties it to the human-approved artwork.

```json
{
 "design_id": "1901-011",
 "result": "UNVERIFIED",
 "source_rule": {
  "status": "VERIFIED",
  "governing_document": "01 — 1901 Listing Render System (CURRENT)",
  "rule_summary": "The render stage consumes the human-approved artwork master file recorded for the design; derived print exports are not the render source."
 },
 "queue_evidence": {
  "row_number": 12,
  "art_path": "https://drive.google.com/drive/folders/1FoLdEr011000000000000000000000000",
  "render_source_path": "",
  "idea_notes": "",
  "notes": ""
 },
 "resolved_file": null,
 "candidates": [],
 "evidence": [
  {
   "source": "Idea Queue row 12",
   "fact": "art_path = 'https://drive.google.com/drive/folders/1FoLdEr011000000000000000000000000'; render_source_path = ''; notes = ''"
  },
  {
   "source": "01 — 1901 Listing Render System (CURRENT)",
   "fact": "The render stage consumes the human-approved artwork master file recorded for the design; derived print exports are not the render source."
  },
  {
   "source": "Drive folder 1901-003 Artwork (folder id 1AbCdEfGhIjKlMnOpQrStUvWxYz012345)",
   "fact": "1901-011_final.png (id 1FILE011final0000000000000000000004): only artwork file in the folder art_path names; no queue note, approval record, or governing reference ties it to the human-approved artwork"
  }
 ],
 "human_action_required": "Record the approval linkage tying 1901-011_final.png to the human-approved artwork for 1901-011 in the governing record, then re-run."
}
```

### 4. NOT_FOUND

The note names what was approved, but the file `art_path` points at is gone and the folder holds nothing for the design.

```json
{
 "design_id": "1901-014",
 "result": "NOT_FOUND",
 "source_rule": {
  "status": "VERIFIED",
  "governing_document": "01 — 1901 Listing Render System (CURRENT)",
  "rule_summary": "The render stage consumes the human-approved artwork master file recorded for the design; derived print exports are not the render source."
 },
 "queue_evidence": {
  "row_number": 15,
  "art_path": "https://drive.google.com/file/d/1FILE014gone0000000000000000000005/view",
  "render_source_path": "",
  "idea_notes": "",
  "notes": "Approved: porch cat v2 master"
 },
 "resolved_file": null,
 "candidates": [],
 "evidence": [
  {
   "source": "Idea Queue row 15",
   "fact": "art_path = 'https://drive.google.com/file/d/1FILE014gone0000000000000000000005/view'; render_source_path = ''; notes = 'Approved: porch cat v2 master'"
  },
  {
   "source": "01 — 1901 Listing Render System (CURRENT)",
   "fact": "The render stage consumes the human-approved artwork master file recorded for the design; derived print exports are not the render source."
  },
  {
   "source": "Drive folder 1901-003 Artwork (folder id 1AbCdEfGhIjKlMnOpQrStUvWxYz012345)",
   "fact": "(art_path target) (id 1FILE014gone0000000000000000000005): the file id named by art_path does not exist or is not visible; the containing folder holds no artwork file for 1901-014"
  }
 ],
 "human_action_required": "Locate or restore the approved source file for 1901-014 that the governing rule requires, record its location in the queue, then re-run."
}
```

### 5. DOCUMENTATION_CONFLICT

The governing docs leave the master-versus-print-file question open. Files are not judged.

```json
{
 "design_id": "1901-003",
 "result": "DOCUMENTATION_CONFLICT",
 "source_rule": {
  "status": "UNRESOLVED",
  "governing_document": "",
  "rule_summary": "Governing docs do not settle whether the render stage consumes the approved master or the upscaled print file; the question is recorded as open."
 },
 "queue_evidence": {
  "row_number": 4,
  "art_path": "https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view",
  "render_source_path": "",
  "idea_notes": "Stamp concept, circular and square explorations",
  "notes": "Approved: Redraw C stamp remaster"
 },
 "resolved_file": null,
 "candidates": [],
 "evidence": [
  {
   "source": "Idea Queue row 4",
   "fact": "art_path = 'https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view'; render_source_path = ''; notes = 'Approved: Redraw C stamp remaster'"
  },
  {
   "source": "00 Foundations (CURRENT)",
   "fact": "Governing docs do not settle whether the render stage consumes the approved master or the upscaled print file; the question is recorded as open."
  }
 ],
 "human_action_required": "Jody must resolve the governing production-source rule (approved master vs upscaled print file vs derived export) before this design can be resolved."
}
```

### 6. INVALID_RECORD

No row carries the id exactly.

```json
{
 "design_id": "1901-999",
 "result": "INVALID_RECORD",
 "source_rule": {
  "status": "UNAVAILABLE",
  "governing_document": "",
  "rule_summary": ""
 },
 "queue_evidence": {
  "row_number": null,
  "art_path": null,
  "render_source_path": null,
  "idea_notes": null,
  "notes": null
 },
 "resolved_file": null,
 "candidates": [],
 "evidence": [
  {
   "source": "Idea Queue",
   "fact": "no row has id exactly equal to 1901-999"
  }
 ],
 "human_action_required": "Correct the Idea Queue record for 1901-999 so exactly one row carries this id, then re-run."
}
```

### 7. SOURCE_UNAVAILABLE

The queue row was read but the governing documentation could not be.

```json
{
 "design_id": "1901-003",
 "result": "SOURCE_UNAVAILABLE",
 "source_rule": {
  "status": "UNAVAILABLE",
  "governing_document": "",
  "rule_summary": ""
 },
 "queue_evidence": {
  "row_number": 4,
  "art_path": "https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view",
  "render_source_path": "",
  "idea_notes": "Stamp concept, circular and square explorations",
  "notes": "Approved: Redraw C stamp remaster"
 },
 "resolved_file": null,
 "candidates": [],
 "evidence": [
  {
   "source": "Idea Queue row 4",
   "fact": "art_path = 'https://drive.google.com/file/d/1FILEc3stamp000000000000000000001/view'; render_source_path = ''; notes = 'Approved: Redraw C stamp remaster'"
  },
  {
   "source": "00 Foundations (CURRENT)",
   "fact": "governing documentation could not be read in this run"
  }
 ],
 "human_action_required": "Restore read access to 00 Foundations (CURRENT) and its governing docs, then re-run."
}
```

## Verification

The skill worked if the reply is one JSON object in the shape above,
`source_rule` names a current governing document or explains why it could not,
every entry in `evidence` was read in this run, `resolved_file` is set only
for `RESOLVED` and `candidates` only for `AMBIGUOUS`, and no sheet cell, Drive
file, document, or external system changed during the run.
