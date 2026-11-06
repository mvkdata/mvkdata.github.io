---
layout: post
title: "How to scrape every Walmart review (and product data) with Python, JavaScript or one curl call"
description: "Get all customer reviews, product details, prices and search results from walmart.com as JSON or CSV, including star filters and results past Walmart's 10-page limit. Tested code inside."
date: 2026-11-06
tags: [walmart, web-scraping, python, reviews, price-monitoring]
---

Walmart product pages hold some of the most useful public e-commerce data there is: prices and sellers that change
daily, UPCs and specs, and thousands of customer reviews per popular product. Getting that data out reliably is harder
than it looks. This guide shows what makes it hard and how to get **every review, product and search result as clean
JSON or CSV** in a few lines of code. Every snippet below was run before publishing.

## Why a quick script usually fails

- **Bot protection.** Walmart answers fast automated traffic with a "Robot or human?" page. A script that works for
  20 requests from your laptop often gets your IP blocked for hours after a few hundred.
- **The 10-page limit.** Search and category pages stop after page 10, roughly 400 products, even when Walmart says
  there are thousands of matches. A TV category with 1,357 matches gives you 400 rows if you just follow "next page".
- **Reviews are paged 10 at a time**, and pages overlap, so counting pages does not give you every review.
- **The data is nested.** Prices, sellers, variants and review summaries sit deep in large JSON blobs whose shape
  differs between page types.

You can solve all of this yourself (rotating proxies, retries, splitting searches into price ranges, de-duplication).
If you would rather just have the data, the rest of this guide uses a ready-made
[Walmart scraper and API on Apify](https://apify.com/mvkdata/walmart-scraper-fast) that already does it; you pay per
row delivered.

## Step 1: get an API token

Create a free Apify account and copy your token from
[Console → Settings → Integrations](https://console.apify.com/settings/integrations). Then:

```bash
export APIFY_TOKEN=your_token
```

## Step 2: one call, no SDK

This returns the product plus its 1- and 2-star reviews, newest first, directly as JSON:

```bash
curl -s -X POST \
  "https://api.apify.com/v2/acts/mvkdata~walmart-scraper-fast/run-sync-get-dataset-items?token=$APIFY_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "startUrls": [{ "url": "https://www.walmart.com/ip/5113175776" }],
    "includeReviews": true,
    "maxReviewsPerProduct": 20,
    "reviewsStarFilter": ["1", "2"],
    "reviewsSort": "submission-desc"
  }'
```

Every row has a `type` (`product`, `searchResult` or `review`), so one run can mix products, searches and reviews.

## Step 3: every negative review into a CSV (Python)

```python
import csv, os
from apify_client import ApifyClient   # pip install "apify-client>=3"

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("mvkdata/walmart-scraper-fast").call(run_input={
    "startUrls": [{"url": "https://www.walmart.com/ip/5113175776"}],
    "includeReviews": True,
    "maxReviewsPerProduct": 0,          # 0 = all reviews
    "reviewsStarFilter": ["1", "2"],
})
rows = [r for r in client.dataset(run.default_dataset_id).iterate_items() if r["type"] == "review"]
with open("reviews.csv", "w", newline="", encoding="utf-8") as f:
    w = csv.DictWriter(f, ["date", "rating", "title", "text", "verifiedPurchase", "helpfulVotes"], extrasaction="ignore")
    w.writeheader()
    w.writerows(rows)
print(len(rows), "reviews")
```

On the Chromebook above this wrote 607 one- and two-star reviews. Without the star filter the same product returns all
4,255 written reviews.

**Finding the recurring complaints** takes a few more lines:

```python
import collections, re
words = collections.Counter(
    w for r in rows for w in re.findall(r"[a-z']{4,}", (r.get("text") or "").lower())
)
print(words.most_common(30))   # "charge", "screen", "battery", "slow" ... tell you what breaks
```

## Step 4: a daily price snapshot for your keywords

```python
run = client.actor("mvkdata/walmart-scraper-fast").call(run_input={
    "searchTerms": ["air fryer", "robot vacuum"],
    "maxItemsPerSearch": 100,
})
for r in client.dataset(run.default_dataset_id).iterate_items():
    print(r["position"], r["price"], r["sellerName"], r["name"][:60])
```

Run it on a schedule (Apify has a built-in scheduler, or use cron) and diff the files to see price drops, new sellers
and products climbing the results. Set `"scrapeProductDetails": true` to open every result for the full record: UPC,
specs, variants with their own prices, stock and delivery dates.

## Step 5: a whole category, past the 10-page limit

```json
{
  "startUrls": [{ "url": "https://www.walmart.com/browse/electronics/tvs/3944_1060825_447913" }],
  "maxItemsPerSearch": 0
}
```

When a search or category has more matches than Walmart will page through, the scraper splits it into price ranges
and de-duplicates the results. In our test with `"maxItemsPerSearch": 1100` it returned all 1,100 TVs asked for, where
plain paging stops at about 400.

## JavaScript

```js
import { ApifyClient } from 'apify-client';   // npm i apify-client

const client = new ApifyClient({ token: process.env.APIFY_TOKEN });
const run = await client.actor('mvkdata/walmart-scraper-fast').call({
    startUrls: [{ url: 'https://www.walmart.com/ip/5113175776' }],
    includeReviews: true,
    maxReviewsPerProduct: 0,
}, { log: null });
const { items } = await client.dataset(run.defaultDatasetId).listItems();
console.log(items.filter((r) => r.type === 'review').length, 'reviews');
```

## What you get per row

- **Product:** name, brand, model, UPC, price, was-price, stock, seller (name, rating, fulfilled by Walmart or not),
  other offers, specs, description, images, variants, delivery and pickup, rating breakdown 1-5 stars and Walmart's
  AI review summary.
- **Search result:** position, price, seller, rating, review count, sponsored flag, fulfillment type.
- **Review:** rating, title, text, date, verified purchase, helpful votes, purchased variant, photos, seller responses.

All examples are in the [GitHub repo](https://github.com/mvkdata/walmart-scraper). Questions or a field you need?
Open an issue on the [actor page](https://apify.com/mvkdata/walmart-scraper-fast/issues).
