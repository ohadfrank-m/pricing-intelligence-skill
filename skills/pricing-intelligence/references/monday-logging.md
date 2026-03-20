# Monday.com logging workflow

After every pricing intelligence output, log the result to the user's chosen monday.com board. The target board is determined once at the start of the first logging call in a session — the user picks it, and it's reused for the rest of the session without asking again.

## Step 1: Determine the target board

This step runs **once per session** — on the first logging call only. Store the resolved board ID and reuse it for all subsequent items in the same session without asking again.

### 1a. Ask the user

Before searching or creating anything, ask:

> "Where should I log pricing outputs on monday.com?
> 1. **Use an existing board** — tell me the board name or paste its URL
> 2. **Create a new board** — I'll set one up and ask which workspace to put it in"

Wait for the user's response before proceeding.

---

### If the user chooses option 1 (existing board)

Use whatever name or URL they provide. If they gave a URL, extract the board ID directly from it (format: `monday.com/boards/{id}`). If they gave a name, search for it:

```
search(searchTerm="{user-provided name}", searchType="BOARD")
```

- Extract the board ID from the result (strip the `board-` prefix — the tool returns `board-123456`, pass `123456` as the numeric ID)
- If no match is found, tell the user and ask them to try again or choose option 2

Skip to Step 2.

---

### If the user chooses option 2 (create new board)

Ask for a board name:

> "What should I name the board? (default: 'Pricing Intelligence')"

If the user skips, use `Pricing Intelligence` as the name.

Then fetch available workspaces:

```
list_workspaces()
```

Present the workspace list to the user (name + ID) and ask:

> "Which workspace should I create the board in?"

Wait for the user's selection. Then create the board:

```
create_board(
  boardName="{chosen name}",
  boardKind="private",
  boardDescription="Tracks pricing changes, company research, and market landscape scans from the Pricing Intelligence skill.",
  workspaceId={selected workspace id}
)
```

Then create these 5 columns in parallel:

```
create_column(boardId={id}, columnType="status", columnTitle="Change Type")
create_column(boardId={id}, columnType="long_text", columnTitle="Summary")
create_column(boardId={id}, columnType="link", columnTitle="PricingSaaS")
create_column(boardId={id}, columnType="text", columnTitle="Workflow")
create_column(boardId={id}, columnType="link", columnTitle="Visual diff")
```

Note: the `date` column is not needed — monday automatically captures creation date.

---

## Step 2: Get board column IDs

Always call this before creating items, regardless of whether the board was just created or already existed. Column IDs are auto-generated and required for populating values.

```
get_board_info(boardId={id})
```

From the response, extract the column IDs for:
- `Change Type` (status column)
- `Summary` (long_text column)
- `PricingSaaS` (link column)
- `Workflow` (text column)
- `Visual diff` (link column)

Store these IDs for use in Step 3.

---

## Step 3: Create items

Create one item per company with changes. For landscape scans with no specific company, create one item per significant pattern identified.

### Item name format

| Workflow | Name format | Example |
|----------|-------------|---------|
| Company research | `{Company} — Research` | `Clay — Research` |
| Trend research | `{Category} — Landscape` | `Project Management — Landscape` |
| Monitoring (per company with changes) | `{Company} — {Period}` | `Figma — 2026W12` |

### Column values

For **monitoring items** (where visual diff images are available from visual-diff.md):

```
create_item(
  boardId={id},
  name="{item name}",
  columnValues="{\"<summary_col_id>\":\"{1-2 sentence summary}\",\"<pricingsaas_col_id>\":{\"url\":\"{pricingsaas url}\",\"text\":\"{Company} on PricingSaaS\"},\"<workflow_col_id>\":\"{workflow name | change type}\",\"<visual_diff_col_id>\":{\"url\":\"{compare-viewer url}\",\"text\":\"Visual diff — {period}\"}}"
)
```

For **all other workflows** (company research, landscape, etc.) — omit the `visual_diff_col_id` field:

```
create_item(
  boardId={id},
  name="{item name}",
  columnValues="{\"<summary_col_id>\":\"{1-2 sentence summary}\",\"<pricingsaas_col_id>\":{\"url\":\"{pricingsaas url}\",\"text\":\"{Company} on PricingSaaS\"},\"<workflow_col_id>\":\"{workflow name | change type}\"}"
)
```

### Visual diff column

The `Visual diff` link column points to the compare-viewer URL constructed in [visual-diff.md](visual-diff.md). It applies to **monitoring items only** — not company research or landscape items.

Compare-viewer URL format:
```
https://pricingsaas.com/compare-viewer?url1={encoded_cloudinary_before}&url2={encoded_cloudinary_after}&w=800&h=600&label1={YYYYMMDD_before}&label2={YYYYMMDD_after}&caption={URL_encoded_caption}
```

Use versionless Cloudinary URLs in the compare-viewer (no `v{version}` needed):
```
https://res.cloudinary.com/dd6dkaan9/image/upload/pricing_pages/{slug}_{YYYYMMDD}.png
```

