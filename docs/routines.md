# Routines

This document covers the three Claude Code scheduled cloud routines that keep the tracker up to date: what each one does, when it runs, which connectors and tools it needs, which placeholders you fill in, and how to create, update or reschedule it. The full prompts live in the `routines/` folder.

## About scheduled routines

A routine is a saved prompt that Claude Code runs on a cron schedule in Anthropic's cloud. Because it runs in the cloud, it works while your computer is off. You create routines at claude.ai/code/routines or with the `/schedule` command in Claude Code.

Things to know:

- The shortest interval is one hour.
- Cron expressions are in UTC. The prompts and this document give times in Asia/Shanghai (UTC+8, no daylight saving).
- Gmail access goes through the claude.ai Gmail connector, which must be enabled on the same account that owns the routine.
- The routines read and write the tracker with the `ArtifactData` tool, which acts as you. Create the routines on the same claude.ai account that owns the artifact.

## Summary

| Routine | Prompt file | Schedule (Asia/Shanghai) | Cron (UTC) | Connectors | Tools |
| --- | --- | --- | --- | --- | --- |
| Daily listings | `routines/daily-listings.md` | 08:00 daily | `0 0 * * *` | none | ArtifactData, WebSearch, WebFetch |
| Application updates | `routines/application-updates.md` | hourly, 08:00 to 23:00 | `0 0-15 * * *` | Gmail | ArtifactData |
| Daily summary | `routines/daily-summary.md` | 21:00 daily | `0 13 * * *` | Gmail | ArtifactData |

## Placeholders

Each prompt contains placeholders. Replace them before you save the routine.

