# purview-action-update

Updates Microsoft Purview Compliance Manager improvement actions from Paramify.

Compliance Manager has **no write API** for improvement actions. The only
supported way to change them in bulk is to export the workbook, edit it, and
upload it back. This tool does the edit. A person does the export and the
upload.

| Step | Who |
|---|---|
| Export the Action Update workbook from Purview | You |
| Read Paramify and fill in three columns | The tool |
| Review what changed | You |
| Upload the workbook back into Purview | You |

It fills exactly three columns and leaves everything else untouched:

| Purview column | Where the value comes from |
|---|---|
| `Implementation Status` | Paramify audit log — the last status change |
| `Implementation Date` | Paramify audit log — that same change's timestamp |
| `Implementation Notes` | The Solution Capability narrative, as plain text |

> The tool never connects to Microsoft 365, Entra or Purview. It reads Paramify
> only. The workbook is the one thing that moves between the two systems, and
> you move it by hand.

**Status.** Used in production against real CMMC Level 2 programs, and
maintained. It is deliberately semi-automated: the export and the re-upload are
manual because Compliance Manager offers no API for either. Issues and pull
requests are welcome.

## How rows are matched

Each improvement action is matched to the Solution Capability with **exactly the
same name**. Nothing else is used — not row order, not the Action Id, not the
count.

**The two systems need not hold the same number of entries.** A Paramify program
covers the improvement actions it covers; the rest of the assessment simply has
no capability, and those rows are left exactly as they were. Every row without a
match, and every capability without a row, is counted in the run report.

---

## Before you start

- A **Paramify API token** for the workspace holding this program.
  - Read-only: `View Solution Capabilities`, `View Evidences`, `View Audit Logs`
  - To also write the workbook to a Paramify Evidence Set: add `Write Evidences`
- Permission to **export** from Compliance Manager and to use **Update actions**
- **Python 3.10+** and `git`
- Outbound HTTPS to Paramify

## Install

```bash
git clone https://github.com/paramify/purview-action-update.git
cd purview-action-update
python3 -m venv .venv && source .venv/bin/activate
pip install -e .
```

```bash
export PARAMIFY_API_TOKEN='your-token-here'
```

Add `PARAMIFY_API_BASE_URL` only if you are not on production — for example
`https://stage.paramify.com/api/v0`. A valid token against the wrong instance
fails as a `401` indistinguishable from a bad token.

Confirm the connection before anything else:

```bash
purview-action-update --check-auth
```

You should see `"authenticated": true`, the base URL you expected, and a
`solution_capabilities_visible` count. **That count does not need to equal the
number of improvement actions** — matching is by name. Only `0`, or a number
obviously wrong for the program, suggests the wrong workspace.

### A note on credentials

The token is read from the environment, or from a `.env` file in the working
directory. It is never written to disk, never logged, and never appears in the
run report. `.env` is gitignored here — keep it that way, and prefer a short-lived
token scoped to a single workspace.

## Where to keep the workbooks

One setting puts the working area wherever you want it. The folder may live
anywhere — home, Desktop, a shared drive.

```bash
export PURVIEW_WRITEBACK_DIR=~/purview-writeback
```

`inbox/` and `runs/` are created on first run:

```
purview-writeback/
    inbox/
        ExportActions.xlsx                  <- the newest export, always here
    runs/
        run-2026-10-02T14-02-11Z/
            ExportActions.updated.xlsx      <- upload this one to Purview
            run_report.json
        run-2026-10-05T09-44-03Z/
```

| Folder | What goes in it |
|---|---|
| `inbox/` | **The newest export from Purview, and nothing else.** Save each export here as `ExportActions.xlsx`, replacing the previous one |
| `runs/` | The tool's output. A new dated folder every run, so nothing is overwritten. Leave these alone |

> **The one rule.** Files move in one direction only: Purview → `inbox/`, and
> `runs/` → Purview. Never copy a file from `runs/` into `inbox/` — that is a
> snapshot of Purview from an earlier day, and re-using it would undo anything
> changed since. The tool stamps its own output and refuses to accept it as input.

To avoid retyping the token and path in every terminal window, add both `export`
lines to `~/.zshrc`.

---

## Running it

**1. Export from Compliance Manager** → Assessments → the assessment → Export.

It must be the **Assessments** page. The Improvement actions page produces a
shorter file missing the columns this tool writes; the run stops and says so.
Save it into `inbox/` as `ExportActions.xlsx`.