If visual-diff.md returns no images (Cloudinary probe failed and no credits), populate the column with the PricingSaaS diff link as fallback:
```
{"url": "https://pricingsaas.com/pulse/companies/{slug}/diffs/{period}", "text": "View diff on PricingSaaS"}
```

### Update feed (image comments) — monitoring items only

After creating the board item, always post an update on it with the before/after images as HTML. This runs in parallel with doc creation.

**If both before and after snapshots exist:**
```html
<p><strong>📸 Visual diff — {Company} {period}</strong></p>
<p><strong>Before ({YYYYMMDD_before}):</strong></p>
<img src="https://res.cloudinary.com/dd6dkaan9/image/upload/pricing_pages/{slug}_{YYYYMMDD_before}.png" alt="{Company} pricing page — before {YYYYMMDD_before}" />
<p><strong>After ({YYYYMMDD_after}):</strong></p>
<img src="https://res.cloudinary.com/dd6dkaan9/image/upload/pricing_pages/{slug}_{YYYYMMDD_after}.png" alt="{Company} pricing page — after {YYYYMMDD_after}" />
<p><em>{1-sentence change description}</em></p>
<p><a href="{compare-viewer url}">Open interactive side-by-side comparison →</a></p>
```

**If only one snapshot exists (before or after):**
```html
<p><strong>📸 Visual diff — {Company} {period}</strong></p>
<img src="https://res.cloudinary.com/dd6dkaan9/image/upload/pricing_pages/{slug}_{YYYYMMDD}.png" alt="{Company} pricing page — {label}" />
<p><em>Only one snapshot available for this period.</em></p>
<p><a href="https://pricingsaas.com/pulse/companies/{slug}/diffs/{period}">View full diff on PricingSaaS →</a></p>
```

**If no snapshots exist:**
```html
<p><strong>📸 Visual diff — {Company} {period}</strong></p>
<p><em>Screenshots not available for this period — no Cloudinary snapshots found and credits at 0.</em></p>
<p><a href="https://pricingsaas.com/pulse/companies/{slug}/diffs/{period}">View before/after on PricingSaaS →</a></p>
```

Call `create_update(itemId={item_id}, body="{html}")` after `create_item` returns the item ID. Run in parallel with any doc creation step.

### PricingSaaS link format

| Workflow | Link |
|----------|------|
| Monitoring | `https://pricingsaas.com/pulse/companies/{slug}/diffs/{period}` |
| Company research (specific period pulled) | `https://pricingsaas.com/pulse/companies/{slug}/diffs/{period}` |
| Company research (no specific period) | `https://pricingsaas.com/pulse/companies/{slug}` |
| Landscape scan | leave blank (`""`) |

The `{period}` is always the period string passed to `get_company_history` or `get_diff_highlight` (e.g., `2026W12`, `2025Q1`). Strip any trailing dot from the slug in the URL (use `clay` not `clay.`).

### Summary field

1–2 sentences capturing the key insight. Examples:
- "Eliminated Starter/Explorer/Pro plans. Introduced Launch ($167/mo) and Growth (dual-metric: actions + data credits)."
- "Raised Professional Full seat 7% ($15→$16/mo), Organization 22% ($45→$55), Enterprise 20% ($75→$90). Introduced 3 seat types (Collab, Dev, Full)."
- "3 add-ons added this week. Follows Q1 2025 seat restructure pattern of modular monetization expansion."

---

## Step 4: Confirm to user

After all items are created, add one line at the end of your response:

```
Logged to [{board name}](https://monday.com/boards/{board_id}) on monday.
```

Do not make this into a separate section or elaborate on it — one line is enough.

---

## Step 5: Create the monday doc

For every item logged, always create a doc attached to the board item — no user prompt needed. Run `create_doc` calls in parallel with the activity update from Step 3.

**Before composing the doc markdown:** run the Cloudinary probe from [visual-diff.md](visual-diff.md) Steps 1a–1d for every item (monitoring, research, teardown, digest). This is mandatory and free. If the probe finds before/after snapshots, embed them in the doc. If it finds nothing, generate a Thum.io screenshot URL for the company's pricing page and embed it as the current-state image (see visual-diff.md Step 1e). Only omit images entirely if Thum.io also fails.

For each item logged, call `create_doc` to attach a doc directly to that board item:

```
create_doc(
  doc_name="{Company/Category} — {Workflow type} — {Month Year}",
  markdown="{structured content — see format below}",
  location="item",
  item_id={item_id returned by create_item in Step 3}
)
```

Doc name examples:
- "Clay — Monitoring — Mar 2026"
- "Project Management Tools — Landscape — Mar 2026"
- "Figma — Research — Mar 2026"

### Doc content format

Do NOT dump a wall of text. Mirror the PricingSaaS diff page structure exactly: a synthesised opening title, numbered change sections with inline images, then strategic context.