| Placeholder | Used in | Replace with |
| --- | --- | --- |
| `<YOUR_ARTIFACT_URL>` | all three | The claude.ai URL of your published tracker. See [Deployment](deployment.md#publish-the-page). |
| `<YOUR_EMAIL>` | application updates, daily summary | Your Gmail address, for example `you@example.com`. Emails are sent from and to this address. |

Do not commit your filled-in prompts if this repository is public. If you want to keep local copies, name them with a `.local.md` ending (for example `routines/daily-listings.local.md`); `.gitignore` excludes those files.

## Daily listings

Runs at 08:00 Asia/Shanghai. Adds new job listings that match your saved preferences.

1. Reads `settings/preferences`. If the document is missing or `roles` is empty, it stops.
2. Lists every document in `applications` and builds a set of existing jobs by posting URL and by lowercase `company|role`. Skipped jobs count as existing, so they are never suggested again.
3. Searches the web (job boards, LinkedIn, Indeed, JobStreet, company career pages) for postings from the last 7 days that match your roles, locations or remote preference, level and salary floor. It drops postings that mention an excluded keyword or come from an excluded company, and prefers postings that mention your included keywords.
4. Opens each candidate posting to confirm it is real, still open and has a working URL. It drops duplicates and jobs already in the tracker.
5. Keeps up to `maxPerDay` of the best matches and creates them in `applications` in one batch, with `status` set to `listing`, `addedBy` set to `claude`, a `match` score and a `fit` sentence.
6. If it added any, creates one `listing` notification such as "4 new listings added".

It never invents postings or details, and it sends no email.

## Application updates

Runs on the hour from 08:00 to 23:00 Asia/Shanghai. Looks for employer replies in your Gmail and updates the matching jobs.

1. Lists `applications` and keeps jobs whose status is `applied`, `responded`, `interview` or `ghosted`, so a late reply to a job you marked No reply is still caught. For each job it notes `company`, `role`, `contact`, `dateApplied`, `lastEmailCheck` and the document version. If there are none, it stops without sending anything.
2. For each job, searches Gmail for messages since the later of `lastEmailCheck` and `dateApplied` (default: the last 2 hours) that come from or mention the company or the `contact`. It ignores newsletters, job alerts, marketing and automatic "we received your application" messages unless they contain a decision.
3. Classifies each genuine reply: interview invite or scheduling request becomes `interview`, offer becomes `offer`, rejection becomes `rejected`, and any other human reply becomes `responded`. It never moves a job backwards.
4. Updates each job with a reply: new `status` and `history` entry when the status changed, `lastUpdate`, and a one-line dated summary prepended to `notes`. Writes are pinned to the document version so they do not overwrite edits you made at the same time; on a conflict it re-reads and retries once.
5. Creates one `reply` notification per reply, linked to the job with `jobId`.
6. If there was at least one reply, sends you one email with the subject "Job tracker: <n> application update(s)".
7. Sets `lastEmailCheck` on every job it checked.

It never emails anyone except you, and never replies to, labels, archives or deletes your email.

## Daily summary

Runs at 21:00 Asia/Shanghai. Sends you one end-of-day email.

1. Lists `applications` and `notifications`.
2. Works out: listings the routine added today, jobs you applied to today, status changes today grouped by new status, follow-ups overdue, follow-ups due tomorrow, and totals (listings waiting, applied, responded, interviews, offers, response rate).
3. Sends one email with the subject "Job tracker: daily summary for <Mon D>" and these sections in order: Today at a glance, Replies and status changes, Applied today, New listings, Follow-ups (overdue, then due tomorrow), then the tracker URL. If nothing happened today and no follow-ups are overdue or due, it still sends a two-line email with the totals.
4. Creates one `summary` notification with a one-sentence version of the email.

Definitions the summary uses, which match the page:

- Follow-ups overdue: status `applied`, no `followedUp`, and `dateApplied` 7 or more days ago. These are the same jobs the page lists under **Follow-ups due**.
- Follow-ups due tomorrow: the same conditions, but `dateApplied` exactly 6 days ago.
- Applied: status is `applied`, `responded`, `interview`, `offer`, `rejected` or `ghosted`.
- Responded: the current status or any `history` entry is `responded`, `interview`, `offer` or `rejected`.
- Response rate: responded divided by applied, rounded to a whole percent.

## Create a routine

Repeat these steps for each of the three prompt files.

1. Open the prompt file in `routines/` and copy everything under the `## Prompt` heading.
2. Replace `<YOUR_ARTIFACT_URL>` and, where present, `<YOUR_EMAIL>`.
3. Create the routine in one of two ways:
   - In a browser, go to claude.ai/code/routines and create a new routine.
   - In Claude Code, run the `/schedule` command and describe the routine, or paste the prompt when asked.

   ```text
   /schedule
   ```

4. Paste the prompt and set the schedule to the cron expression from the [summary table](#summary).
5. For Application updates and Daily summary, attach the Gmail connector.
6. Make sure the routine can use the tools listed in the summary table: `ArtifactData` for all three, plus `WebSearch` and `WebFetch` for Daily listings.
7. Save the routine, then run it once by hand to check that it can reach the tracker and, where needed, Gmail.

## Update a routine

When you change a prompt file in `routines/`, the saved routine does not change by itself. To apply the change:

1. Copy the new prompt text and fill in the placeholders again.
2. Open the routine at claude.ai/code/routines and replace its prompt, or run `/schedule` in Claude Code and ask Claude to update the routine by name.
3. Run it once by hand to confirm it still works.

If you republish the page to a new artifact URL instead of updating it in place, update `<YOUR_ARTIFACT_URL>` in all three routines.

## Change the time zone or schedule

The time zone appears in two places in each routine, and both must agree:

- The cron expression, which is always UTC.
- The prompt text, which says "Asia/Shanghai" wherever it defines "today".

To move to another time zone, convert your local times to UTC for the cron expression and replace "Asia/Shanghai" in the prompt with your IANA time zone name, for example `Europe/Berlin`. For zones with daylight saving, the UTC cron does not shift with the clocks, so the local run time moves by an hour twice a year.

For example, to run the summary at 20:00 in UTC+1:

```text
0 19 * * *
```

Keep the application updates routine at an hourly interval or longer.
