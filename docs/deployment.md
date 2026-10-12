# Deployment

This document explains how to put the tracker into use: publishing `app/index.html` as a private claude.ai Artifact, updating it later without losing data, setting your search preferences and schedule, creating the hourly routine with the network access it needs, and linking the routine to the page. There is no server and no build step; the page file is published as it is.

## Requirements

- A claude.ai account with Artifacts and Claude Code scheduled routines.
- Claude Code, run from a copy of this repository.
- The Gmail connector enabled on the same claude.ai account, for reply checks and the summary email.
- A cloud environment for the routine with network access to job sites. See [Allow network access to job sites](#allow-network-access-to-job-sites).

## Publish the page

The page must be published as an Artifact with two runtime capabilities:

- `db`: the Artifact database that stores jobs, notifications, preferences, the schedule and search requests.
- `user`: identifies the signed-in viewer.

Steps:

1. Open Claude Code in the folder that contains `app/index.html`.
2. Ask Claude to publish the page, for example:

   ```text
   Publish app/index.html as a private Artifact with the db and user capabilities.
   ```

3. Claude returns a claude.ai artifact URL. Save it: this is your `<YOUR_ARTIFACT_URL>`, used by the routine.
4. Open the URL in a browser where you are signed in to claude.ai. The header should read "Nothing tracked yet" and the Listings tab should show a **Set preferences** prompt.

The artifact is private, so only your account can open it. The page only works inside claude.ai. If you open `app/index.html` directly from disk, it renders but shows "Saving isn't available in this view." and cannot load or save anything.

## Update the page

To change the page, edit `app/index.html` and republish it to the same artifact URL. Data in the Artifact database belongs to the artifact, so an in-place update keeps your jobs, notifications, preferences and schedule.

1. Edit `app/index.html`.
2. In Claude Code, ask Claude to update the existing artifact:

   ```text
   Republish app/index.html to <YOUR_ARTIFACT_URL>, keeping the existing capabilities.
   ```

3. Reload the artifact in your browser.

If you publish to a new URL instead, you get a new, empty database. You must then update `<YOUR_ARTIFACT_URL>` in the routine and store the routine URL again in the new database.

If you change any field name or allowed value, update `routines/hourly-check.md` and [Data model](data-model.md) in the same change.

## Set your search preferences

The listings search does nothing useful until you save at least one job title.

1. Open the tracker and select **Preferences** in the header, or **Set preferences** in the Listings tab.
2. Fill in the fields. Separate multiple entries with commas.
3. Select **Save preferences**.

The routine reads the saved values on its next listings search. Each field is described in the [README configuration section](../README.md#search-preferences) and in [Data model](data-model.md#settingspreferences).

## Set your schedule

1. Select **Schedule** in the header.
2. Choose your time zone, the reply-check window (from and to), the listings hour and the summary hour. Untick a task to turn it off. Hours are whole hours only.
3. Select **Save schedule**.

The routine reads the schedule at the start of every run, so changes apply from the next hourly run. Until you save, the routine and the page use the defaults in the [README schedule table](../README.md#schedule).

## Allow network access to job sites

Routines run in a Claude Code cloud environment, and each environment has a network access setting. The listings search uses `WebFetch` to open each posting, so the environment must allow job sites:

- Full: allows any site. This is the simplest choice.
- Custom: allows only the domains you list. Add the job boards you want searched (for example LinkedIn, Indeed, JobStreet and Glassdoor) and any company career sites you care about.

If the environment blocks these sites, `WebFetch` cannot open postings and the listings task marks the search request `failed`. Choose or edit the environment when you create the routine at claude.ai/code/routines.

## Create the routine

Create one routine from `routines/hourly-check.md`, scheduled every hour (cron `0 * * * *`), with the Gmail connector and an environment that allows job sites. Replace `<YOUR_ARTIFACT_URL>` with your artifact URL and `<YOUR_EMAIL>` with your Gmail address (for example `you@example.com`), then create it at claude.ai/code/routines or with `/schedule` in Claude Code. Step-by-step instructions are in [Create the routine](routines.md#create-the-routine).

The scheduler may start the routine some minutes past the hour (for example at :08). That is expected.

## Store the routine URL

After you save the routine, copy its URL from claude.ai/code/routines. This is your `<YOUR_ROUTINE_URL>`. Store it in the `routineUrl` field of `settings/schedule` so the page can show a **Run it now on claude.ai** link next to a pending search request. The Schedule dialog has no field for it.

The simplest way is to ask Claude in Claude Code, which writes it with the `ArtifactData` tool:

```text
In the artifact <YOUR_ARTIFACT_URL>, update the document settings/schedule and set routineUrl to <YOUR_ROUTINE_URL>. Keep the other fields.
```

The URL must start with `https://`, or the page ignores it. Saving the Schedule dialog later keeps `routineUrl`.

## Check that it works

1. On the Listings tab, select **Find jobs now**. The line under the button should read "Search requested just now. It runs at the next hourly check." followed by a **Run it now on claude.ai** link.
2. Select the link and then **Run now** on the routine page. Within a few minutes the line should change to "Searching for jobs…" and then to a result such as "Last search 2m ago: 4 new listings". New cards should appear with an Added date, and the bell should show a "New listing" notification.
3. Select **Apply** on one card. It should move to the Applied tab with today's date.
4. To test reply checks and the summary without waiting, open **Schedule**, set the reply window or the summary hour to include the current hour, save, and select **Run now** on the routine page again. With no replies, reply checks send nothing but set `lastEmailCheck` on each checked job. The summary sends one email from your own address and adds a "Daily summary" notification. Set the schedule back afterwards.

## Troubleshooting

- "Saving isn't available in this view. Open the tracker on claude.ai while signed in.": the page could not open the database. Open the artifact URL on claude.ai while signed in, and check that the artifact was published with the `db` capability.
- "Lost connection to your saved jobs. Reload the page.": the live connection dropped. Reload the page.
- **Find jobs now** is greyed out: save at least one job title in Preferences. The button is also disabled while a request is pending or running; after 2 hours with no progress it becomes available again.
- No **Run it now on claude.ai** link: `settings/schedule.routineUrl` is not set or does not start with `https://`. See [Store the routine URL](#store-the-routine-url). The link only appears while a request is pending.
- Search stays at "Search requested": the routine has not run since the request. Wait for the next hourly run, or open the routine and select **Run now**. Check that the routine exists and is enabled.
- "Last search failed": read the message after it. "Set your search preferences first" means no job titles are saved. If the message says postings or sites could not be reached, the routine's cloud environment is blocking job sites; set its network access to Full, or Custom with the job board domains. See [Allow network access to job sites](#allow-network-access-to-job-sites).
- No new listings after a successful search: check that your filters (level, salary floor, excluded keywords) are not too narrow. Some job sites block automated access, so results vary.
- Nothing happens at an hour you expected: check the schedule line under the page title. The hours are in the schedule's time zone, not your browser's. Runs outside the reply window and away from the listings and summary hours exit without doing anything.
- No reply detected: reply checks only run inside the reply-check window, only check jobs in Applied, Responded, Interview or No reply, and only search by company name and the `contact` field. Add the recruiter's name or email to `contact` to improve matching.
