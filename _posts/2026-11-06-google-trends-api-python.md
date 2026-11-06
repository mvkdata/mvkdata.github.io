---
layout: post
title: "How to get Google Trends data with Python, JavaScript or one curl call (Google Trends API alternative)"
description: "Google has no public Trends API and pytrends breaks often. Get interest over time, interest by region, and top and rising related searches as JSON or CSV in a few lines. Tested code inside."
date: 2026-11-06
tags: [google-trends, google-trends-api, python, seo, keyword-research]
---

Google Trends is the best free signal for what people search for: seasonality, which product is gaining, where a term
is popular, and which related searches are rising right now. The catch is that **Google has no public Trends API**.
This guide shows why the usual workarounds break and how to get **Trends data as clean JSON or CSV** with a few lines
of code. Every snippet below was run before publishing, and the full scripts are on
[GitHub](https://github.com/mvkdata/google-trends-api).

## Why the usual approaches break

- **No official API.** Trends data comes from internal endpoints that need a fresh token per request and change
  without notice.
- **Rate limits (HTTP 429).** pytrends and home-made scripts get blocked after a handful of requests from one IP,
  and on cloud servers often from the first one.
- **Browser automation is slow.** Driving Chrome through trends.google.com takes minutes per term and still hits
  blocks.
- **Scales differ per request.** Values are 0-100 *within one request*, so terms fetched separately can't be compared.
  To compare, they must be fetched together (up to 5 per request).

You can build all of this yourself (token handling, rotating proxies, retries). If you'd rather just have the data,
the rest of this guide uses a ready-made [Google Trends scraper and API on Apify](https://apify.com/mvkdata/google-trends-fast)
that already does it with plain HTTP (no browser, a term takes seconds). You pay per result delivered; terms that fail
cost nothing.

## Step 1: get an API token

Create a free Apify account and copy your token from
[Console → Settings → Integrations](https://console.apify.com/settings/integrations). Then:

```bash
export APIFY_TOKEN=your_token
```

## Step 2: one call, no SDK

Two terms in the US over the past 12 months, straight back as JSON:

```bash
curl -s -X POST \
  "https://api.apify.com/v2/acts/mvkdata~google-trends-fast/run-sync-get-dataset-items?token=$APIFY_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"searchTerms": ["bitcoin", "ethereum"], "geo": "US", "timeRange": "today 12-m"}'
```

Each term comes back as one item with 53 weekly points in `interestOverTime_timelineData`, every US state in
`interestBySubregion`, cities in `interestByCity`, and 25 top + 25 rising related searches.

## Step 3: compare terms and save a CSV (Python)

Put the terms on one line with `isMultiple: true` so they share one 0-100 scale, exactly like the chart on
trends.google.com:

```python
# pip install "apify-client>=3"
import csv, datetime, os
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("mvkdata/google-trends-fast").call(run_input={
    "searchTerms": ["iphone, samsung, pixel"],
    "isMultiple": True,
    "geo": "US",
    "timeRange": "today 5-y",
}, logger=None)
item = next(iter(client.dataset(run.default_dataset_id).iterate_items()))
with open("interest_over_time.csv", "w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["date"] + item["terms"])
    for p in item["interestOverTime_timelineData"]:
        day = datetime.datetime.fromtimestamp(int(p["time"]), datetime.timezone.utc).date()
        w.writerow([day] + p["value"])
```

Result: 262 weekly rows, one column per term:

```text
date,iphone,samsung,pixel
2021-10-31,46,31,3
2021-11-07,46,32,3
...
2026-11-01,49,26,4
```

## Step 4: rising searches for keyword research

```python
run = client.actor("mvkdata/google-trends-fast").call(run_input={
    "searchTerms": ["pickleball", "air fryer"],
    "geo": "US",
    "timeRange": "today 3-m",
    "includeCities": False,
}, logger=None)
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item["searchTerm"])
    print("  rising:", [q["query"] for q in item["relatedQueries_rising"][:10]])
    print("  top:   ", [q["query"] for q in item["relatedQueries_top"][:10]])
```

For "pickleball" this gave rising searches like *national pickleball day (+160%)* next to top searches like
*pickleball paddle* and *pickleball near me*. A tip: rising lists sometimes contain unrelated "Breakout" entries.
They are in Google's own data (trends.google.com shows the same ones), so filter them by your own topic words.

## Step 5: where is a term searched most? (JavaScript)

No SDK needed, Node 18+:

```js
const res = await fetch(
  `https://api.apify.com/v2/acts/mvkdata~google-trends-fast/run-sync-get-dataset-items?token=${process.env.APIFY_TOKEN}`,
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ searchTerms: ['pickleball'], geo: 'US', timeRange: 'today 12-m', includeCities: false }),
  },
);
const [item] = await res.json();
for (const r of item.interestBySubregion.slice(0, 10)) console.log(r.geoName, r.value[0]);
```

## Useful options

- `timeRange`: `now 1-H` up to `all` (since 2004); short ranges give hourly points. `customTimeRange` takes exact dates.
- `property`: measure YouTube, News, Images or Shopping searches instead of web search.
- `startUrls`: paste a trends.google.com explore URL and its terms, region, time and category are used as-is.
- `category`: limit interest to one Trends category.

## FAQ

**Is this the official Google Trends API?** No. Google announced an alpha API with very limited access; this reads
the same public data trends.google.com shows.

**What about pytrends?** It's a good library, but it runs from your own IP and breaks when Google changes tokens or
rate-limits you. The approach above retries on fresh IPs for you.

**Is scraping Google Trends legal?** The data is public and aggregated (no personal data). Check your own use case
and local rules.

Full scripts: [github.com/mvkdata/google-trends-api](https://github.com/mvkdata/google-trends-api) ·
Actor: [apify.com/mvkdata/google-trends-fast](https://apify.com/mvkdata/google-trends-fast)
