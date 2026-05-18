---
description: Draft Josh's SSW Daily Scrum email from yesterday's GitHub activity and copy the email + Yesterday section to the clipboard.
---

You are running the `/good-morning` workflow. Follow these steps in order. Be concise in user-facing text — no narration, no preamble, just the questions and the drafts.

## Step 1 — Resolve the date window

Read `~/.claude/config/daily-scrum.json` and extract `githubUsername`, `trelloBoard`, `constants`. Also destructure `constants.daysUntilClient` as `$DAYS_UNTIL_CLIENT` and `constants.joinedScrum` as `$JOINED_SCRUM` — these are referenced by name in the Step 7 email template.

Parse the user's invocation message for date modifiers:
- "yesterday was Tuesday" / "treat Monday as yesterday" → use the named weekday from this week as the window
- "pull the last N days" → window = N days ending yesterday
- "skip today section" → set a flag to omit Step 6
- No modifier → default

Default: "yesterday" = last working day. Run `date +%u` to get today's weekday (1=Mon..7=Sun). If today is Monday (1), yesterday = `date -v-3d +%Y-%m-%d`. Otherwise yesterday = `date -v-1d +%Y-%m-%d`.

Store the resolved window as `$START` and `$END` (same date for single-day, range for multi-day).

## Step 2 — Prompt for inbox count

Ask the user exactly: `inbox count?`

Accept a number, or `skip` to leave it blank. Store as `$INBOX`.

## Step 3 — Fetch GitHub activity

Run these `gh` queries (substitute `$USER` with `githubUsername`, `$START`/`$END` with the resolved window). Always use the range form `$START..$END` — when `$START == $END` it correctly degenerates to a single day.

**PRs authored (any state in window):**
```
gh api -X GET search/issues -f q="is:pr author:$USER updated:$START..$END" \
  --jq '.items[] | {number, title, url: .html_url, state, draft, repo: (.repository_url | sub("https://api.github.com/repos/"; "")), merged_at: .pull_request.merged_at, updated_at, closed_at}'
```

`pull_request.merged_at` is the reliable way to know if/when a PR merged. `state` alone returns `closed` for both merged and closed-without-merge.

**PRs merged in window:**
```
gh search prs --author="$USER" --merged-at="$START..$END" \
  --json number,title,url,repository,closedAt,state
```

Note: the flag is `--merged-at` (takes a date), not `--merged` (which is a boolean). For merged PRs, `closedAt` equals the merge time and `state` is `"merged"`. `mergedAt` is NOT a valid `--json` field — use `closedAt` + `state == "merged"`.

**PRs reviewed by user in window:**
```
gh search prs --reviewed-by="$USER" --updated="$START..$END" \
  --json number,title,url,state,repository,updatedAt
```

**Issues touched (assigned + updated):**
```
gh search issues --assignee="$USER" --updated="$START..$END" \
  --json number,title,url,state,repository,updatedAt,closedAt
```

**Issues commented on:**
```
gh search issues --commenter="$USER" --updated="$START..$END" \
  --json number,title,url,state,repository,updatedAt
```

**Issues closed by user:**
```
gh search issues --assignee="$USER" --closed="$START..$END" \
  --json number,title,url,repository,closedAt
```

If any query fails (auth, rate limit), STOP and tell the user: `❌ gh query failed: <error>. Fix and re-run.` Don't fall back.

Merge the results into a single deduped list keyed by `repo + #number`. As you merge, **tag each item with its source queries** (e.g. `["authored"]`, `["reviewed-by"]`, `["assignee", "commenter"]`). Dedup is load-bearing because:

- `--reviewed-by` includes the user's own PRs (which also appear in PRs authored) — keep the authored entry, drop the reviewed dup
- `--commenter` is very noisy — often returns issues that have a corresponding PR in another list; keep the PR entry, drop the issue dup
- Linked PR↔issue pairs appear in both PR and issue lists — keep the PR entry

Preserve the original title verbatim including any emojis. Treat `closedAt = "0001-01-01T00:00:00Z"` as null (open item).

### Step 3.1 — Filter "review-only" PRs by size

Identify "review-only" items: PRs whose source tags are exactly `["reviewed-by"]` (the user did not author, was not assigned, did not comment). These are PRs Josh only reviewed.

For each review-only PR, fetch diff stats:

```
gh api repos/{repo}/pulls/{number} --jq '{additions, deletions, changed_files, author: .user.login}'
```

Dispatch these fetches in parallel (one Bash call per fetch, all in one message).

**Filter rule:** keep a review-only PR ONLY if `(additions + deletions) > 100` OR `changed_files > 5`. Drop the rest entirely (don't classify, don't render). Mark survivors with `reviewOnly: true` so rendering knows to use the "Reviewed" prefix.

