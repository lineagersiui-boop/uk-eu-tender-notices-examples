# Export notices as CSV (spreadsheets, Google Sheets, Excel)

Every run writes its notices to a dataset. Apify's API can return any dataset as CSV. `APIFY_TOKEN` is your Apify API token; prefer a dedicated one for scripts.

## One run's notices
Use the run's `defaultDatasetId`, shown in Apify Console on the run's *Storage* tab and in the API response when you start a run:

```bash
curl -sS --fail -H "Authorization: Bearer $APIFY_TOKEN" -o notices.csv \
  "https://api.apify.com/v2/datasets/DATASET_ID/items?format=csv&clean=true&fields=publishedAt,source,title,buyerName,country,valueAmount,valueCurrency,deadline,cpvCodes,noticeUrl,licence"
```

- `fields` picks and orders the columns. Leave it out to get every field, including `description`.
- `cpvCodes` is a list, so the CSV splits it into numbered columns (`cpvCodes/0`, `cpvCodes/1`, ...).
- Use `format=xlsx` for an Excel file, or `format=json` for JSON.
- Keep the `licence` column when you share the data (see the README's attribution section).

## The latest scheduled run
If you run a saved Task on a schedule, this always returns the dataset of its last successful run:

```bash
curl -sS --fail -H "Authorization: Bearer $APIFY_TOKEN" -o notices.csv \
  "https://api.apify.com/v2/actor-tasks/TASK_ID/runs/last/dataset/items?status=SUCCEEDED&format=csv&clean=true"
```

`TASK_ID` is shown in the Task's API panel in Apify Console, or use `your-username~your-task-name`. Without a Task, use `https://api.apify.com/v2/acts/lineagersiui~public-tender-feed/runs/last/dataset/items?status=SUCCEEDED&format=csv&clean=true`, which returns your own last successful run of the Actor.

With `onlyNew`, each run's CSV holds only that run's new notices. To build a running list, append each day's file to a sheet rather than replacing it.

## Google Sheets
- **Manual:** download the CSV and use *File > Import > Upload* with *Append to current sheet*.
- **Automatic:** use Apify's own integrations (for example its Google Sheets, Make or Zapier connections) to push each run's dataset to a sheet.
- `=IMPORTDATA(...)` with the export URL needs the token in the URL (`&token=...`). Anyone who can open or edit the sheet can then read the token, so don't use it in shared sheets.

## Excel
*Data > From Text/CSV* with the downloaded file, or *Data > From Web* with an export URL (the same token caution applies).

Apify's dataset API reference: https://docs.apify.com/api/v2
