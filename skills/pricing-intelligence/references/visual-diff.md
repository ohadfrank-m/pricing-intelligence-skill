# Visual diff workflow

Capture before/after screenshots of a pricing page change and embed them in the monday doc. Call this from any workflow after a high-signal period is identified.

**High-signal change types that trigger this workflow:** `Price Increased`, `Price Decreased`, `Plan Added`, `Plan Removed`, `Plan Renamed`, `Discount Removed`, `Discount Added`.

---

## How it works

PricingSaaS diff pages are client-side rendered — WebFetch returns 404, and the browser MCP is blocked for pricingsaas.com. Three paths to get the images, in order of preference:

| Path | Cost | How |
|------|------|-----|
| Cloudinary direct probe | Free | Guess snapshot dates → curl probe → versionless URLs work |
| `get_diff_highlight` | 1 credit | Returns `image_url` directly |
| Fallback link only | Free | Embed PricingSaaS diff page link; no visual |

**Key finding:** Cloudinary serves PricingSaaS pricing page images **without the `v{version}` number**. The URL `https://res.cloudinary.com/dd6dkaan9/image/upload/pricing_pages/{slug}_{YYYYMMDD}.png` works directly if the snapshot exists for that date. This makes it possible to get images for free by probing likely dates.

Cloudinary image URLs are publicly accessible and can be embedded directly in monday docs as markdown images.

---

## Step 1: Get image URLs — Cloudinary direct probe (free, always try first)

The Cloudinary image URL pattern is:
```
https://res.cloudinary.com/dd6dkaan9/image/upload/pricing_pages/{slug}_{YYYYMMDD}.png
```

The `v{version}` timestamp is NOT needed — Cloudinary serves the image by public_id alone. This means you can probe for snapshot files by date without any credits or browser access.

### 1a: Identify candidate dates to probe

From the `discovery_only` history, map the high-signal period to target dates:

| Period | Before dates to try | After dates to try |
|--------|--------------------|--------------------|
| `{YYYY}Q1` | Dec 1, Dec 15 of prior year | Mar 1, Mar 15, Apr 1 |
| `{YYYY}Q2` | Mar 1, Mar 15 | Jun 1, Jun 15, Jul 1 |
| `{YYYY}Q3` | Jun 1, Jun 15 | Sep 1, Sep 15, Oct 1 |
| `{YYYY}Q4` | Sep 1, Sep 15, Oct 1 | Dec 1, Dec 8, Dec 15, Jan 1 |
| `{YYYY}M{MM}` | First of prior month | First of next month, 8th, 15th |
| `{YYYY}W{WW}` | 7 days before week end | Week end date + 1, + 7 |

Also always probe: the date in `discovery_only`'s `tracking_since` and `latest_version` fields.

### 1b: Probe for available snapshots

Run a batch of curl probes (all in parallel, using Shell):

```bash
for d in {date1} {date2} {date3} ...; do
  code=$(curl -s -o /dev/null -w "%{http_code}" \
    "https://res.cloudinary.com/dd6dkaan9/image/upload/pricing_pages/{slug}_${d}.png")
  echo "${d}: ${code}"
done
```

A `200` response means the snapshot exists and is usable. A `404` means no snapshot for that date.

**Narrow the range:** If many 404s, try intermediate dates (8th, 15th, last day of each month) until you find files. PricingSaaS typically captures 2–4 snapshots per month.

### 1c: Identify before and after URLs

From the probe results, select:
- `cloudinary_before` = the latest `200` date that falls **before** the change period started
- `cloudinary_after` = the earliest `200` date that falls **within or after** the change period

If only "after" snapshots exist (PricingSaaS started tracking after the change), note this clearly and use the earliest available as the baseline.

The final versionless URLs are:
```
https://res.cloudinary.com/dd6dkaan9/image/upload/pricing_pages/{slug}_{YYYYMMDD_before}.png
https://res.cloudinary.com/dd6dkaan9/image/upload/pricing_pages/{slug}_{YYYYMMDD_after}.png
```

### 1d: Construct the compare-viewer link

```
https://pricingsaas.com/compare-viewer?url1={encoded_cloudinary_before}&url2={encoded_cloudinary_after}&w=800&h=600&label1={YYYYMMDD_before}&label2={YYYYMMDD_after}&caption={URL_encoded_caption}
```

The compare-viewer renders the two images side by side in the user's browser — it works even with versionless Cloudinary URLs.

If no probes return `200`, fall through to Step 2.

---

## Step 2: Get image URLs — `get_diff_highlight` path (paid, fallback if Cloudinary probe returns no results)

Cost: 1 credit per call. Confirm before running if credits are low.

```
get_diff_highlight(slug="{slug}", period="{period}", query="{change description}")
```

From the response:
- `image_url` = the Cloudinary URL for the **after** state of the highlighted change
- Use this as `cloudinary_after`. There is no `cloudinary_before` from this call alone.

---

## Step 3: Embed in the monday doc

Once you have at least one Cloudinary URL (both from browser path; after-only from `get_diff_highlight`), pass this to `add_content_to_doc`:

```markdown
### Visual diff — {period}

**Before ({label_before}):**
![Pricing page — before {label_before}]({cloudinary_before})

**After ({label_after}):**
![Pricing page — after {label_after}]({cloudinary_after})

> {caption}

[Open interactive side-by-side comparison →]({full_compare-viewer_url})
```

If only the after image is available (from `get_diff_highlight`):

```markdown
### Visual diff — {period}

![Pricing page — after change ({label_after})]({cloudinary_after})

[View full diff on PricingSaaS →](https://pricingsaas.com/pulse/companies/{slug}/diffs/{period})
```

**Note on rendering:** Monday Workdocs render external images inline when the URL is publicly accessible. Cloudinary URLs are public — images should appear directly in the doc. If the image renders as a link instead of an inline image, the compare-viewer link still provides the visual.

---

## Step 4: If both paths fail

If the Cloudinary probe returns no `200` results and credits are 0, omit the visual section silently. Do not write any placeholder, credit message, or note about unavailability — the doc simply has no image block for that change.

The Cloudinary probe (Step 1) is the primary image path and is entirely free. Only reach Step 4 after Step 1 has been fully attempted across all candidate dates with no results.

---

## When to call this workflow

| Caller | When |
|--------|------|
| `company-research.md` Step 3 | After any high-signal period is identified (paid or credit-zero path) |
| `monitoring.md` | After `fetch_diffs` or `get_diff_highlight` returns results for a watchlist company |
| `weekly-digest.md` | For the top 1–2 highest-signal changes in the digest period |
| `battlecard-generator.md` | When the battlecard includes a competitor's recent price increase |

Run this workflow **after** the main pricing analysis is complete — not during. It adds the visual layer on top of the text analysis, and should not delay delivering the core findings.