**2. Preview.** Reads Paramify, writes nothing.

```bash
purview-action-update --dry-run --print-changes-brief
```

**3. Produce the workbook.** Your export is never modified; a copy goes to a new
dated folder.

```bash
purview-action-update --print-changes-brief
```

The last line should read `verification PASS — the workbook satisfies Purview's
rules`. **If it says FAIL, do not upload it.**

**4. Read the run report.** Every cell the tool declined to write is listed with
its reason. This is the part worth reading.

**5. Upload** → Compliance Manager → Assessments → **Update actions** → the
`ExportActions.updated.xlsx` from the newest folder in `runs/`.

**6. Optional — file it in Paramify** as evidence. Needs `Write Evidences`.

```bash
purview-action-update --upload
```

---

## Options

The defaults are the cautious ones: fill blanks, change nothing that already has
a value, refuse anything ambiguous. Decide these once for your program, then
keep using the same flags.

| Flag | Effect |
|---|---|
| `--mode fill-empty` *(default)* | Write only into blank cells |
| `--mode sync` | Also replace cells that disagree with Paramify |
| `--notes-policy follow-mode` *(default)* | Notes follow `--mode` |
| `--notes-policy append` | Keep the note already in the workbook, add the narrative beneath. Idempotent |
| `--notes-policy replace` | The narrative overwrites the cell |
| `--on-date-conflict skip` *(default)* | Leave rows alone where the new date is after the Test Date |
| `--on-date-conflict advance-test` | Move Test Date to match |
| `--on-date-conflict clear-test` | Clear Test Date and Test Status — the control needs retesting |
| `--status-fallback-solcap` | Where no audit activity exists, use the capability's current status (no date) |
| `--timezone ZONE` | Defaults to this machine's zone — see below |

<details>
<summary>Paths, Paramify, diagnostics and cleanup</summary>

| Flag | Effect |
|---|---|
| `--writeback-dir PATH` | The working area. Also `PURVIEW_WRITEBACK_DIR` |
| `--workbook` / `--out` | Override either path individually |
| `--no-run-dir` | Write into `--out` directly instead of a dated subfolder |
| `--dry-run` | Plan and report, write nothing |
| `--print-changes` / `--print-changes-brief` | List written cells; brief omits the long narratives |
| `--check-auth` | Validate the token; reports the auth scheme and base URL |
| `--base-url` | Override `PARAMIFY_API_BASE_URL` |
| `--probe-audit` / `--probe-values` | Dump the audit-log shape. Keys and counts; values only with the second flag, and only short ones |
| `--audit-start YYYY-MM-DD` | Lower bound on the audit scan |
| `--activity-types` | Audit activity types to scan (default `HISTORY`) |
| `--upload` | Attach the workbook **and** run report to an Evidence Set |
| `--evidence-reference-id` / `--evidence-name` / `--evidence-id` | Target a different Evidence Set |
| `--list-artifacts [EVIDENCE]` | Show what is attached, to confirm an upload landed |
| `--allow-no-changes` | Upload even when nothing changed |
| `--min-match-rate` | Refuse to upload below this name-match fraction. Off by default |
| `--accept-near-matches` | Join names matching only after case/whitespace normalization |
| `--strict-test-status` | Refuse rows with a blank Test Status where the status does not permit `None` |
| `--allow-chained-input` | Accept a workbook this tool produced (refused by default) |
| `--prune-no-op-artifacts` / `--delete-evidence-set` | Cleanup; both need `--confirm` |

</details>

### Dates

Only the calendar date has to be right, so dates render in **the operator's own
timezone, not UTC**. An audit event at `2026-10-02T02:00Z` is still 1 October
everywhere in the US; rendered in UTC it would be written as the 2nd.

Resolution order: `--timezone` → `TZ` → the system zone from `/etc/localtime` →
the OS's current UTC offset → UTC. The report records which was used.

---

## Reading the output

| Line | Meaning |
|---|---|
| `exact name matches` | How many rows found their capability |
| `cells written` | What changed, by column |
| `cells held back` | Proposed but declined — reasons in the report |
| `audit activity found` | Capabilities whose status change the tool could read |
| `evidence set` | Whether anything reached Paramify |
| `verification` | `PASS` means the file satisfies Purview's rules. `FAIL` means do not upload |

Lines beginning `!` are advisories — things that *were* written but deserve a
look. Not errors.

