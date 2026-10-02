# purview-action-update

Updates Microsoft Purview Compliance Manager improvement actions from Paramify.

Compliance Manager has **no write API** for improvement actions. The only
supported way to change them in bulk is to export the workbook, edit it, and
upload it back. This tool does the edit. A person does the export and the
upload.

> [!NOTE]
> The tool never connects to Microsoft 365, Entra or Purview. It reads Paramify
> only. The workbook is the one thing that moves between the two systems, and
> you move it by hand.

**Status.** Used in production against real CMMC Level 2 programs, and
maintained. It is deliberately semi-automated: the export and the re-upload are
manual because Compliance Manager offers no API for either. Issues and pull
requests are welcome.

---

## Contents

1. [What this tool does](#1-what-this-tool-does)
2. [What you need before you start](#2-what-you-need-before-you-start)
3. [One-time setup](#3-one-time-setup)
4. [Where to keep the workbooks](#4-where-to-keep-the-workbooks)
5. [Running it](#5-running-it)
6. [Options you may want to change](#6-options-you-may-want-to-change)
7. [If something goes wrong](#7-if-something-goes-wrong)
8. [What this tool will never do](#8-what-this-tool-will-never-do)

Reference: [Notes on the Paramify API](#notes-on-the-paramify-api) ·
[Development](#development) · [Contributing](#contributing) ·
[Licence](#licence)

---

## 1. What this tool does

| Step | Who does it |
|---|---|
| Export the Action Update workbook from Purview | You |
| Read Paramify and fill in three columns | The tool |
| Review what changed | You |
| Upload the workbook back into Purview | You |

It fills in exactly three columns, and leaves everything else in the workbook
untouched:

| Purview column | Where the value comes from |
|---|---|
| `Implementation Status` | Paramify audit log — the last status change |
| `Implementation Date` | Paramify audit log — that same change's timestamp |
| `Implementation Notes` | The Solution Capability narrative, as plain text |

### How rows are matched

Each improvement action is matched to the Solution Capability with **exactly the
same name**. Nothing else is used — not row order, not the Action Id, not the
count.

**The two systems do not need to hold the same number of entries.** A Paramify
program covers the improvement actions it covers; the rest of the assessment
simply has no capability, and those rows are left exactly as they were. Every
row without a match, and every capability without a row, is counted in the run
report so you can see the coverage.

---

## 2. What you need before you start

- A **Paramify API token** for the workspace holding this program.
  - Read-only: `View Solution Capabilities`, `View Evidences`, `View Audit Logs`
  - To also write the workbook to a Paramify Evidence Set: add `Write Evidences`
- Permission to **export** from Compliance Manager and to use **Update actions**
  to upload.
- **Python 3.10 or newer** and **git** on the machine you will run this from.
- Outbound HTTPS access to Paramify from that machine.

> [!NOTE]
> The token is read from the environment, or from a `.env` file in the working
> directory. It is never written to disk, never logged, and never appears in the
> run report. `.env` is gitignored here — keep it that way, and prefer a
> short-lived token scoped to a single workspace.

---

## 3. One-time setup

You only do this once. Afterwards, skip to [section 5](#5-running-it) each time.

### 1. Download the code

Put it somewhere permanent.

```bash
git clone https://github.com/paramify/purview-action-update.git
cd purview-action-update
```

### 2. Set up Python

Keeps everything self-contained and off your system Python.

```bash
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -e .
```

That installs the `purview-action-update` command used below.

### 3. Add your Paramify token

This lasts for the current terminal window only. Repeat it in a new window, or
see the tip at the end of [section 4](#4-where-to-keep-the-workbooks).

```bash
export PARAMIFY_API_TOKEN='your-token-here'
```

Only if you are **not** on the production instance, also set the base URL. A
valid token against the wrong instance fails as a `401` that looks exactly like
a bad token:

```bash
export PARAMIFY_API_BASE_URL='https://your-instance/api/v0'
```

### 4. Check it is connected to the right place

Do this first every session. It makes one small read and changes nothing.

```bash
purview-action-update --check-auth
```

> [!TIP]
> **What you should see:** `"authenticated": true`, the base URL you expected,
> and a `solution_capabilities_visible` count.
>
> **That count does not need to equal the number of improvement actions.**
> Matching is done by name, not by number — see [section 1](#1-what-this-tool-does).
> Only a count of **0**, or one obviously wrong for this program, suggests the
> token is pointing at a different Paramify workspace.

---

## 4. Where to keep the workbooks

Set this up once. The folder can live **anywhere you like** — your home folder,
Desktop, a shared drive. Tell the tool where it is and it handles the rest, so
the commands in section 5 need no file paths at all.

```bash
export PURVIEW_WRITEBACK_DIR=~/purview-writeback
```

Change that path to wherever you want it. The tool creates the two folders
inside it on first run:

```
purview-writeback/
    inbox/
        ExportActions.xlsx                  <- the newest export, always here
    runs/
        run-2026-10-02T14-02-11Z/
            ExportActions.updated.xlsx      <- upload this one to Purview
            run_report.json
        run-2026-10-05T09-44-03Z/
            ...
```

| Folder | What goes in it |
|---|---|
| `inbox/` | **The newest export from Purview, and nothing else.** Each time you export, save it here as `ExportActions.xlsx`, replacing the previous one. You only ever keep one file here |
| `runs/` | The tool's output. It creates a new dated folder every run, so nothing is overwritten and you can always see what was uploaded and when. Leave these alone |

> [!CAUTION]
> **The one rule.** Files move **in one direction only**: from Purview into
> `inbox/`, and from `runs/` back into Purview. Never copy a file from `runs/`
> into `inbox/` — that is a snapshot of Purview from an earlier day, and
> re-using it would undo anything changed since. The tool stamps its own output
> and refuses to accept it as input.

> [!TIP]
> **Make it permanent.** To avoid retyping the token and this path in every new
> terminal window, add both `export` lines to the end of your shell profile
> (`~/.zshrc` on a Mac), then open a new window.

---

## 5. Running it

Repeat these steps whenever you want to refresh Purview.

### 1. Export the workbook and save it to `inbox/`

Go to Compliance Manager, then **the Assessment of your choice**, and export. It
must be the Assessments page — the Improvement actions page produces a shorter
file that is missing the columns this tool writes. If you use the wrong one, the
tool stops and tells you.

Save the downloaded file into the `inbox/` folder from section 4, named
`ExportActions.xlsx`, replacing whatever is already there.

> [!IMPORTANT]
> Always export a **new** workbook. Never re-use the file a previous run
> produced. That file is a snapshot of Purview from that moment, so anything
> changed since is missing from it, and uploading it would undo those changes.

### 2. Preview without changing anything

This reads Paramify and shows what it would do. It writes no file.

```bash
purview-action-update --dry-run --print-changes-brief
```

### 3. Produce the updated workbook

The same command without `--dry-run`. Your original export is never modified; a
copy is written to a new dated folder.

```bash
purview-action-update --print-changes-brief
```

> [!TIP]
> **What you should see:** the last line reads `verification PASS — the workbook
> satisfies Purview's rules`, and the line above it tells you where the file was
> written. If it says **FAIL, do not upload it** — open an issue with the run
> report attached.

### 4. Look at what changed

Every cell the tool chose not to write is listed with the reason why. This is
the part worth reading.

```bash
open "$PURVIEW_WRITEBACK_DIR"/runs/run-*/run_report.json
```

### 5. Upload it back into Purview

Compliance Manager, then **Assessments**, then **Update actions**. Choose the
`ExportActions.updated.xlsx` file from the newest folder in `runs/` — step 3
printed its exact path.

### 6. Optional — file the result in Paramify as evidence

Attaches the workbook and its run report to an Evidence Set, created once and
reused on later runs. Needs `Write Evidences` on the token.

```bash
purview-action-update --upload
```

To confirm afterwards that an upload landed:

```bash
purview-action-update --list-artifacts
```

---

## 6. Options you may want to change

The defaults are the cautious ones: fill in blanks, change nothing that already
has a value, and refuse anything unclear. Decide these once for your program,
then keep using the same flags.

### How much to overwrite

| Flag | What it does |
|---|---|
| `--mode fill-empty` **(default)** | Writes only into empty cells. Nothing already there is touched |
| `--mode sync` | Also replaces values that disagree with Paramify |

### What happens to notes already in the workbook

| Flag | What it does |
|---|---|
| `--notes-policy follow-mode` **(default)** | Notes follow `--mode` |
| `--notes-policy append` | Keeps the existing note and adds the Paramify narrative underneath. Safe to re-run — it will not stack duplicates |
| `--notes-policy replace` | The Paramify narrative replaces the note entirely |

### When a new date collides with a recorded test

Purview requires `Test Date` to be on or after `Implementation Date`.

| Flag | What it does |
|---|---|
| `--on-date-conflict skip` **(default)** | Leaves the row alone and reports it |
| `--on-date-conflict advance-test` | Moves Test Date to match. The recorded result stands, but its date no longer reflects when the test ran |
| `--on-date-conflict clear-test` | Clears Test Date and Test Status — truthful, since the control needs retesting, but it discards a recorded pass |

### Dates and timezones

Only the calendar date has to be right, so dates render in **the operator's own
timezone, not UTC**. An audit event at `2026-10-02T02:00Z` is still 1 October
everywhere in the US; rendered in UTC it would be written as the 2nd.

Resolution order: `--timezone` → `TZ` → the system zone from `/etc/localtime` →
the OS's current UTC offset → UTC. The run report records which was used.

<details>
<summary><b>Full flag reference</b> — paths, diagnostics, upload and cleanup</summary>

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
| `--status-fallback-solcap` | Where no audit activity exists, use the capability's current status (no date derivable) |
| `--strict-test-status` | Refuse rows with a blank Test Status where the status does not permit `None` |
| `--upload` | Attach the workbook **and** run report to an Evidence Set |
| `--evidence-reference-id` / `--evidence-name` / `--evidence-id` | Target a different Evidence Set |
| `--list-artifacts [EVIDENCE]` | Show what is attached, to confirm an upload landed |
| `--allow-no-changes` | Upload even when nothing changed |
| `--min-match-rate` | Refuse to upload below this name-match fraction. Off by default |
| `--accept-near-matches` | Join names matching only after case/whitespace normalization |
| `--allow-chained-input` | Accept a workbook this tool produced (refused by default) |
| `--prune-no-op-artifacts` / `--delete-evidence-set` | Cleanup; both need `--confirm` |

</details>

---

## 7. If something goes wrong

### Reading the summary

| Line | What it tells you |
|---|---|
| `exact name matches` | How many rows found their capability |
| `cells written` | What actually changed, by column |
| `cells held back` | Proposed but declined — reasons in the report |
| `audit activity found` | Which capabilities had a status change the tool could read |
| `evidence set` | Whether anything reached Paramify |
| `verification` | `PASS` means the file satisfies Purview's rules. `FAIL` means do not upload it |

Lines beginning `!` are advisories: things that *were* written but deserve a
look. They are not errors.

### Messages

| Message | What it means |
|---|---|
| Rejected the token under both auth schemes | Wrong or expired token, or the right token for a different Paramify instance |
| Missing required column | The workbook came from the Improvement actions page. Export again from Assessments |
| Refusing to run: produced by this tool | You passed a previous run's output. Export a fresh one from Purview |
| NOTHING matched by name | No action name matched any capability name. The token is almost certainly pointing at a different workspace |
| Refusing to upload: the run changed no cells | Nothing needed changing. Everything already matches |
| Refusing to upload: only *n* rows matched | Only appears if you set `--min-match-rate` yourself. A partial match is normal and does not stop anything by default |
| `verification FAIL` | Do not upload the file |

> [!NOTE]
> `run_report.json`, saved next to the workbook, records every decision and
> every reason. It contains no passwords or tokens, and it is the single most
> useful thing to attach to an issue.

---

## 8. What this tool will never do

These are built into the code, not left to care. Every refusal is recorded with
its reason.

- **Connect to your Microsoft tenant.** It holds no Microsoft credentials.
- **Change your original exported file.** It always writes a copy.
- **Write to any column other than the three** in section 1 — five, if you opt
  into a date-conflict resolution.
- **Empty a cell** because Paramify had nothing to say.
- **Write a status for `PARTIALLY_IMPLEMENTED` or `NOT_SET`** — neither has an
  honest Purview equivalent.
- **Write a status that invalidates the row's Test Status**, or a date later
  than it, unless you ask for a resolution.
- **Produce a workbook Purview would reject** — it re-reads the finished file
  and checks it against Purview's own rules, and `--upload` refuses a failing one.
- **Accept its own output as input**, so a stale snapshot cannot be re-uploaded
  by accident.
- **Guess.** An unclear status, an unreadable date or a name that only nearly
  matches is reported, never resolved quietly.

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
recreates every record, leaving the status history in the source.
`--probe-audit` diagnoses that case by name.

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
repo, and none should ever be added. The end-to-end tests need a real Purview
export and skip without one; point `PURVIEW_EXPORT` at a workbook to run them.

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

The design principle throughout: **decide, then apply.** The planner produces
proposed changes with a written/declined flag and a reason; only then does
anything touch the workbook. That is why every refusal can explain itself.

## Contributing

Issues and pull requests welcome. Two things make a change easy to accept:

- **A test that fails before the change and passes after.** The suite is the
  specification, particularly for the refusals — most of them exist because a
  real run got something wrong.
- **No tenant data.** Fixtures are synthetic. If a bug needs real data to
  reproduce, describe the shape rather than attaching the file.

## Licence

GPL-3.0-only. See [LICENSE](LICENSE).
