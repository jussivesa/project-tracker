# project-tracker

Automatic daily/weekly work journal. Collects activity from git history, GitHub PRs/CI/releases, Claude Code
conversations, code TODOs, and meeting notes — then renders a Markdown report per day with a Friday weekly roll-up.

**Script**: `~/Projects/project-tracker/project-tracker`
**Config**: `~/.config/project-tracker/config.toml`
**Reports**: `~/Documents/ProjectTracking/`
**Log**: `~/.config/project-tracker/tracker.log`

---

## Usage

```fish
project-tracker                          # today, all projects (stdout only)
project-tracker day                      # same as above
project-tracker day 2026-04-22           # specific past day
project-tracker day --project abc        # one project only (substring match)
project-tracker day --save               # stdout + write to ~/Documents/ProjectTracking/
project-tracker day --no-claude          # skip Claude AI summaries (faster)
project-tracker day --no-gh              # skip all GitHub API calls (fastest)

project-tracker week                     # Mon–Fri of the current week
project-tracker week --last              # previous week
project-tracker week --save              # write to ~/Documents/ProjectTracking/
project-tracker week --no-claude         # skip Claude AI summaries
project-tracker week --no-gh             # skip GitHub

project-tracker catchup                  # backfill any missed daily/weekly reports
project-tracker setup                    # (re-)install LaunchAgents
```

All flags are combinable: `project-tracker week --last --save --no-gh`.

### Saving to file

Running without `--save` prints to stdout only — useful for a quick look without overwriting an existing report. The
scheduled LaunchAgents always pass `--save` automatically, so `~/Documents/ProjectTracking/` is kept up to date without
manual intervention.

---

## Features

### Per-day report

Each active project produces a section showing:

| Section              | Details                                                                                                              |
|----------------------|----------------------------------------------------------------------------------------------------------------------|
| **Commits**          | Grouped by conventional prefix (`feat/fix/refactor/chore/docs/test`), with total lines added/removed                 |
| **PRs merged**       | Title, number, and URL of GitHub PRs merged that day                                                                 |
| **CI / Deployments** | GitHub Actions workflow runs with pass/fail icon, branch, and local time                                             |
| **Releases**         | GitHub releases published that day                                                                                   |
| **Claude sessions**  | Number of Claude Code sessions started that day and the first task message from each                                 |
| **TODOs**            | `TODO/FIXME/HACK/XXX` comments from code (file:line), plus `TODO`/`- [ ]` lines found in Claude conversations        |
| **Meeting notes**    | `.txt`, `.docx`, `.md` files modified that day in the matched `~/Documents/` customer folder, summarised with Claude |

A project appears in the report if it had at least one commit, PR, CI run, release, Claude session, or document update
that day.

Customer folders in `~/Documents/` that have no associated git repo (e.g. `~/Documents/Client Abc`) also appear
automatically when documents are modified — covering projects that are purely meetings or workshops.

### Weekly report

Collects all five working days (Mon–Fri) and produces:

- **Project totals**: estimated hours, commit count, PR count, CI runs, releases
- **Day-by-day breakdown**: per-project commit groups, PR/release lines, and Claude session notes for each day

### Time estimation

**When commits exist** — timestamps are grouped into work sessions. A gap of more than **2 hours** between commits
starts a new session:

```
estimated minutes = Σ sessions × (base_minutes + per_commit_minutes × commits_in_session)
```

**When there are no commits** — Claude sessions and meeting documents are summed independently:

```
estimated minutes = (Claude sessions × session_claude_minutes) + (documents × session_doc_minutes)
```

All four constants are configurable in `config.toml` under `[estimation]`.

### Document summaries

When `--no-claude` is not set, each meeting note or document is passed to `claude -p` with a structured prompt. The
response is rendered as indented Markdown under the filename with four sections:

| Section                  | Contents                                                                                          |
|--------------------------|---------------------------------------------------------------------------------------------------|
| **Summary**              | One-sentence topic; **Attendees** (if names found); **Keywords** (3–5 key topics or proper nouns) |
| **Key Points**           | 3–5 bullet points of the most important information                                               |
| **Action Points**        | Concrete tasks or decisions requiring follow-up; omitted if none                                  |
| **Suggested Next Steps** | 0–3 specific actionable suggestions for what to do next; omitted if nothing actionable            |

