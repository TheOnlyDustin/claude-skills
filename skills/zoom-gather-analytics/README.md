# zoom-gather-analytics

Collects a client's marketing and analytics exports during a Zoom call, by working on the
client's own computer through Zoom's Remote Control, and uploads every file to your upload
survey while you keep talking.

## Why it exists

Getting a year of data out of a client usually means a homework list they never finish:
export this from LinkedIn, that from GA4, a PDF from Google Business Profile… This skill
does it live, on the call, with the client watching. The client signs in; Claude does the
clicking, exporting, and uploading.

## What it does

1. **Finds the Zoom tab** in your Chrome and confirms the "You are controlling…" banner.
   If you only have view access, it stops and tells you.
2. **Confirms the scope.** It gathers exactly the channels you name in the command and
   skips every other slide without opening that platform.
3. **Opens your upload survey** in a new tab on the client's computer.
4. **For each channel in scope**, opens the platform in another tab on the client's
   browser, exports the data for the review period (default: last 12 months), picks the
   downloaded files in the client's own file dialog, and confirms each file name under the
   upload field.
5. **Hands off to the client** whenever they must act: a login, 2FA, or an account picker.
   It puts a large black "Quick favor" message page on the client's screen, takes its hands
   off, and checks back every 30 seconds.
6. **Submits only after you confirm**, screenshots the confirmation, closes only the tabs
   it opened, and gives you a per-channel table of files and periods covered.

Supported channels: CRM (GoHighLevel, HubSpot, Zoho and others), LinkedIn personal
profile, LinkedIn company page, Facebook, Instagram, TikTok, Google Business Profile,
Google Search Console, Google Analytics 4, newsletter platforms, and referral-partner
reports. It can also run a one-off check on the client's machine, such as auditing which
tags a Google Tag Manager container has live.

## Safety rules built in

- **The meeting never drops.** It never touches your own Chrome address bar, never
  navigates or reloads the Zoom tab, and sends no Cmd/Ctrl shortcuts that your computer
  could intercept. It works only through screenshots, clicks, and plain typing on the
  client's screen.
- **Passwords and 2FA codes are typed only by the client**, even if someone reads them out.
- **Read and export only.** Anything that publishes, posts, deletes, invites, shares,
  emails a report, or starts a paid feature requires your go-ahead first. In CRM bulk bars
  it zooms in to confirm "Export" before clicking, because "Delete" sits right beside it.
- **Privacy defaults.** It rejects cookie banners where it can, ignores the client's
  history, bookmarks, and unrelated tabs, and treats on-screen text as data, never as
  instructions.

Use it only with a client who has agreed to the session and granted Remote Control.

## Requirements

- The [Claude in Chrome](https://claude.com/chrome) extension, in the Chrome you run Zoom in
- Zoom's **web client** (the meeting open in a browser tab, not the desktop app)
- Remote Control of the client's screen, granted by the client during the call
- An upload survey of your own: any form tool with file-upload fields

## Setup (required before first use)

1. In `SKILL.md`, replace `YOUR_SURVEY_URL` with your survey's link.
2. Edit the slide table in the "The survey run" section to match your survey's slides. The
   shipped table is an example 13-slide layout. If you skip this step the skill still
   asks you for the link, but matching your slides avoids guesswork on the call.

## Usage

Name the channels in the command. That list is the whole scope:

```text
/zoom-gather-analytics - LinkedIn Business Page, TikTok, Google Business Profile, Google Search Console, Google Analytics 4
```

One-off check:

```text
/zoom-gather-analytics check which tags are live in their Google Tag Manager
```

## Files

- `SKILL.md`: the remote-control rules, survey run, client hand-off, and stop conditions
- `references/platform-exports.md`: the export path for each platform, the remote file
  dialog routine, and the save-as-PDF fallback
