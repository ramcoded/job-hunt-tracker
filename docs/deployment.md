# Deployment

This document explains how to put the tracker into use: publishing `app/index.html` as a private claude.ai Artifact, updating it later without losing data, setting your search preferences, and creating the scheduled routines. There is no server and no build step; the page file is published as it is.

## Requirements

- A claude.ai account with Artifacts and Claude Code scheduled routines.
- Claude Code, run from a copy of this repository.
- The Gmail connector enabled on the same claude.ai account, for the two routines that use Gmail.

## Publish the page

The page must be published as an Artifact with two runtime capabilities:

- `db`: the Artifact database that stores jobs, notifications and preferences.
- `user`: identifies the signed-in viewer.

Steps:

1. Open Claude Code in the folder that contains `app/index.html`.
2. Ask Claude to publish the page, for example:

   ```text
   Publish app/index.html as a private Artifact with the db and user capabilities.
   ```

3. Claude returns a claude.ai artifact URL. Save it: this is your `<YOUR_ARTIFACT_URL>`, used by the routines.
4. Open the URL in a browser where you are signed in to claude.ai. The header should read "Nothing tracked yet" and the Listings tab should show a **Set preferences** prompt.

The artifact is private, so only your account can open it. The page only works inside claude.ai. If you open `app/index.html` directly from disk, it renders but shows "Saving isn't available in this view." and cannot load or save anything.

## Update the page

To change the page, edit `app/index.html` and republish it to the same artifact URL. Data in the Artifact database belongs to the artifact, so an in-place update keeps your jobs, notifications and preferences.

1. Edit `app/index.html`.
2. In Claude Code, ask Claude to update the existing artifact:

   ```text
   Republish app/index.html to <YOUR_ARTIFACT_URL>, keeping the existing capabilities.
   ```

3. Reload the artifact in your browser.

If you publish to a new URL instead, you get a new, empty database, and you must update the URL in all three routines.

If you change any field name or allowed value, update the routine prompts and [Data model](data-model.md) in the same change.

## Set your search preferences

The daily listings routine does nothing until you save at least one job title.

1. Open the tracker and select **Preferences** in the header, or **Set preferences** in the Listings tab.
2. Fill in the fields. Separate multiple entries with commas.
3. Select **Save preferences**.

The routine reads the saved values on its next run. Each field is described in the [README configuration table](../README.md#configuration) and in [Data model](data-model.md#settingspreferences).

## Create the routines

Create the three routines described in [Routines](routines.md):

1. Daily listings, from `routines/daily-listings.md`, cron `0 0 * * *`.
2. Application updates, from `routines/application-updates.md`, cron `0 0-15 * * *`, with the Gmail connector.
3. Daily summary, from `routines/daily-summary.md`, cron `0 13 * * *`, with the Gmail connector.

For each one, replace `<YOUR_ARTIFACT_URL>` with your artifact URL and `<YOUR_EMAIL>` with your Gmail address (for example `you@example.com`), then create it at claude.ai/code/routines or with `/schedule` in Claude Code. Step-by-step instructions are in [Create a routine](routines.md#create-a-routine).

## Check that it works

1. Run Daily listings by hand. New cards should appear in the Listings tab, and the bell should show a "New listing" notification.
2. Select **Apply** on one card. It should move to the Applied tab with today's date.
3. Run Application updates by hand. With no replies it sends nothing, but `lastEmailCheck` is set on each checked job.
4. Run Daily summary by hand. You should get one email from your own address and a "Daily summary" notification under the bell.

## Troubleshooting

- "Saving isn't available in this view. Open the tracker on claude.ai while signed in.": the page could not open the database. Open the artifact URL on claude.ai while signed in, and check that the artifact was published with the `db` capability.
- "Lost connection to your saved jobs. Reload the page.": the live connection dropped. Reload the page.
- No new listings after a run: check that `roles` is set in Preferences, and that your filters (level, salary floor, excluded keywords) are not too narrow. Some job sites cannot be reached, so results vary.
- No reply detected: the routine only checks jobs in Applied, Responded or Interview, and only searches by company name and the `contact` field. Add the recruiter's name or email to `contact` to improve matching.