If Claude is unavailable or times out (60 s per document), the raw file path is shown instead.

---

## Configuration

All configuration lives in `~/.config/project-tracker/config.toml`. Changes take effect on the next run. Run
`project-tracker setup` after changing `[schedule]` values to reinstall the LaunchAgents with the new times.

### `[paths]`

| Key             | Default                       | Meaning                                                                  |
|-----------------|-------------------------------|--------------------------------------------------------------------------|
| `projects_dir`  | `~/Projects`                  | Root directory scanned for git repos                                     |
| `documents_dir` | `~/Documents`                 | Root directory scanned for meeting notes and standalone customer folders |
| `claude_dir`    | `~/.claude/projects`          | Claude Code conversation storage                                         |
| `output_dir`    | `~/Documents/ProjectTracking` | Where `--save` writes Markdown files                                     |

### `[estimation]`

| Key                          | Default | Meaning                                                                   |
|------------------------------|---------|---------------------------------------------------------------------------|
| `session_gap_hours`          | `2`     | Hours gap between commits that starts a new work session                  |
| `session_base_minutes`       | `30`    | Base minutes added per session regardless of commit count                 |
| `session_per_commit_minutes` | `15`    | Extra minutes per commit within a session                                 |
| `session_claude_minutes`     | `30`    | Minutes credited per Claude Code session when there are no commits        |
| `session_doc_minutes`        | `60`    | Minutes estimated per meeting/workshop document when there are no commits |

### `[scanning]`

| Key                | Default             | Meaning                                                                                                                                                      |
|--------------------|---------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `code_extensions`  | `["*.cs","*.ts",…]` | File extensions scanned for TODO/FIXME/HACK/XXX                                                                                                              |
| `docs_noise_words` | `["api","app",…]`   | Words stripped before fuzzy-matching repo names to `~/Documents/` folders — extend this list if a common word in your repo names causes wrong folder matches |

### `[schedule]`

| Key                             | Default | Meaning                                                 |
|---------------------------------|---------|---------------------------------------------------------|
| `daily_hour` / `daily_minute`   | `18:00` | Time to auto-save the daily report                      |
| `weekly_hour` / `weekly_minute` | `17:30` | Time to auto-save the weekly summary (always on Friday) |

The catch-up agent always runs at **08:00** and is not configurable in the file — change `cmd_setup` in the script if
needed.

### Project → Documents folder matching

For each git repo, the fuzzy matcher looks for a corresponding `~/Documents/` subfolder. It tokenises both names on `-_`
and spaces, drops common noise words (configured via `docs_noise_words`), and scores by word overlap. The
highest-scoring directory wins; a score of 0 means no match.

Example: `abc-oy-key-file-system` → tokens `{abc, key, file}` → matches `~/Documents/ABC Oy`.

After all git projects are matched, any `~/Documents/` subfolder not yet claimed is scanned independently and appears in
the report if it had document activity that day. This covers customer folders with no git repo.

### Code TODO scanning

Keywords matched: `TODO`, `FIXME`, `HACK`, `XXX` (case-insensitive). Paths containing `.git/`, `node_modules/`, `obj/`,
or `bin/` are excluded. Up to **30** items are kept per project.

### Claude conversation TODO scanning

Lines matching `\bTODO\b`, `\bFIXME\b`, or `- [ ]` inside user messages in Claude session JSONL files are extracted. Up
to **5** per session, **4** shown in the report.

---

## Data sources

### 1 · Git (`git log`)

Run per project with `--numstat --no-merges`. Merge commits (`Merge pull request …` / `Merge branch …`) are filtered
out. LOC stats are collected from `--numstat` output in the same pass.

### 2 · GitHub PRs (`gh pr list`)

Requires `gh` CLI authenticated. Filtered by `merged:YYYY-MM-DD..YYYY-MM-DD+1` search. Only repos with a `github.com`
remote are queried.

### 3 · GitHub Actions (`gh run list`)

Filtered with `--created YYYY-MM-DD`. Workflow runs are deduplicated by workflow name (latest wins) to avoid showing
multiple entries for the same workflow. Runs in `in_progress` or `queued` state still appear.

### 4 · GitHub Releases (`gh release list`)

Fetches the latest 20 non-draft releases and filters to those published on the target day.

