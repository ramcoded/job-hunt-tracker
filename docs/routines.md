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
- The routine runs in a cloud environment. Its network access setting must allow job sites and feeds, or `WebFetch` cannot open postings and most results end up as unverified near misses. See [Deployment](deployment.md#allow-network-access-to-job-sites).

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

1. Reads `settings/schedule`. Any missing field falls back to the page defaults: `timezone` `Asia/Shanghai`, `updatesEnabled` true, `updatesFrom` 8, `updatesTo` 23, `listingsEnabled` true, `listingsHour` 8, `summaryEnabled` true, `summaryHour` 21, `importEnabled` true.
2. Works out the current local hour H (0 to 23) and today's date in `timezone`.
3. Reads `settings/searchRequest`. A request counts as pending if `status` is `pending`, or if `status` is `running` and `startedAt` is more than 2 hours ago (a previous run stopped partway).
4. Decides which tasks are due:
   - Listings: a search request is pending, or `listingsEnabled` is true and H equals `listingsHour`.
   - Reply checks: `updatesEnabled` is true and H is inside the window from `updatesFrom` to `updatesTo`, inclusive. If `updatesFrom` is greater than `updatesTo`, the window wraps past midnight: H is inside it when H is at least `updatesFrom` or at most `updatesTo`.
   - Import from email: `importEnabled` is true and H is inside the reply-check window, by the same rule as reply checks (so it also needs `updatesEnabled` to be true).
   - Summary: `summaryEnabled` is true and H equals `summaryHour`.
5. If nothing is due, the run exits immediately and changes nothing.

Due tasks run in this order: listings, import from email, reply checks, summary. The import runs before reply checks so that jobs it imports are checked for replies in the same run.

For example, with the defaults in Asia/Shanghai, the 08:00 run does listings, an email import and a reply check, the 09:00 to 20:00 runs do an email import and a reply check, the 21:00 run does an email import, a reply check and the summary, and the runs from 00:00 to 07:00 exit straight away.

## Listings

Adds new job listings that match your saved preferences and reports what it found. It runs at `listingsHour`, or at the next run after you select **Find jobs now**.

1. If a search request triggered the run, sets `settings/searchRequest` to `status` `running` with `startedAt`.
2. Reads `settings/preferences`. If the document is missing or `roles` is empty, it marks the request (if any) `failed` with the message "Set your search preferences first" and moves on to the next task.
3. Lists every document in `applications`, including skipped jobs, and builds a duplicate index with two keys per job: the normalized posting URL and the normalized `company|role`. See [Duplicate keys](data-model.md#duplicate-keys). Skipped jobs count as existing, so they are never suggested again.
4. Gathers candidates from every source and records each source's outcome as `{name, status, found, note}`. `status` is `ok`, `empty`, `blocked` (HTTP 401, 403 or 429, a captcha, or a DNS failure) or `error`. It tries every source even when one is blocked:
   - OnlineJobs.ph search pages, one search per role.
   - The Remotive and Himalayas remote job feeds, keeping only jobs open to your locations (worldwide, Asia, APAC or your country).
   - A web search for your roles, locations and level, covering LinkedIn, Indeed, JobStreet, Glassdoor, Kalibrr, company career pages and ATS pages such as Greenhouse, Lever, Workable and Ashby.
5. Tries to open each candidate posting and classifies it as exactly one of:
   - duplicate: matches the duplicate index or a candidate already accepted in this run.
   - closed: the posting says it is closed, removed or no longer accepting applications.
   - old: posted more than 7 days ago.
   - mismatched: wrong level or location, mentions an excluded keyword, comes from an excluded company, or shows a salary below your floor.
   - unverified: the posting could not be opened, but the search result clearly names a company, role and direct posting URL.
   - good: open, posted within the last 7 days, and matches.
6. Adds the best `good` candidates, up to `maxPerDay`, to `applications` in one batch, with `status` `listing`, `addedBy` `claude`, a `match` score, a `fit` sentence, and `datePosted` when the posting shows one.
7. Picks up to 15 near misses: candidates that are old but posted within the last 60 days, mismatched on one rule only, or unverified. Each has a short `reason`, such as "Posted 34 days ago" or "Couldn't open to confirm it is still open". Duplicates and closed jobs are never near misses.
8. If it added any, creates one `listing` notification such as "4 new listings added".
9. Writes the result to `settings/searchRequest` on every listings run, whether or not a request triggered it: `status` `done` (or `failed` only if every source failed or an unrecoverable error occurred), `finishedAt`, `added`, a short `message`, and a `report` with the per-source outcomes, the counts from step 5 and the near misses. It keeps `requestedAt` as it is. The write is pinned to the document version and retried once on a conflict. The page shows this in its **Last search** block. The full shape is in [Data model](data-model.md#report).

It never invents postings or details, and it sends no email.

What to expect from the sources:

- JobStreet and Indeed block automated access (HTTP 403) whatever the network setting, so they usually appear as `blocked` in the report.
- The routine needs network access to open postings. Without it, most candidates cannot be verified and appear as unverified near misses rather than listings. See [Deployment](deployment.md#allow-network-access-to-job-sites).

## Import from email

Runs every hour inside the reply-check window while `importEnabled` is on. Finds application confirmation emails in your Gmail for jobs you applied to outside the tracker, and adds or updates those jobs so they appear in the Applied tab without you entering them by hand. New jobs go straight to `applied`; there is no review step.

1. Reads `settings/emailImport`. The search window starts 1 hour before `lastScanAt`; the overlap is harmless because step 4 skips repeats. If `lastScanAt` is missing (the first run), it searches the last 21 days.
2. Searches Gmail for application confirmations, such as:
   - Subjects or text like "Your application was sent to" (LinkedIn), "Application submitted" or "application has been submitted" (Indeed, JobStreet, Kalibrr, OnlineJobs.ph), "Thank you for applying", "We received your application" and "application received".
   - Senders from job boards and applicant tracking systems: linkedin.com, indeed.com, jobstreet.com, kalibrr.com, onlinejobs.ph, greenhouse.io, lever.co, workable.com and workablemail.com, ashbyhq.com, Workday, SmartRecruiters and iCIMS.

   It leaves out job alerts, newsletters, recommendations such as "jobs you may like", and emails that are rejections or interview invites. Those decisions are handled by reply checks.
3. Reads each confirmation for the company, role, date applied (the email's date in the schedule `timezone`), posting URL if present (`http` or `https` only, preferring a direct posting link over tracking or unsubscribe links), source (the site name, such as "LinkedIn", "Indeed", "Greenhouse" or "Company site") and location if stated. If the company or role cannot be read clearly, it skips the email and counts it as unreadable. It never guesses or invents details.
4. Skips any email whose Gmail message ID is already stored in a job's `emailMessageId`. It matches the rest against every job in `applications`, and against jobs imported earlier in the same run, using the [duplicate keys](data-model.md#duplicate-keys):
   - Match in `listing` (or a missing or legacy `found` status) or `skipped`: moves the job to `applied`. Sets `dateApplied` to the email date only if it is empty, appends an `applied` entry to `history`, sets `emailMessageId`, `importedAt` (today) and `lastUpdate`, and prepends the note line `<YYYY-MM-DD> Applied outside the tracker (from email: <subject>)`. It keeps `addedBy` as it is. Counted as moved to Applied.
   - Match in `applied`, `responded`, `interview`, `offer`, `rejected` or `ghosted`: leaves the status alone and only sets `emailMessageId` if it is empty. Counted as already tracked.
   - No match: creates a new job with `status` `applied`, `addedBy` `email`, `dateApplied` set to the email date, `dateFound` and `importedAt` set to today, `match` `null`, `datePosted` `null`, empty `salary` and `fit`, the note `<YYYY-MM-DD> Imported from email: <subject>`, and `emailMessageId`. `contact` is the sender only when that is a real person, not a no-reply address. Counted as added.
5. Writes updates pinned to the document version and retries once on a conflict. New jobs are created in one batch.
6. If it added or moved at least one job, creates one `import` notification, such as "3 applications imported from email (1 moved from Listings)".
7. Writes `settings/emailImport` on every run: `lastResult` with the counts (emails found, added, moved to Applied, already tracked, unreadable) and, only if the Gmail search succeeded, a new `lastScanAt`. The page shows this result in the **Schedule** dialog. The full shape is in [Data model](data-model.md#settingsemailimport).

It only reads your Gmail. It never replies to, labels, archives or deletes email, and it sends no email itself.

To turn the import off, clear **Import applications from email** in the **Schedule** dialog. Because it uses the reply-check window, it also stops when reply checks are off.

## Reply checks

Runs every hour inside the reply-check window. Looks for employer replies in your Gmail and updates the matching jobs.

1. Lists `applications` and keeps jobs whose status is `applied`, `responded`, `interview` or `ghosted`, so a late reply to a job you marked No reply is still caught. For each job it notes `company`, `role`, `contact`, `dateApplied`, `lastEmailCheck` and the document version. If there are none, it skips this task.
2. For each job, searches Gmail for messages since the later of `lastEmailCheck` and `dateApplied` (default: the last 2 hours) that come from or mention the company or the `contact`. It ignores newsletters, job alerts, marketing and automatic "we received your application" messages unless they contain a decision. Those confirmation emails are handled by the [import task](#import-from-email) instead.
3. Classifies each genuine reply: interview invite or scheduling request becomes `interview`, offer becomes `offer`, rejection becomes `rejected`, and any other human reply becomes `responded`. It never moves a job backwards.
4. Updates each job with a reply: new `status` and `history` entry when the status changed, `lastUpdate`, and a one-line dated summary prepended to `notes`. Writes are pinned to the document version so they do not overwrite edits you made at the same time; on a conflict it re-reads and retries once.
5. Creates one `reply` notification per reply, linked to the job with `jobId`.
6. If there was at least one reply, sends you one email with the subject "Job tracker: <n> application update(s)".
7. Sets `lastEmailCheck` on every job it checked.

It never emails anyone except you, and never replies to, labels, archives or deletes your email.

## Summary

Runs at `summaryHour`. Sends you one end-of-day email.

1. Lists `applications` and `notifications`.
2. Works out: listings the routine added today, jobs you applied to today, jobs imported from email today (`importedAt` is today), status changes today grouped by new status, follow-ups overdue, follow-ups due tomorrow, and totals (listings waiting, applied, responded, interviews, offers, response rate). "Today" is the date in the schedule's `timezone`.
3. Sends one email with the subject "Job tracker: daily summary for <Mon D>" and these sections in order: Today at a glance, Replies and status changes, Applied today, Imported from email (each job as "company — role" marked "new" or "moved from Listings"; left out when there are none), New listings, Follow-ups (overdue, then due tomorrow), then the tracker URL. If nothing happened today and no follow-ups are overdue or due, it still sends a two-line email with the totals.
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
9. Test it: select **Find jobs now** on the tracker, then select **Run now** on the routine page. Within a few minutes the line under **Find jobs now** should show the result, and the **Last search** block should list each source with its outcome.

## Update the routine

When you change `routines/hourly-check.md`, the saved routine does not change by itself. To apply the change:

1. Copy the new prompt text and fill in the placeholders again.
2. Open the routine at claude.ai/code/routines and replace its prompt, or run `/schedule` in Claude Code and ask Claude to update the routine by name.
3. Run it once by hand to confirm it still works.

If you republish the page to a new artifact URL instead of updating it in place, update `<YOUR_ARTIFACT_URL>` in the routine and set `routineUrl` again in the new artifact's `settings/schedule`.

## Change the time zone or schedule

Use the **Schedule** dialog on the page. It saves `settings/schedule`, and the routine reads it at the start of every run, so changes apply from the next hourly run. You do not need to edit the routine or its cron line. Keep the routine's cron at every hour (`0 * * * *`); a longer interval would skip hours that the schedule expects to run.

The time zone list in the dialog covers common zones. Hours are whole hours only.
