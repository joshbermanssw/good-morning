# /good-morning

A Claude Code skill that drafts an SSW Daily Scrum email from yesterday's GitHub activity, classifies each item, and copies the result to the clipboard as rich HTML — ready to paste into Outlook with clickable PR/issue links.

Also produces a Yesterday-only block for pasting into TimePro as the day's timesheet.

## What it does

1. Resolves "yesterday" (last working day, or whatever you tell it: `/good-morning yesterday was Tuesday`)
2. Pulls every PR + issue you touched in that window via `gh`
3. Filters out trivial PR reviews (keeps only those above a size threshold)
4. Dispatches a Haiku subagent per item to classify it as `old` / `new` / `carryover` and pick a status emoji
5. Drafts the email — grouped by repo, formatted in the SSW Daily Scrum template, signature-free (Outlook adds its own)
6. Lets you iterate conversationally ("tighten item 3", "drop the YakShaver section", "add a TODO line about X")
7. On approval, copies the full email to clipboard as rich HTML, then on enter copies the Yesterday block

## Install

```bash
git clone https://github.com/joshbermanssw/good-morning ~/.claude/skills/good-morning
cp ~/.claude/skills/good-morning/daily-scrum.example.json ~/.claude/config/daily-scrum.json
$EDITOR ~/.claude/config/daily-scrum.json   # set githubUsername + trelloBoard
```

Restart Claude Code (or open a fresh session) — `/good-morning` will appear in completions.

## Requirements

- macOS (uses `pbcopy` and `osascript` for rich-text clipboard)
- [`gh`](https://cli.github.com/) CLI, authenticated (`gh auth status`)
- Claude Code

## Config

`~/.claude/config/daily-scrum.json`:

| Field | What it is |
|---|---|
| `githubUsername` | Your GitHub handle — used in every `gh search` query |
| `trelloBoard` | Link to your personal backlog board — appears in the email header |
| `constants.daysUntilClient` | Hardcoded value for the "I have N day until my next client booking" line (default `"N/A"`) |
| `constants.joinedScrum` | Default rendered for "I have joined my daily scrum meeting" (default `"❌"`; toggle in Outlook after paste if needed) |

## Usage

```
/good-morning                          # default: last working day
/good-morning yesterday was Tuesday    # specific weekday override
/good-morning pull the last 2 days     # wider window
/good-morning skip today section       # email without the Today block
```

You'll be prompted once for inbox count (type a number, or `skip`).

## How items are classified

Each surviving GitHub item is sent to a parallel Haiku classifier that returns:

- **bucket** — `old` (work that's done; Yesterday only), `new` (fresh arrival; Today only), or `carryover` (in flight; both sections)
- **status** — one of `✅ Done`, `❌ Closed`, `👀 In review`, `🚧 In progress`, `⚒️ TODO` (PRs you only reviewed render as `Reviewed` instead)
- **nextStep** — optional one-line follow-up if the PR/issue mentions one explicitly

PR reviews where you weren't the author are filtered to "substantive" only — kept if `additions + deletions > 100` OR `changed_files > 5`. Tune the threshold in `SKILL.md` Step 3.1.

## Output format

Mirrors the SSW Daily Scrum template:

```
Hi BenchMasters,

I have N/A day until my next client booking
I have 7 emails in my inbox
My Trello board is at https://trello.com/b/...
I have joined my daily scrum meeting ❌


Yesterday

      tina.io:
✅ Done - 🐛 Fix Release Notes auto-merge permission error #4541
✅ Reviewed - Massive auth refactor #4480

      tinacms:
👀 In review - Replace cloudinary assets with in-repo assets #6839


Today

      tinacms:
👀 In review - Replace cloudinary assets with in-repo assets #6839
🚧 In progress - 🧑‍🚀 Document Astro + Tina visual editing setup #6793
```

Pasted into Outlook, every `#N` is a live link to the corresponding GitHub PR/issue.

## License

MIT.