### 5 · Claude Code sessions

Session JSONL files live at:

```
~/.claude/projects/-Users-vesa-Projects-{repo-name}/{session-uuid}.jsonl
```

The path mapping is: `/Users/vesa/Projects/foo` → replace every `/` with `-` → `-Users-vesa-Projects-foo`.

A session is attributed to a day based on the **file creation timestamp** (`st_birthtime` on macOS). The first
`"timestamp"` field found in any JSONL line is used as the session start time; birthtime is the fallback. User messages
starting with `<` (system tags) or `/` (slash commands) are skipped.

### 6 · Documents / meeting notes

Files with `.txt`, `.docx`, or `.md` extensions whose `mtime` falls within the target day. `.docx` files are converted
to plain text with `textutil -convert txt`.

Scanning runs in two passes:

1. **Git-project pass** — for every git repo in `~/Projects/`, the fuzzy matcher looks for a corresponding
   `~/Documents/` subfolder and collects matching files.
2. **Standalone pass** — every `~/Documents/` subfolder not claimed by a git project is scanned independently. Any
   folder with matching file activity becomes its own report entry.

---

## Scheduling (LaunchAgents)

Running `project-tracker setup` installs three macOS LaunchAgents.

### Files

| File                                                            | Purpose                                  |
|-----------------------------------------------------------------|------------------------------------------|
| `~/Library/LaunchAgents/com.vesa.project-tracker.daily.plist`   | Saves daily report at 18:00              |
| `~/Library/LaunchAgents/com.vesa.project-tracker.weekly.plist`  | Saves weekly summary on Fridays at 17:30 |
| `~/Library/LaunchAgents/com.vesa.project-tracker.catchup.plist` | Backfills any missed reports             |

### Schedule

| Agent    | When                                        | What it does                                                                                                                                |
|----------|---------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| Daily    | Weekdays at **18:00**                       | `project-tracker day --save`                                                                                                                |
| Weekly   | Fridays at **17:30**                        | `project-tracker week --save`                                                                                                               |
| Catch-up | Every day at **08:00** + **on every login** | Checks the last 14 days for missing daily reports and the last 4 completed weeks for missing weekly reports — generates any that are absent |

The catch-up agent runs at 08:00 and also has `RunAtLoad: true`, meaning it fires automatically on login after a
restart. Together these two triggers handle the Mac being asleep when the daily or weekly agent was scheduled to run:

- **Slept through 18:00 or 17:30**: the catch-up runs at 08:00 the next morning and backfills the missing file.
- **Mac was shut down**: the catch-up fires at login and backfills everything missing from the last 14 days and 4 weeks.
- Catch-up is fully idempotent — it checks whether each output file already exists before generating it, so running it
  multiple times is safe.

### Verifying the agents are loaded

```fish
launchctl list | grep project-tracker
```

A `-` in the PID column means not currently running (normal). A `0` exit code means the last run succeeded.

### Log

All agents write stdout and stderr to:

```
~/.config/project-tracker/tracker.log
```

Tail it to debug a failed run:

```fish
tail -f ~/.config/project-tracker/tracker.log
```

### Re-installing after moving the script

Run `project-tracker setup` again. It unloads the old plists before writing and loading the new ones. The plist
`ProgramArguments` records the **absolute path** of the script at setup time.

---

## Output files

Reports are written to `~/Documents/ProjectTracking/` when `--save` is passed or when a scheduled agent runs:

| Filename        | Contents                                       |
|-----------------|------------------------------------------------|
| `2026-04-24.md` | Daily report for that date                     |
| `2026-W16.md`   | Weekly summary (week number, starts on Monday) |

---

## Dependencies

| Tool | Required fo-----------------tes |
|------------------ -|------------ -|-------|
| Python 3.11+ | Running the script | Uses `tomllib` (stdlib since 3 .11); tested on 3.14 |
| `git`          | Commit/LOC da ta | Must be on PATH |
| `gh` (GitHub CLI) | PRs, CI runs, releases | Optional; skipped per-repo if no `github. com` remote, o r globally with
`--no-gh` |
| `claude` CLI | Meeting note summaries | Optional; skipped automatic ally if not found or `--no-claude` is passed |
| `textutil`        | Reading `.docx` files | Built into macOS; no install needed |

No third-party Python packages are required.
