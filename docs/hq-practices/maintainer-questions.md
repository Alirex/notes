---
summary: "Asking the maintainer through issues, defaults, blocking, manual-action issues"
group: agents
tags: [agents, issues]
applies-to: [all]
reviewed: 2026-10-29
status: stable
---

# Questions for the maintainer

Agents, above all unattended ones ([autonomous-sessions.md](autonomous-sessions.md)), hit judgment calls that only
the maintainer can make. They ask as a **GitHub issue with the label `maintainer-question`** in the repo the
question is about, take a safe default, and keep working.

## Why issues, not a file in git

| Need | Shared `QUESTIONS.md` | One file per question on a `questions` branch | **Issues** |
| --- | --- | --- | --- |
| Visible the moment it is asked | no, only after the branch lands | yes | yes |
| No conflicts between parallel sessions | no, same lines | yes | yes |
| Answer from the phone | clumsy (edit a file) | clumsy (switch branch, edit) | a comment in the GitHub app |
| One list across all repos | clone and read each | fetch each | one `gh search issues` |
| Code to write and keep | none | plumbing script per repo | a 50-line wrapper |
| Touches `main` | at landing | never | never |

What issues cost: they need the network and `gh` logged in, and the record lives on GitHub, not in git. That is
fine for a question; anything lasting from the answer goes into git anyway (a decision file, a changelog entry, a
backlog item). No free push notifications: agents post as the maintainer's own account, and GitHub does not notify
you about your own actions; check the list instead (below), until a notifier exists (HQ web app design).

## Asking

`scripts/docs/questions.sh ask [--blocking] "<question>" <body file|->` (in HQ: `just docs-questions-ask`;
the script copies to any repo unchanged). Title: the question itself, one line. Body (`questions.sh template` prints
it):

````markdown
**Context.** 2-5 sentences, links to files, design docs or the branch.

**Options**

1. Keep the key file; losing it resets 2FA for everyone. *(default, done meanwhile)*
2. Derive the key from a passphrase prompted at start.