To confirm an upload after the fact:

```bash
purview-action-update --list-artifacts
```

### If something goes wrong

| Message | Cause |
|---|---|
| Rejected the token under both auth schemes | Wrong or expired token, or the right token for a different instance |
| Missing required column | Exported from the Improvement actions page. Re-export from Assessments |
| Refusing to run: produced by this tool | You passed a previous run's output. Export a fresh one |
| NOTHING matched by name | The token points at a different workspace |
| Refusing to upload: the run changed no cells | Nothing needed changing |
| `verification FAIL` | Do not upload. Send the run report |

`run_report.json` records every decision and every reason, and contains no
credentials. It is the single most useful thing to attach to an issue.

---

## What it will never do

Enforced in code, not left to care. Every refusal is recorded with its reason.

- Touch your Microsoft tenant — it holds no Microsoft credentials
- Modify your original export — it always writes a copy
- Write any column beyond the three (five, under `--on-date-conflict`)
- Blank a cell because Paramify had nothing to say
- Write a status for `PARTIALLY_IMPLEMENTED` or `NOT_SET` — neither has an honest
  Purview equivalent
- Write a status that invalidates the row's Test Status, or a date later than it
- Produce a workbook Purview would reject — it re-reads the finished file and
  checks it against Purview's own rules, and `--upload` refuses a failing one
- Accept its own output as input, so a stale snapshot cannot be re-uploaded
- Guess

---

## Notes on the Paramify API

Two things are not in the published OpenAPI document, both established by probe:

- **`GET /audit-logs` is served but undocumented.** A nonexistent path returns
  404 even unauthenticated while this returns 401, so routing precedes auth and
  the route exists. A 404 is still handled as degraded rather than fatal.
- **Its change entries carry only `old`/`new`, with no field name.** A status
  change is therefore identified by *both* sides being members of Paramify's
  status vocabulary — requiring the pair, so a field edited *to* the literal text
  `IMPLEMENTED` is not misread as a transition.

**A freshly copied workspace has no modification history at all** — copying
recreates every record, leaving the status history in the source. `--probe-audit`
diagnoses that case by name.

There are also **two auth schemes** (`Authorization: Bearer`, and a legacy header
literally named `Bearer`). The client tries the current one, falls back, and pins
whichever authenticates.

### One unverified reading

The rules tab constrains Test Status by Implementation Status and does not list
`None` among the values permitted alongside `Implemented`. It is silent on
whether a **blank** cell is allowed, and no export seen so far settles it.

This tool takes the permissive reading — a blank cell is an absence, not a value
— and counts every affected row under `advisories.blank_test_status`. The first
re-upload confirms it; `--strict-test-status` excludes them.

---

## Development

```bash
pip install -e '.[dev]'
pytest -q
```

**150 tests, fully offline** against synthetic fixtures. No tenant data in the
repo, and none should ever be added. The end-to-end tests need a real Purview export and
skip without one; point `PURVIEW_EXPORT` at a workbook to run them.

| File | Role |
|---|---|
| `config.py` | Purview field rules from the workbook's own rules tab; base URL and timezone resolution |
| `mapping.py` | Pure transforms — status map, date format, notes rendering, conflict checks |
| `matching.py` | Action Title ↔ SolCap name join, and match quality |
| `audit.py` | Audit log → implementation status and date, plus the shape probe |
| `planner.py` | Per-row decisions: what to write, what to refuse, what to flag |
| `workbook.py` | Read the tab, apply writes, save a copy, stamp provenance |
| `verify.py` | Re-read the produced file and check it against Purview's rules |
| `report.py` | `run_report.json`, the terminal summary, the changes table |
| `paramify_api.py` | Read-only client plus evidence upload; dual auth scheme |
| `upload.py` | Evidence Set, artifacts, and gated cleanup |
| `run.py` | CLI |

### Contributing

Issues and pull requests welcome. Two things make a change easy to accept:

- **A test that fails before the change and passes after.** The suite is the
  specification, particularly for the refusals — most of them exist because a
  real run got something wrong.
- **No tenant data.** Fixtures are synthetic. If a bug needs real data to
  reproduce, describe the shape rather than attaching the file.

The design principle throughout: **decide, then apply.** The planner produces
proposed changes with a written/declined flag and a reason; only then does
anything touch the workbook. That is why every refusal can explain itself.

---

## Licence

GPL-3.0-only. See [LICENSE](LICENSE).
