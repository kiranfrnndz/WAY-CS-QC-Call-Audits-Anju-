# CS QC Leadership Dashboard

A single-file dashboard for the Customer Success call audit report. Open the page,
choose the audit export, and every metric is calculated in the browser.

## How it works

- `index.html` is the entire application. No build step, no server, no database.
- SheetJS parses the workbook and Chart.js draws the two charts, both from a public CDN.
- Audit data is read into browser memory only. Nothing is uploaded, written to disk or
  persisted between sessions.

## Deploying to GitHub Pages

1. Create the repository and commit `index.html` and this README. Nothing else.
2. Settings > Pages > Deploy from branch > `main` / root.
3. The page is live at `https://<username>.github.io/<repo>/`.

**Rollback:** `git revert <commit>` and push, or re-point Pages at the previous commit.
Pages redeploys in about a minute. There is no state to migrate or restore, because the
application holds no data.

## Third-party libraries

Two libraries are loaded at page load: SheetJS (reads the workbook) and Chart.js (draws
the two charts).

SheetJS is deliberately **not** loaded from cdnjs. The widely-used `xlsx@0.18.5` build on
npm and cdnjs is abandoned and carries CVE-2023-30533, a prototype-pollution flaw
triggered by reading a crafted spreadsheet — the exact code path this page uses. The fix
ships only through the vendor's own CDN, so the page loads it from `cdn.sheetjs.com`.
Check <https://cdn.sheetjs.com> for the current version before pinning.

**For production, vendor both files rather than fetching them:**

```bash
mkdir -p vendor
curl -o vendor/xlsx.full.min.js https://cdn.sheetjs.com/xlsx-0.20.3/package/dist/xlsx.full.min.js
curl -o vendor/chart.umd.min.js https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.min.js
sha256sum vendor/*.js > vendor/CHECKSUMS.txt
```

Then point the two `<script src>` tags at `./vendor/...` and commit the files. This removes
the remaining live dependency: a compromised or hijacked CDN would otherwise be serving
JavaScript into a page that has your audit file open in memory, and CDN-hosted script is
the one route by which that data could leave the device. Vendoring also survives the
corporate network blocking the CDN. Record the checksums, and re-verify on any upgrade.

If you keep the CDN links instead, add Subresource Integrity hashes to both tags so the
browser refuses altered files.

## Data handling

The audit export contains caller phone numbers, confirmation numbers and free-text
remarks. Those columns are read so their absence can be reported, then discarded — they
are never rendered, never charted and never written to the Excel export.

**The audit file must never be committed to this repository.** A `.gitignore` covering
`*.xlsx`, `*.xls` and `*.csv` is the safeguard; check it is present before the first push.
If an audit file is ever committed, treat it as a data disclosure: purge the object from
history, force-push, and raise it through the normal incident path — deleting the file in
a later commit does not remove it from the repository history.

## Updating the dashboard

Upload the latest audit export whenever it changes. The file is cumulative, so the period
filter controls the reporting window:

- **All data in file** — everything in the export
- **Current month** — the calendar month of the latest audit date
- **Last 7 / 14 / 30 days** — rolling windows off the latest audit date
- **Custom range** — any two dates

KPI deltas compare the selected window against the immediately preceding window of the
same length.

## Calculation rules

Metrics follow the QC Leadership Dashboard Logic guide:

- Average QA is the mean of Total Scores. "90% or above" is the share of audits scoring 90+.
- FCR, SOP compliance, CRM accuracy and critical script are the share of audits at the
  fully-compliant answer for each field.
- Handle-time exceptions are "Excessive AHT" or "Excessive/Unnecessary hold time", rated
  against audits where handle time was actually assessed, not against all audits.
- Fields containing "Not applicable" (follow-up, cross-sell, proactive engagement) are
  rated on the applicable population only.
- Tier A–D come from Feedback Category. The dashboard does not create a second scoring model.
- Repeat quality risk is 2+ Tier C/D audits in the period. High = 2+ Tier D or an average
  risk score below 80%; Medium = average risk score below 90%; otherwise Monitor.
- Call reason labels are normalised before any grouping. Every merge is listed on the Data
  quality tab so the grouping can be audited.

## Governance

- Confluence: keep the calculation rules above and the reason-normalisation list in the QC
  space, and update them whenever the audit form changes.
- Jira: raise a ticket for the repository creation and Pages enablement, and for any change
  to scoring logic, tier bands or the priority rule, since those change what leadership sees.
- Any new question added to the audit form needs the column mapping in `index.html` checked
  before the next reporting cycle.