**Run.** Only when an option needs the maintainer to run something ([Commands](#commands)).

In `~/dev/hq`:

```bash
just web-admin-setup-code
```

**Meanwhile.** What was done instead of stopping, and what changes if the answer is not the default.

Backlog: B-261023-webauthn · Branch: feat/totp · Asked by: <session name>
````

- One question per issue. A design with six open points is one issue that links the design's question list.
- **No question** when the repo's rules or a practice already decide it.
- **Not blocking** (the usual case): take the safest option that is easy to undo, do it, keep going. A different
  answer later becomes a follow-up item. After 7 days without an answer the default stands: comment
  `Defaulted to option <n>; comment to change it.`, add the label `defaulted` and close it.
- **Blocking** (`--blocking`, label `blocking`): only when every option is hard to undo (data migrations,
  deletions, public names, money, anything outward-facing). Do not choose; set the backlog item to `design` with a
  link to the issue, and pick other work. Never defaulted: it waits for the answer.
- At most 5 open questions per repo; past that, add sub-questions to one bundled issue instead of a new one each.
- More context later: edit the issue body, do not comment. **Any comment is the maintainer's answer**, which is how
  `answered` finds them.

## Answering

Comment on the issue: an option number (`2`), several (`1, 3`: the HQ web app posts that when more than one option is ticked), a number with changes, or free text. In chat with an agent works too:
the agent posts the answer as a comment (starting `Answer from chat:`), so the record is the same.

Where to look: `just docs-questions` in HQ (every repo), `scripts/docs/questions.sh ls` in a repo, or the GitHub app
with the filter `is:open label:maintainer-question author:@me`.

Only unanswered questions wait for the maintainer. An answered, still open one is agent work: `ls`, the HQ web app's
badge and the widget count it apart ("waiting for agents"), in neutral colours. One still open a day after the answer
is an agent or fleet problem (the asker is gone, or nobody checks `answered`), never a reason to ask the maintainer
again. In HQ, `just cc-watch` flags those (and alerts on its timer), and `just machine-capacity` counts answers to
apply as a repo's pickable work, so a repo with nothing else open still gets a worker.

**Skip (not now)** is the third thing the maintainer can do with an unanswered question, next to answering it and
leaving it: a button in the HQ web app (tomorrow, 3 days, a week, a date, or until restored). It sets the question
aside without answering it:

- The question stays open and gets the label `skipped`, plus `skip-until-<YYYYMMDDTHHMMZ>` (UTC) for a dated skip.
  **No comment is posted**, since any comment is an answer; the labels are the whole record. Adding the labels on
  GitHub does the same; removing them (or **Restore** in the web app) brings it back.
- A skipped question leaves every "waiting for you" count (Inbox badge and heading, home, the widget and top bar, the
  repo overview) and is never flagged stale, until its date passes or it is restored. The web app lists it under a
  "skipped" toggle with its date.
- It is not an answer and not a decision: the asking agent keeps going on its stated default, exactly as for any
  unanswered question, and does not treat the labels as consent to the default. A blocking question stays blocking
  for the work that waits on it. `questions.sh ls` leaves skipped ones out (`--skipped` shows them in a third
  group); `answered`, `wait` and `just cc-watch` only ever look at comments, so skipping changes none of them.

## Acting on the answer

- Agents check `questions.sh answered` (and `actions.sh closed`) at session start, before picking the next item and
  after each land; an answered question comes before new work. Any session may act on it: the asker may be gone.
- The work goes through the repo's normal branch and landing. Mention `#<n>` in the commit message, so GitHub links
  the commit to the question. A lasting choice also gets a decision file or a changelog line.
- Then close it: `gh issue close <n> --comment "Applied in <commit or branch>."`. A question that no longer matters:
  `gh issue close <n> --reason "not planned" --comment "Withdrawn: <why>."`.

## Waiting

Never wait while other work is pickable. With nothing left but an answer to wait for, run
`scripts/docs/questions.sh wait 30` in the background: it exits 0 with the answer when a question of this repo gets a
comment, or 1 after 30 minutes. Then act on it, or stop the session with a summary. For a manual action the work
waits on, `scripts/docs/actions.sh wait <n> 30` does the same: it exits when the action is closed, skipped, gets a
help question or a new comment.

## Hearing the answer

An answer or a changed action reaches the agent three ways, fastest first:

- **Woken in tmux.** `questions.sh ask` and `actions.sh add` run inside tmux put the asking session in the body: the
  visible "Asked from" line (questions) and a hidden `<!-- hq-asker: tmux=<session> host=<host> -->` marker. When
  the maintainer answers, reopens, marks an action done, declines, skips, restores or asks for help in the HQ web
  app, the server types one line into that session, `HQ web: <repo>#<n> <what changed>. Read it with …`
  (the `gh issue view` command), and submits it; a busy session gets it as its next message. Only `tmux` runs, with the
  marker's plain session name and a line the server builds; a session that is gone or sits at a shell prompt is
  left alone. A change made on GitHub itself (a comment without `Via: hq-web`, a close, a new `skipped` or
  `needs-help` label) wakes it too, from the 60 s poll.
- **Its own wait.** A session with nothing else to do runs `questions.sh wait` or `actions.sh wait <n>` in the
  background (above); this is the way for a desktop session, which the server cannot type into.
- **The next round.** A session that ended never hears; the repo's next session (`/fleet`, `/autopilot`, or a
  person) checks `questions.sh answered` and `actions.sh closed` (the last 7 days, with each one's last comment) at
  start and between items, and the backlog item that waits on an action links it.

## Without `gh`

A session that cannot reach GitHub writes the question into the backlog item's Progress (`Question for the
maintainer: …`, same parts as the body above) and names it in its final summary; the next session with `gh` files
it as an issue.

## Manual actions

A step only the maintainer can do (a production deploy the agent may not run, a publish with their token, a login,
a check on a machine the agent cannot reach) is a `manual-action` issue, never only a line in a chat summary: chat
scrolls away, an issue stays visible until it is done.

`scripts/docs/actions.sh add [--repo <owner/repo>] [--no-command] "<what to do>" <body file|->` (in HQ:
`just docs-actions-add`). Title: the action as an imperative, one line. Body (`actions.sh template` prints it):

````markdown
**Why.** What it unblocks or fixes, and why the agent could not do it.

**Run.**

In `~/dev/hq`:

```bash
sudo tailscale set --operator=$USER
```

In `~/dev/hq`:

```bash
just web-tailnet apply
```

**Done when.** What the maintainer sees when it worked, ideally a command and its expected output.

**Waiting on it.** Backlog item, branch or follow-up that continues after it (or "nothing").
````

- The commands follow [Commands](#commands). `add` refuses a body without a command block; `--no-command` is for a
  step that is not a command (a click in a settings page, a login in a browser): then **Run.** names the exact page
  and button instead.
- One action per issue, filed in the repo it concerns (HQ for cross-repo or machine steps).
- The maintainer closes the issue when done (a comment is optional). An agent that sees it done (the check passes)
  may close it with `actions.sh done <n> "<what it saw>"`.
- Not every action gets done. **Decline** (it will not be done) closes the issue as not planned with the reason as
  the comment. **Skip** (not now) keeps it open with the `skipped` label and a comment naming the day it comes back
  (a hidden `<!-- hq-skip-until: … -->` marker), or "until restored"; a skipped action stays out of the counts until
  then. Both are buttons in the HQ web app; on GitHub, closing as not planned or adding the label does the same.
- **Help** (stuck or unclear): the maintainer's question goes on the issue as a comment (`Help needed: …`) with the
  `needs-help` label; the action stays open, shows "Waiting for help" and leaves the maintainer's counts, since it is
  agent work now. An agent reads the issue, answers there with clearer steps or a fix (editing the body when the steps
  were wrong), and drops the label: `actions.sh answer <n> "<reply>"`. Find them with
  `gh search issues --label needs-help --state open`; `actions.sh ls` marks them "(needs help)".
- Work that waits on it continues in a later session: the backlog item links the issue.
- Where to look: the HQ web app's Inbox (every repo, above the questions), `just docs-actions` in HQ
  (`--commands`: each action's commands below it), `scripts/docs/actions.sh ls` in a repo, or the GitHub filter `is:open label:manual-action author:@me`.

## Commands

Whenever the maintainer has to run something, for a manual action or for an option of a question, the body gives the
exact commands, so they paste and run without editing. Retyping a command from prose is where steps go wrong, and the
HQ web app shows each block with a copy button.

- **One command per block**, a fenced `bash` block (```` ```bash ````), in the order to run them. A long command may
  continue over lines with `\`; a pipeline is one command.
- **The folder on the line above**: `` In `~/dev/hq`: `` (a clone of another repo:
  `` In `~/dev/hq/repos/<owner>/<repo>`: ``; any folder: `` In `any folder`: ``). No `cd` inside the block.
- **Paste-ready**: no `$` prompt, no output mixed in (expected output goes in the text after the block), real paths
  and values. A value only the maintainer has (a token, a password) is a named placeholder `<github-token>`, and the
  text says where it comes from; never put a secret in the issue.
- `sudo` where it is needed, and a word on why.
- **Done when.** (manual actions): how the maintainer sees that it worked, as a command when there is one.
- File paths in the text are repo-relative (`scripts/web/tailnet/web-tailnet.py:42`): the HQ web app links them to the
  file and opens them in an IDE.

`actions.sh add` and `questions.sh ask` warn about a block without a folder line, a `$` prompt or several commands in
one block; `actions.sh add` refuses a body with no command block (see [Manual actions](#manual-actions)).

## In a repo

The repo's AGENTS.md says, in one line: questions are `maintainer-question` issues, asked with
`scripts/docs/questions.sh`, and steps only the maintainer can do are `manual-action` issues
(`scripts/docs/actions.sh`), rules in this page.