#### How to generate the opening title

After collecting all `get_diff_highlight` text matches, synthesise a single descriptive sentence that names the two or three most significant changes — same pattern PricingSaaS uses on their diff pages. No extra credit spend. Examples:

- "Notion launches Custom Agents add-on, opens guest seats to unlimited"
- "Clay eliminates Starter/Explorer/Pro, introduces dual-metric Launch and Growth plans"
- "HubSpot adds pipeline limits to Free and Starter, shifts metric from seats to pipelines"

#### How to get change text

Call `get_diff_highlight` once per individual change, running all calls in parallel (1 credit each). Each call returns a text match describing that specific change plus optionally a cropped `image_url`. Map queries from the change types returned by `get_company_history`:

```
# Example — Notion 2026W10 (3 changes), run in parallel:
get_diff_highlight(slug="notion.", period="2026W10", query="guest seats external guest limit unlimited")
get_diff_highlight(slug="notion.", period="2026W10", query="Custom Agents add-on launched AI credits")
get_diff_highlight(slug="notion.", period="2026W10", query="Notion Agent AI agent rename")
```

If credits = 0: use the change type labels from `discovery_only` as the section headings and show only the Cloudinary full-page images plus the diff link as a placeholder.

#### How to get images

**Per-change image (credits path):** `get_diff_highlight` returns `image_url` — a cropped, annotated screenshot of the changed area. Embed it directly in that change's section.

**Full-page before/after (free path):** Run [visual-diff.md](visual-diff.md) Cloudinary probe to get `cloudinary_before` and `cloudinary_after` full-page snapshots. Use these in every section when no per-change image is available, or in addition to per-change images.

#### Doc template

```markdown
# {Synthesised title — e.g. "Notion launches Custom Agents add-on, opens guest seats to unlimited"}

**Source:** [View on PricingSaaS](https://pricingsaas.com/pulse/companies/{slug}/diffs/{period})

---

## 1. {Change type} | {Category}
e.g. "## 1. Capacity Increased | Packaging"

{Full text match from get_diff_highlight, including bracketed detail}
e.g. "Guest seat limits changed to Unlimited for Plus, Business, and Enterprise plans. Row renamed from 'Guest seats' to 'External guest limit'. [Previously Plus=100, Business=250, Enterprise=Starting at 250; now all three show Unlimited guests]"

**Before ({YYYYMMDD_before}):**
![{Company} pricing page — before {YYYYMMDD_before}]({cloudinary_before OR get_diff_highlight image_url for before state})

**After ({YYYYMMDD_after}):**
![{Company} pricing page — after {YYYYMMDD_after}]({cloudinary_after OR get_diff_highlight image_url})

[Open side-by-side comparison →]({compare-viewer URL})

{If no images available: omit the image block entirely — no placeholder, no link, no credit message.}

---

## 2. {Change type} | {Category}

{Text match}

{Images — same pattern as above}

---

## {N}. {Change type} | {Category}

{Text match}

{Images}

---

## Strategic read

{2-3 sentences: what this set of changes signals about the company's positioning, growth motion, or monetisation direction}

## So what for monday.com

- {Competitive implication 1}
- {Competitive implication 2}
- {Competitive implication 3}
```

**Key rules:**
- H1 is always the synthesised title — never `# {Company} — {Period} — Pricing Changes`
- Section headings: `## {N}. {Change type} | {Category}` — numbered, pipe-separated category, no "Change" prefix
- Category comes from the PricingSaaS change event: always one of `Pricing`, `Packaging`, or `Product`
- Images go immediately after the text match, inside each numbered section — no separate `### Visual diff` subsection
- Embed images as `![label](url)` — Cloudinary URLs are public and render inline in monday Workdocs
- Always include the compare-viewer link even when images are present
- If both Cloudinary probe and `get_diff_highlight` are unavailable, omit the image block entirely — no placeholder or credit message
- "So what for monday.com" replaces "Context: pricing history" as the closing section

For monitoring sessions with multiple companies: one doc per company, each attached to its respective item. Run `create_doc` calls in parallel.

For landscape scans: one doc for the full scan with one section per company that had changes.

The doc is always created. Do not ask the user whether they want one.

---

## Error handling

- If `search` returns no board and `create_board` fails: skip logging silently, do not surface the error to the user
- If `create_item` fails for one company: continue with the others, note at the end: "Note: monday logging failed for {Company}."
- If `create_doc` fails: note it briefly but do not retry
- Never block the main pricing output waiting for logging to complete — deliver the pricing results first, log after

---

## Multiple companies in one session

For monitoring sessions where multiple companies have changes, create one item per company — not one item for the whole session. Run `create_item` calls in parallel.

Example: monitoring check returns changes for Clay and Figma → create "Clay — 2026W12" and "Figma — 2026W12" as two separate items, with links pointing to `.../clay/diffs/2026W12` and `.../figma/diffs/2026W12` respectively.