Also drop any review-only PR whose `author` equals `$USER` — defensive against gh search returning self-reviews even when the dedup step should have caught it.

### Step 3.2 — Fetch body + comments for surviving items

For each surviving item (anything not dropped by Step 3.1), fetch body + last 3 comments. Dispatch ALL fetches concurrently in a single message using parallel Bash invocations.

```
gh api repos/{repo}/issues/{number} --jq '{body, html_url}'
gh api repos/{repo}/issues/{number}/comments?per_page=100 --jq '[.[-3:][] | {user: .user.login, body, created_at}]'
```

## Step 3.5 — Classify each item (parallel Haiku subagents)

For each item, dispatch a fresh subagent in parallel using the Agent tool with `subagent_type="general-purpose"` and `model="haiku"`. Each subagent receives this prompt (substitute the placeholders):

```
You are classifying one GitHub item for Josh's Daily Scrum email. Output JSON only — no prose.

Window: {start} to {end} (inclusive).
User: @{githubUsername}

Item metadata:
{item_json}

Body:
{item_body}

Last 3 comments:
{recent_comments}

Decide:
1. bucket — one of:
   - "old": work that is now DONE/CLOSED (merged PR, closed issue). Goes in Yesterday only. Use this whenever merged_at or closed_at is non-null, regardless of whether it was opened the same day.
   - "new": a PR or issue that is still OPEN and was first created in the window. Goes in Today only.
   - "carryover": a PR or issue that is still OPEN and predates the window OR was opened in the window but not yet resolved. Goes in both Yesterday and Today.

   Key rule: if merged_at is non-null → bucket = "old". If state is open → "new" if created_at falls within {start}..{end}; "carryover" if created_at is before {start} (item predates the window but is still in flight).

2. status — pick the {emoji, text} pair that matches. "Done" covers any work Josh completed (his own merged PRs AND closed issues). The "Reviewed" label is applied later in rendering ONLY for items Josh did not author:
   - merged PR (item is a PR AND merged_at non-null) → emoji ✅, text "Done"
   - closed-without-merge PR (item is a PR, closed_at non-null, merged_at null) → emoji ❌, text "Closed"
   - open PR (authored or reviewed by user) → emoji 👀, text "In review"
   - closed issue (item is an issue, closed_at non-null) → emoji ✅, text "Done"
   - open issue + assigned + activity in window → emoji 🚧, text "In progress"
   - open issue + assigned, no activity in window → emoji ⚒️, text "TODO"

3. nextStep — IF the body or comments mention an explicit follow-up the user should do next (e.g. "needs review from X", "waiting on Y", "follow-up PBI #123", "blocked on Z"), include it as one short sentence. Otherwise null. Do NOT invent next steps.

Output exactly:
{"bucket": "...", "status": {"emoji": "...", "text": "..."}, "nextStep": "..." | null}
```

If a classifier returns malformed JSON or fails, fall back: rule-map the status from the table above; bucket = `old` if state is closed/merged, else `carryover`; `nextStep` = null.

Attach each classifier's output to its item as `item.classification`.

## Step 4 — Group by repo

For each item, derive its section heading from `repo`: take the part after the slash (`SSWConsulting/SSW.YakShaver` → `SSW.YakShaver`). Build groups in the order repos first appear in the activity.

## Step 5 — Render the Yesterday section

Include every item whose `classification.bucket` is `old` or `carryover`. Render each as one line:

```
{classification.status.emoji} {classification.status.text} - <title-with-emojis> #<number>
```

**Review-only override:** if `item.reviewOnly === true` (from Step 3.1), replace `{classification.status.text}` with the literal `Reviewed` regardless of what the classifier returned. The emoji still reflects state (✅ for merged, 👀 for open). Example: `👀 Reviewed - <title> #N` for an open PR Josh only reviewed, `✅ Reviewed - <title> #N` for a merged one.

If `classification.nextStep` is non-null, emit it on an indented next line beneath the item.

Group layout:

```
      <RepoHeading>:
<item lines>
```

(Six leading spaces before the heading. Items flush-left under it.)

## Step 6 — Render the Today section

Skip entirely if the user said "skip today section."

Include every item whose `classification.bucket` is `new` or `carryover`. Use the same line format as Step 5.

If the Today section is empty after classification, fall back in order:

1. **Recent assignments** — issues assigned to `$USER` in the last 3 days not already shown. Query:
   ```
   THREE_DAYS_AGO=$(date -v-3d +%Y-%m-%d)
   gh search issues --assignee="$USER" --updated=">=$THREE_DAYS_AGO" --state=open --json number,title,url,repository,updatedAt
   ```
