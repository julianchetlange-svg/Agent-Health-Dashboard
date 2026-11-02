# Agent Health monitor: hourly check

This file is the procedure for the hourly check that keeps `data.json` current. The dashboard (`index.html`) reads `data.json` and nothing else, so the check only ever changes `data.json`.

Runs hourly from 6 AM to 6 PM Central. Each check finishes with a commit and push when something changed, and always updates `generatedAt` so the page can show how recent the last check was.

## Data file

`data.json` holds three lists:

- `agents`: one entry per agent. `check` says where to look for signs of a run. `cadenceHours` is the expected interval. `window` (daily time window, Central) and `skipDays` (0 = Sunday, 6 = Saturday) refine "late".
- `runs`: one entry per observed run: `{agentId, at (epoch ms), status ("ok" | "warn" | "failed"), summary}`. Keep the newest 30 per agent.
- `inputs`: email feeds that start an agent: `{id, agentId, name, lastAt, maxQuietHours, gmail}`.

An agent may have `startAt` (epoch ms). Ignore any push, email or file older than `startAt` for that agent, so setup commits and test emails are not counted as runs.

## Steps

1. Read `data.json`.
2. For each agent with `check.github`: get the repo's last commit time. Do not use the GitHub API or `gh` for this (the scheduled session only has API access to this dashboard repo). Instead read it with a light clone into the scratchpad, which works for any public repo:
   `git clone -q --depth 1 --filter=blob:none --no-checkout https://github.com/<owner>/<repo> <scratch>/<repo>` then `git -C <scratch>/<repo> log -1 --format='%ct %s'` (`%ct` is the commit time in epoch seconds; multiply by 1000). If a clone fails (for example a private repo), treat that repo as unreadable under the warnings rule. If the commit time is newer than the agent's newest run, add a run with `status: "ok"` and a summary such as "Page updated on GitHub (push Mon 10/26, 10:34 AM CT)". Use Central time in summaries.
3. For each agent with `check.gmailSent`: search Gmail for the newest matching sent message. If it is newer than the agent's newest run of that kind, add a run noting the email was sent. Flag in the summary if the email went out before the GitHub push (the page may not be final yet).
4. For each agent with `check.drive`: find the newest matching Drive file. If newer than the newest run, add a run.
5. For each agent with `check.hubLabel`: read the report hub's `index.html` from the same light clone of `julianchetlange-svg/Store-Dev-Facility-Fleet-Landing-Page` used in step 2 (`git -C <scratch>/<repo> checkout HEAD -- index.html`; the github.io site itself is blocked from the scheduled session). Find the `report-card` whose `<h3>` is the label and read its `Updated M/D/YYYY` text. This gives a date only, so only add a run when that date is later than the date of the newest run, and say in the summary that the time is approximate.
6. For each entry in `inputs`: search Gmail with its `gmail` query and set `lastAt` to the newest matching message's date.
7. Mark `status: "warn"` on a run when the page or email itself reports a data problem, such as a missing day of source data. Mark `status: "failed"` only when a report states a failure. A missed run is not a failed run: the page works out late from the schedule.
8. Set `generatedAt` to the current time in epoch milliseconds.
9. If anything other than `generatedAt` changed, commit with a message such as `Update status 2026-11-02 08:00 CT` and push to `main`. If nothing else changed, still push the `generatedAt` update only when the last push was more than 3 hours ago, so the page can show the check is alive without filling the history with commits.

## Labor saved

The dashboard shows labor saved as `runCount` (per agent) times `laborMinutesPerRun` (top level, currently 30). `runCount` is cumulative and is not trimmed with the run history. When step 2 to 5 adds a run for an agent on a Central calendar date that agent has no earlier non-failed run for, add 1 to that agent's `runCount`. Several runs on the same day for one agent (for example the email and the page push) count once. Failed runs do not count.

## Rules

- Never write email contents into `data.json`. Record only the sender, subject, and date or time.
- Only change `data.json`. Do not edit `index.html` unless asked.
- If a source cannot be read (Gmail, Drive or GitHub unavailable), leave that agent's data as it was and note the problem in the `warnings` list at the top level of `data.json`, replacing any older warnings. Clear the list when everything reads cleanly.
- Report back in one or two lines: how many agents are healthy, late or failed, and what changed.
