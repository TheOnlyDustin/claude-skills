# Platform exports on the client's computer

Each survey slide prints the client-facing recipe. These notes add what helps when you
drive it remotely. Menus move over time, so trust the screen over these notes. Default
period: last 12 months, unless the host set another.

## Shared routines

**Finding the download.** After clicking an export, wait and screenshot. Chrome and Edge
show a download icon or bubble near the client's address bar. `zoom` to read the file
name and note it, because names repeat (`data-export.csv`, `data-export (1).csv`). If a
"Save as" dialog appears, save to Downloads with the suggested name.

**Uploading on the survey slide.** Click the slide's upload area. The client's own OS
file dialog opens on their screen (Finder on macOS, File Explorer on Windows). Click
**Downloads** in its sidebar, switch to list view if needed, and sort by date modified so
the newest files are on top. Click the file you noted, `zoom` to confirm the name, then
click Open. Repeat once per file rather than multi-selecting. Confirm each name appears
under the field. If the dialog opens somewhere unexpected or won't respond, press Escape
once, screenshot, and tell the host.

**Fallback when there is no export button.** Several slides accept screenshots or PDFs.
Make a PDF on the client's computer through visible menus. In Chrome or Edge, use the
three-dot menu, then Print, set Destination to "Save as PDF", click Save, and keep
Downloads as the location. In Safari, use File, then Export as PDF. Before saving, set the
page so the account name, date range and metric labels are visible. Your own screenshots
live on the host's computer and cannot be uploaded through the client's file dialog.

## Per channel

**CRM** (GoHighLevel, Deal Studio, HubSpot, Zoho and others). The survey wants three
exports: (1) contacts CSV for the period, with created date, source and tags; (2)
opportunities or pipeline CSV, with stage and value; (3) an appointment report export, or
a PDF if there is none. In GoHighLevel, select contacts with the header checkbox, choose
select all, then use the Export action in the bulk-action bar. Zoom first, because Delete
sits beside it. Some CRMs (HubSpot among them) email the export instead of downloading it.
Tell the host rather than opening the client's email.

**LinkedIn – personal profile.** Me, then Analytics (or "Post impressions"). Set the date
range, then Export (XLSX). If Export isn't offered, save PDFs that show the profile name,
dates, totals and post activity.

**LinkedIn – company page.** Switch to the Page admin view. Under Analytics, export
Content, Visitors and Followers (XLS) for the period, one export per tab. This is separate
from the personal profile.

**Facebook.** Meta Business Suite, then Insights, then Export data. Choose the Facebook
Page and the period, then CSV. If export is unavailable, save PDFs of Results and Content
with the account name, date range and metric labels visible.

**Instagram.** Meta Business Suite, then Insights, then Export data. Choose the Instagram
account, then CSV. Alternatively, save PDFs of the Professional dashboard (reach,
engagement, profile activity). The account doesn't need to be linked to Facebook.

**TikTok.** On desktop: TikTok Studio, then Analytics, then Download data, for both
Overview and Content. TikTok caps the date range, so download consecutive periods to cover
the review period. Keep the actual dates visible in any PDF.

**Google Business Profile.** Search the business name on Google while signed in, or open
Business Profile Manager. Under Performance, pick the period, then Download. It should
cover views, searches, calls, website clicks and directions. Do not use the spreadsheet
for editing business details.

**Google Search Console.** Pick the property that matches the client's website. Go to
Performance, then Search results, set Last 12 months, then Export. Choose Download CSV or
Excel, not "Export to Google Sheets", which would create a file in the client's Drive.

**Google Analytics 4.** Use the existing property. Go to Reports, then Acquisition, then
Traffic acquisition, and set Last 12 months. Then Share this report, Download file, CSV.
If the client tracks form fills, repeat for Engagement, then Events. Never create a
property.

**Newsletter / email marketing.** The client signs in first (client hand-off), then you take over. In the
campaign reports, export the period as CSV: every send, with send date, recipients, opens,
clicks and unsubscribes. With no export button, save PDFs of the reports list. Also ask
the host for the current list size and date checked, and any replies or enquiries the
newsletter drove. The slide asks for these, but there is no text field, so report them to
the host.

## Ad-hoc check: Google Tag Manager

Confirm the container matches the client's site. Tags list: GA4 shows as a **Google Tag**
with a `G-` ID or **GA4 Event** tags. Meta Pixel shows as a "Facebook/Meta Pixel" template
tag or a **Custom HTML** tag containing `fbq('init', '<id>')`. Check **Versions** for what
is live versus unpublished workspace changes. Avoid Submit, Publish, Save and Preview.
Preview opens Tag Assistant windows.

## Referral partners (only when the host names it)

The client signs in to the software that tracks referrals, through the client hand-off,
then you take over. The slide wants the last 12 months, one row per seller lead or listing
that came from a referral, with the date, the partner type (accountant, attorney, banker,
past client and so on) and the outcome (meeting, listing, closed, no deal). It must have
no client names or contact details. Export the closest matching report, or the client's
existing spreadsheet if the host points to it. If the only available export includes
names or contact details, tell the host before uploading.