2. **Backlog suggestions** — only if (1) is also empty. Pick the 3 most-recently-updated open issues from the repos that had activity in the window. Prefix the section with `Suggested (no carryover available):`.

## Step 7 — Assemble the full email

```
Hi BenchMasters,

I have {daysUntilClient} day until my next client booking
I have {INBOX} emails in my inbox
My Trello board is at {trelloBoard}
I have joined my daily scrum meeting {joinedScrum}


Yesterday

{yesterday_groups}


Today

{today_groups}

```

If `$INBOX` is blank (user said skip), omit the "I have N emails in my inbox" line. If today section was skipped, drop the entire "Today" block and the preceding blank line. No signature — Outlook appends its own.

### Step 7.1 — Also build the HTML version for rich clipboard paste

Build a parallel HTML representation of the same email body. This is what gets copied to the clipboard so Outlook receives rich text with clickable links.

HTML structure (keep it minimal — Outlook is picky):

```html
<html><body>
<div>Hi BenchMasters,</div>
<div>&nbsp;</div>
<div>I have N/A day until my next client booking</div>
<div>I have 0 emails in my inbox</div>
<div>My Trello board is at <a href="https://trello.com/b/9TzpTUjI/josh-berman-ssw-backlog">https://trello.com/b/9TzpTUjI/josh-berman-ssw-backlog</a></div>
<div>I have joined my daily scrum meeting ❌</div>
<div>&nbsp;</div>
<div>&nbsp;</div>
<div>Yesterday</div>
<div>&nbsp;</div>
<div>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;tina.io:</div>
<div>✅ Done - 🐛 Fix Release Notes auto-merge permission error <a href="https://github.com/tinacms/tina.io/pull/4541">#4541</a></div>
... etc ...
</body></html>
```

Rules:
- One `<div>` per line of the plain text version
- Blank lines become `<div>&nbsp;</div>`
- The 6-space repo heading indent uses `&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`
- Every `#N` becomes `<a href="{item.url}">#N</a>` using the item's `html_url` from Step 3
- Every bare URL (Trello link, etc.) becomes `<a href="X">X</a>`
- Emojis stay as literal UTF-8 characters
- Preserve any nextStep indented context lines with `&nbsp;&nbsp;` prefix

Store as `$EMAIL_HTML`. Also build `$YESTERDAY_HTML` containing just the header block + Yesterday section (for the second clipboard copy in Step 9).

## Step 8 — Show drafts for review

Print the assembled email body in a fenced code block. Then ask: `Edits? (or "ship it")`

Loop on conversational edits:
- "tighten item 3" / "drop the YakShaver section" / "add a TODO line about X" → regenerate the relevant section, reprint full email, re-ask.
- Anything matching "ship it" / "looks good" / "send" / "yes" → proceed to Step 9.

Also nudge once after the first draft: `Anything you worked on yesterday that's not in GitHub yet? (e.g. TimePro sprint summary, CTF review)`. Apply any additions to the Yesterday section.

If Step 3 produced zero items, render Yesterday as: `Nothing logged in GitHub yesterday — was I off / in meetings? (tell me what to add)` and skip the nudge.

## Step 9 — Two-stage clipboard handoff (rich HTML)

On approval, copy the HTML version to the clipboard so Outlook pastes as rich text with clickable links. Use `osascript` with raw hex data — this bypasses all string-escaping issues that arise from quoting HTML in shell.

### Step 9a — Copy the full email as HTML

```bash
# Write the HTML to a temp file (avoids shell escaping of the HTML body)
cat > /tmp/good-morning-email.html << 'HTMLEOF'
{paste $EMAIL_HTML here verbatim}
HTMLEOF

# Convert to hex and set the clipboard with HTML class type
hex=$(xxd -p /tmp/good-morning-email.html | tr -d '\n')
osascript -e "set the clipboard to «data HTML${hex}»"
```

(The `«data HTML...»` syntax tells macOS the clipboard contains raw bytes typed as HTML, which Outlook recognizes for rich-text paste.)

After running: say `✅ email body copied as rich text. Paste into Outlook (links will be clickable), then hit enter.`

### Step 9b — Wait, then copy the Yesterday block

1. Wait for user input.
2. Build the Yesterday-only HTML in `$YESTERDAY_HTML` (header block + Yesterday section, no Today, no signature).
3. Repeat the same `cat > /tmp/...html` → `xxd -p` → `osascript` pattern with `$YESTERDAY_HTML` written to `/tmp/good-morning-yesterday.html`.
4. Say: `✅ Yesterday section copied as rich text. Paste into TimePro.`

### Fallback

If `osascript` fails (rare — only on locked-down systems), fall back to plain text: `printf '%s' "$EMAIL" | pbcopy` and tell the user the links will be plain URLs they may need to click separately.

Done.
