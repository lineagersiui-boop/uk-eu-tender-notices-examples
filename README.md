# UK and EU tender notices as CSV, Google Sheets or Slack alerts

Bid teams usually watch several portals by hand: **Find a Tender** and **Contracts Finder** in the UK, and **TED** for the EU. Each has its own search, its own format and its own alert settings, and the same UK opportunity often appears on both UK portals.

This repository has ready-to-use **inputs, a CSV export recipe and a daily-digest workflow** for getting new contract opportunities from all three into one spreadsheet or channel, with [Public Tender Feed](https://apify.com/lineagersiui/public-tender-feed), a pay-per-event Apify Actor that reads the official APIs of those three sources.

It covers **opportunity notices only**. Contract award notices, awarded suppliers, buyer contact details and tender documents aren't included.

## Filters
All filters you set must match (AND). Within one list, any value can match (OR).

| Input | What it does |
|---|---|
| `keywords` | Any keyword appears, case-insensitive, in the title or description. Up to 20. |
| `cpvCodes` | CPV codes or prefixes. Trailing zeros mean the whole branch: `72000000` (or `72`) matches all IT services. Up to 30. |
| `countries` | Buyer countries as ISO two-letter codes. `GB` (or `UK`) selects the UK sources; EU/EEA codes such as `DE`, `FR`, `IE` select TED. Empty means all. |
| `publishedSince` | A date such as `2026-10-20`, or a period such as `7 days`, `2 weeks`, `48 hours`. Default 7 days; at most 90 days back. |
| `minValue`, `maxValue` | Estimated value in the notice's own currency (GBP for UK, usually EUR for TED). No currency conversion. |
| `includeNoValue` | With a value filter, keep notices that publish no value. Off by default, and many notices publish none. |
| `includeClosed` | Keep notices whose tender deadline has passed. Off by default. |
| `onlyNew` | Return only notices not returned before for this filter set (see below). |
| `maxNotices` | Upper limit per run, default 200, at most 1,000. |
| `sources` | `fts` (Find a Tender), `cf` (Contracts Finder), `ted`. Empty means all three. |

**How filtering works per source:** TED filters by keyword, CPV and country on its own servers. The UK APIs have no keyword search, so recent UK notices are read page by page (up to 2,400 Find a Tender and 1,000 Contracts Finder notices per run) and filtered by the Actor. Keywords are re-checked locally for all three sources. A keyword TED can't search, such as `C++`, is matched only locally.

**De-duplication:** UK notices are matched across the two UK portals by OCID, otherwise by title, buyer and deadline, and returned once.

## "Only new" for daily scheduled runs
With `"onlyNew": true`, each run returns only the notices it hasn't returned before for the same filters. That's what you want for a daily digest: schedule it once a day with `publishedSince` of a few days, and each run delivers just the new ones.
- The Actor keeps short digests (not notice text) in a key-value store named `public-tender-feed-state` in **your own** Apify account.
- The memory is per filter set. Changing keywords, CPV codes, countries, sources, value filters or the closed/no-value options starts a fresh list.
- Notices skipped as already seen aren't billed.
- An amendment to a notice that was already returned isn't returned again under `onlyNew`.

## Output fields
One dataset item per notice:

| Field | Content |
|---|---|
| `source` | `find-a-tender`, `contracts-finder` or `ted` |
| `noticeId`, `ocid` | Notice ID; OCID for the UK sources |
| `title`, `description` | Description shortened to 2,000 characters |
| `buyerName`, `country` | Buyer organisation only, ISO country code |
| `cpvCodes` | List of 8-digit CPV codes |
| `valueAmount`, `valueCurrency` | Estimated value where the notice publishes one |
| `publishedAt`, `deadline`, `procedureType` | TED dates are day-only |
| `noticeUrl` | Link to the official notice |
| `licence` | Attribution for that source |

A `SUMMARY` record in the run's key-value store has the counts (`returned`, `duplicatesRemoved`, `alreadySeenSkipped`, ...), each source's status in `sourceStatus`, and billing totals.

## Sample output
Real output from a pre-release test run of the Actor on 2026-10-27 (not the free demo): four of the notices it returned, one from each UK source and two from TED. [`examples/sample-output.json`](examples/sample-output.json). The first item:

```json
{
  "source": "find-a-tender",
  "noticeId": "101112-2026",
  "ocid": "ocds-h6vhtk-07905a",
  "title": "Schools Microsoft Enterprise Licensing Agreement",
  "buyerName": "Newcastle City Council",
  "country": "GB",
  "cpvCodes": ["72000000"],
  "valueAmount": null,
  "valueCurrency": null,
  "publishedAt": "2026-10-26T14:55:55.000Z",
  "deadline": "2026-11-19T12:00:00.000Z",
  "procedureType": "Restricted procedure",
  "noticeUrl": "https://www.find-tender.service.gov.uk/Notice/101112-2026"
}
```
This excerpt leaves out `description` and `licence`. These notices may have closed by the time you read this; they show the format only.

## Examples
- [`examples/input-it-services-uk.json`](examples/input-it-services-uk.json): UK IT and software opportunities, new ones only.
- [`examples/input-construction-eu.json`](examples/input-construction-eu.json): EU construction and design work from TED, with a value floor. The CPV codes are explained in [`examples/cpv-codes.md`](examples/cpv-codes.md).
- [`examples/export-csv.md`](examples/export-csv.md): CSV export, Google Sheets and Excel.
- [`examples/github-actions/daily-digest.yml`](examples/github-actions/daily-digest.yml): a daily GitHub Actions run that posts new notices to Slack and saves a CSV.

## Run it automatically
1. **Apify Task and Schedule (no code).**
   - In Apify Console, open the Actor, enter your filters with `onlyNew` on, and save them as a **Task**.
   - Add a **Schedule** that runs the task daily.
   - Open the dataset's *Tender notices* view, or export it (see `export-csv.md`). Apify's own integrations (for example Slack, e-mail, Make or Zapier) can forward each run's results.
2. **GitHub Actions.** Copy `daily-digest.yml` into `.github/workflows/` of your own repository, add a repository secret `APIFY_TOKEN`, optionally `SLACK_WEBHOOK_URL`, and a `tender-input.json` file in the format of the example inputs. It starts a run, waits for it, posts the new notices to Slack and uploads a CSV.
   - The job fails if the run fails, if the run has no `SUMMARY`, or if no requested source worked. A source that stopped early while others worked adds an "(incomplete: ...)" line to the Slack message.
   - If the job is cancelled or times out, a final step aborts the Apify run.
   - GitHub disables scheduled workflows after 60 days without repository activity.

**Token and visibility:**
- Use a dedicated Apify API token for automation rather than your main one. If your Apify account lets you scope tokens, limit it to running this Actor and reading its runs' datasets and key-value stores. With `onlyNew` it also needs access to the `public-tender-feed-state` key-value store.
- Workflow logs and artifacts of a public repository are visible to anyone. If your bid filters are sensitive, run the workflow in a private repository.
- Optionally pin the actions to commit SHAs.

## Pricing (set on Apify, may change; check the Actor's Pricing tab)
- **Pay per notice returned:** each notice written to your dataset is one `notice-returned` event, **$0.003** at the time of writing.
- **Not billed:** the demo, duplicates removed by de-duplication, notices skipped as already seen (`onlyNew`), empty runs, failed sources and runs stopped for invalid input.
- **Example:** a daily `onlyNew` run that finds 20 new notices a day is about 600 notices a month, roughly $1.80.
- **Cap:** set `maxNotices`, and a maximum charge per run in Apify's run options.
- **Free demo:** leave keywords, CPV codes and countries empty. It returns up to 10 notices from the last 24 hours, each with `demo: true`, and isn't billed. TED gives publication days only, so its demo notices can be from yesterday.

## Limits
- **No completeness guarantee.** The feed holds what the official APIs return during the run, within its limits. Sources can be slow, limited or unavailable; `SUMMARY.sourceStatus` shows what happened for each.
- **The official notice is the authentic version.** Fields are normalised and descriptions are shortened. For TED, the legally authentic text is the one published in the Official Journal (OJ S). Check `noticeUrl` before you bid.
- **Contact details are removed on a best-effort basis.** Contact fields are never read. In free text, e-mail addresses, phone numbers and common forms of contact names are replaced with `[removed]`, but a personal name can occasionally remain. Use the official notice to contact a buyer.
- Each run is capped at 1,000 notices and about 9 minutes of fetching. Up to 2,400 recent Find a Tender and 1,000 Contracts Finder notices are scanned per run, at most one request every 10 seconds to each UK source.
- Values are compared without currency conversion, and many notices publish no value.
- If a source refuses or rate-limits requests (403, 429), or has a server error (5xx), the Actor stops using it for the rest of that run. The one exception: after a rate limit (429 or a "rate limit" message) or a 503 that asks for a wait of up to 30 seconds, it waits and retries once.

## Attribution and licences
Notice data from the Actor, including [`examples/sample-output.json`](examples/sample-output.json), carries these attributions:
- Contains public sector information licensed under the Open Government Licence v3.0. Licence: https://www.nationalarchives.gov.uk/doc/open-government-licence/version/3/
- TED notices © European Union, 1998-2026, reused under the Commission reuse policy; normalised by this Actor. Source: https://ted.europa.eu

Each item's `licence` field repeats the attribution for its source. Keep it when you pass the data on.

## About
- Made by an independent developer. It isn't affiliated with or endorsed by Apify. It's also not affiliated with or endorsed by the UK Government, Crown Commercial Service or the EU Publications Office.
- The Actor and this repository were built with the help of AI tools, under the developer's direction. The examples were checked against the Actor's code, and the sample output comes from a real run.
- Issues and suggestions are welcome in this repository's Issues.

License: MIT (examples and templates in this repository). The MIT licence doesn't cover the notice data in `sample-output.json`, which stays under the licences above.
