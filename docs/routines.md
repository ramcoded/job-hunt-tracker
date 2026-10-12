# Routines

This document covers the single Claude Code scheduled cloud routine that keeps the tracker up to date, defined in `routines/hourly-check.md`: when it runs, how each run decides what to do, what each task does, which connectors, tools and network access it needs, which placeholders you fill in, and how to create or update it.

## About scheduled routines

A routine is a saved prompt that Claude Code runs on a cron schedule in Anthropic's cloud. Because it runs in the cloud, it works while your computer is off. You create routines at claude.ai/code/routines or with the `/schedule` command in Claude Code.

Things to know:

- The shortest interval is one hour.
- Cron expressions are in UTC. Because this routine runs every hour, the time zone of the cron line does not matter; the routine converts to your time zone itself.
- The scheduler may offset the start minute. A routine scheduled for every hour can run at, for example, :08 past each hour. The routine only looks at the hour, so this does not change which tasks run.
- Gmail access goes through the claude.ai Gmail connector, which must be enabled on the same account that owns the routine.
- The routine reads and writes the tracker with the `ArtifactData` tool, which acts as you. Create the routine on the same claude.ai account that owns the artifact.
- The routine runs in a cloud environment. Its network access setting must allow job sites, or `WebFetch` cannot open postings. See [Deployment](deployment.md#allow-network-access-to-job-sites).

## Summary

| Prompt file | Schedule | Cron (UTC) | Connectors | Tools |
| --- | --- | --- | --- | --- |
| `routines/hourly-check.md` | every hour | `0 * * * *` | Gmail | ArtifactData, WebSearch, WebFetch |

The prompt also runs `date -u +%Y-%m-%dT%H:%M:%SZ` in Bash to get the current time for every timestamp it writes.

## Placeholders

The prompt contains placeholders. Replace them before you save the routine.

| Placeholder | Replace with |
| --- | --- |
| `<YOUR_ARTIFACT_URL>` | The claude.ai URL of your published tracker. See [Deployment](deployment.md#publish-the-page). |
| `<YOUR_EMAIL>` | Your Gmail address, for example `you@example.com`. Emails are sent from and to this address. |

Do not commit your filled-in prompt if this repository is public. If you want to keep a local copy, name it with a `.local.md` ending (for example `routines/hourly-check.local.md`); `.gitignore` excludes those files.

## How a run decides what is due

Every run starts with the same check (Step 0 in the prompt):

1. Reads `settings/schedule`. Any missing field falls back to the page defaults: `timezone` `Asia/Shanghai`, `updatesEnabled` true, `updatesFrom` 8, `updatesTo` 23, `listingsEnabled` true, `listingsHour` 8, `summaryEnabled` true, `summaryHour` 21.
2. Works out the current local hour H (0 to 23) and today's date in `timezone`.
3. Reads `settings/searchRequest`. A request counts as pending if `status` is `pending`, or if `status` is `running` and `startedAt` is more than 2 hours ago (a previous run stopped partway).
4. Decides which tasks are due:
   - Listings: a search request is pending, or `listingsEnabled` is true and H equals `listingsHour`.
   - Reply checks: `updatesEnabled` is true and H is inside the window from `updatesFrom` to `updatesTo`, inclusive. If `updatesFrom` is greater than `updatesTo`, the window wraps past midnight: H is inside it when H is at least `updatesFrom` or at most `updatesTo`.
   - Summary: `summaryEnabled` is true and H equals `summaryHour`.
5. If nothing is due, the run exits immediately and changes nothing.

Due tasks run in this order: listings, reply checks, summary.

For example, with the defaults in Asia/Shanghai, the 08:00 run does listings and a reply check, the 09:00 to 20:00 runs do reply checks only, the 21:00 run does a reply check and the summary, and the runs from 00:00 to 07:00 exit straight away.

## Listings

Adds new job listings that match your saved preferences. It runs at `listingsHour`, or at the next run after you select **Find jobs now**.

1. If a search request triggered the run, sets `settings/searchRequest` to `status` `running` with `startedAt`.
2. Reads `settings/preferences`. If the document is missing or `roles` is empty, it marks the request `failed` with the message "Set your search preferences first" and moves on to the next task.
3. Lists every document in `applications` and builds a set of existing jobs by posting URL and by lowercase `company|role`. Skipped jobs count as existing, so they are never suggested again.
4. Searches the web (LinkedIn, Indeed, JobStreet, Glassdoor, company career pages) for postings from the last 7 days that match your roles, locations or remote preference, level and salary floor. It drops postings that mention an excluded keyword or come from an excluded company, and prefers postings that mention your included keywords.
5. Opens each candidate posting with `WebFetch` to confirm it is real, still open and has a working URL. It drops duplicates and jobs already in the tracker.
6. Keeps up to `maxPerDay` of the best matches and creates them in `applications` in one batch, with `status` `listing`, `addedBy` `claude`, a `match` score, a `fit` sentence, and `datePosted` when the posting shows one.
7. If it added any, creates one `listing` notification such as "4 new listings added".
8. If a search request triggered the run, sets the request to `status` `done` with `finishedAt`, `added` (the number of new listings) and a short `message`. On an error it cannot recover from, it sets `status` `failed` with a short reason in `message` instead.

It never invents postings or details, and it sends no email.

## Reply checks

Runs every hour inside the reply-check window. Looks for employer replies in your Gmail and updates the matching jobs.

1. Lists `applications` and keeps jobs whose status is `applied`, `responded`, `interview` or `ghosted`, so a late reply to a job you marked No reply is still caught. For each job it notes `company`, `role`, `contact`, `dateApplied`, `lastEmailCheck` and the document version. If there are none, it skips this task.
2. For each job, searches Gmail for messages since the later of `lastEmailCheck` and `dateApplied` (default: the last 2 hours) that come from or mention the company or the `contact`. It ignores newsletters, job alerts, marketing and automatic "we received your application" messages unless they contain a decision.
3. Classifies each genuine reply: interview invite or scheduling request becomes `interview`, offer becomes `offer`, rejection becomes `rejected`, and any other human reply becomes `responded`. It never moves a job backwards.
4. Updates each job with a reply: new `status` and `history` entry when the status changed, `lastUpdate`, and a one-line dated summary prepended to `notes`. Writes are pinned to the document version so they do not overwrite edits you made at the same time; on a conflict it re-reads and retries once.
5. Creates one `reply` notification per reply, linked to the job with `jobId`.
6. If there was at least one reply, sends you one email with the subject "Job tracker: <n> application update(s)".
7. Sets `lastEmailCheck` on every job it checked.

It never emails anyone except you, and never replies to, labels, archives or deletes your email.

## Summary

Runs at `summaryHour`. Sends you one end-of-day email.

1. Lists `applications` and `notifications`.
2. Works out: listings the routine added today, jobs you applied to today, status changes today grouped by new status, follow-ups overdue, follow-ups due tomorrow, and totals (listings waiting, applied, responded, interviews, offers, response rate). "Today" is the date in the schedule's `timezone`.
3. Sends one email with the subject "Job tracker: daily summary for <Mon D>" and these sections in order: Today at a glance, Replies and status changes, Applied today, New listings, Follow-ups (overdue, then due tomorrow), then the tracker URL. If nothing happened today and no follow-ups are overdue or due, it still sends a two-line email with the totals.
4. Creates one `summary` notification with a one-sentence version of the email.

Definitions the summary uses, which match the page:

- Follow-ups overdue: status `applied`, no `followedUp`, and `dateApplied` 7 or more days ago. These are the same jobs the page lists under **Follow-ups due**.
- Follow-ups due tomorrow: the same conditions, but `dateApplied` exactly 6 days ago.
- Applied: status is `applied`, `responded`, `interview`, `offer`, `rejected` or `ghosted`.
- Responded: the current status or any `history` entry is `responded`, `interview`, `offer` or `rejected`.
- Response rate: responded divided by applied, rounded to a whole percent.

## Create the routine

1. Open `routines/hourly-check.md` and copy everything under the `## Prompt` heading.
2. Replace `<YOUR_ARTIFACT_URL>` and `<YOUR_EMAIL>`.
3. Create the routine in one of two ways:
   - In a browser, go to claude.ai/code/routines and create a new routine.
   - In Claude Code, run the `/schedule` command and describe the routine, or paste the prompt when asked.

   ```text
   /schedule
   ```

4. Paste the prompt and set the schedule to every hour:

   ```text
   0 * * * *
   ```

5. Attach the Gmail connector.
6. Make sure the routine can use `ArtifactData`, `WebSearch` and `WebFetch`.
7. Choose a cloud environment whose network access allows job sites (Full, or Custom with job board domains). See [Deployment](deployment.md#allow-network-access-to-job-sites).
8. Save the routine and copy its URL (`<YOUR_ROUTINE_URL>`). Store it in `settings/schedule.routineUrl` as described in [Deployment](deployment.md#store-the-routine-url).
9. Test it: select **Find jobs now** on the tracker, then select **Run now** on the routine page. Within a few minutes the line under **Find jobs now** should show the result.

## Update the routine

When you change `routines/hourly-check.md`, the saved routine does not change by itself. To apply the change:

1. Copy the new prompt text and fill in the placeholders again.
2. Open the routine at claude.ai/code/routines and replace its prompt, or run `/schedule` in Claude Code and ask Claude to update the routine by name.
3. Run it once by hand to confirm it still works.

If you republish the page to a new artifact URL instead of updating it in place, update `<YOUR_ARTIFACT_URL>` in the routine and set `routineUrl` again in the new artifact's `settings/schedule`.

## Change the time zone or schedule

Use the **Schedule** dialog on the page. It saves `settings/schedule`, and the routine reads it at the start of every run, so changes apply from the next hourly run. You do not need to edit the routine or its cron line. Keep the routine's cron at every hour (`0 * * * *`); a longer interval would skip hours that the schedule expects to run.

The time zone list in the dialog covers common zones. Hours are whole hours only.
