# 🖼️ AutoPinEngine — Pinterest Automated Content Pipeline
### Full Project Documentation & Workflow Portfolio

---

## 📌 Project Title

**AutoPinEngine** — A fully automated, multi-account Pinterest pin generation system powered by Midjourney, Placid, OpenAI, and n8n. Designed to produce SEO-optimized Pinterest pins (images + titles + descriptions + board assignments) at scale, across multiple Discord/Midjourney accounts, using a dynamic prompt and template system.

---

## 📋 Project Description

AutoPinEngine is an end-to-end n8n automation pipeline that:

1. **Reads keywords** from a Google Sheets spreadsheet
2. **Generates Midjourney images** via a self-hosted API, across multiple accounts in parallel
3. **Upscales the best images** (U1–U4) using the same Midjourney API
4. **Combines pairs of upscaled images** with Placid templates to create Pinterest-ready pin graphics
5. **Generates SEO content** (title, description, tagline, board) for each pin using GPT-4.1 Mini
6. **Saves all outputs** back to Google Sheets and to per-account sheets for scheduling/publishing

The system is built to run multiple Discord accounts simultaneously, each with its own port, token, prompt style, and Placid template — making it capable of producing tens or hundreds of unique Pinterest pins per keyword with minimal manual intervention.

---

## 🗂️ System Architecture Overview

```
Google Sheets (Keywords)
        │
        ▼
  [Main Workflow]
  ┌─────────────────────────────────────────────────────┐
  │  1. Read keyword rows (state-filtered)              │
  │  2. Assign accounts, tokens, ports, prompts         │
  │  3. Send /imagine to Midjourney API per account     │
  │  4. Poll until done → save image hash + grid URL   │
  │  5. Upscale U1–U4 per account                      │
  │  6. Save all U URLs to Google Sheets               │
  │  7. Mark state → "u" (upscale done)                │
  └─────────────────────────────────────────────────────┘
        │
        ▼
  [Main Workflow continues — "u" state rows]
  ┌─────────────────────────────────────────────────────┐
  │  8. Read rows with state = "u"                     │
  │  9. Build image combos (U1+U2, U1+U3, etc.)       │
  │  10. Per combo → call Sub-Workflow (get="u")       │
  └─────────────────────────────────────────────────────┘
        │
        ▼
  [Main Workflow — "p" state rows]
  ┌─────────────────────────────────────────────────────┐
  │  11. Read rows with state = "p"                    │
  │  12. Per row → call Sub-Workflow (get="p")         │
  └─────────────────────────────────────────────────────┘
        │
        ▼
  [Sub-Workflow — Loop Api Link Strawberry Discourd Brave]
  ┌─────────────────────────────────────────────────────┐
  │  Switch on "get" parameter:                        │
  │  ├── get="v" → Midjourney /imagine + poll + save  │
  │  ├── get="u" → Midjourney /upscale ×4 + save      │
  │  └── get="p" → GPT content + Placid render + save │
  └─────────────────────────────────────────────────────┘
        │
        ▼
  Google Sheets (Results saved per-row, per-account sheet)
```

---

## 📁 Workflows

### Workflow 1 — Main Orchestrator

**File:** Main Workflow JSON  
**Purpose:** Master controller. Reads the spreadsheet, dispatches jobs to the sub-workflow, and manages pipeline state.

---

#### 🔧 Configuration Nodes

| Node | Type | Description |
|------|------|-------------|
| `config` | Set | Defines `nbrAccont` (max accounts to use, e.g. 3) and `accont selected` (array of active account IDs e.g. `[1,2,3,5,6]`) |
| `midourny promps` | Set | Defines Midjourney suffix prompts per account (accounts 1–10). All currently use: `"Different from its predecessor Focus on the details --iw 2 --ar 1:1 --v 7 --raw"` |
| `account tokens` | Set | Stores Discord/Midjourney API tokens per account (accounts 1–10) |
| `account port` | Set | Stores port numbers for the self-hosted Midjourney API per account (e.g. account 1 → port 5006, account 3 → port 5007) |

---

#### 📊 Google Sheets Data Source

**Spreadsheet ID:** `1-IV5--hOOorr8ndeREXnktOrVeKI21Ja0DARc-rmL-Y`  
**Sheet:** `Feuille 1` (gid=0)  
**Name:** `5 My Coffee Blog 03/08`

**Columns used:**

| Column | Purpose |
|--------|---------|
| `KEYWORDS` | The recipe/food keyword to generate pins for |
| `link` | Article link to embed in the pin |
| `state` | Pipeline state flag: empty → `u` → `p` → `d` |
| `account` | Comma-separated account IDs to use for this row (e.g. `1,3`) |
| `templite` | Comma-separated template combo indexes to use (1–6) |
| `v1`–`v10` | Midjourney job hash per account |
| `image generated Account N` | Full grid image URL per account |
| `U1 N`–`U4 N` | Upscaled image URLs (upscale 1–4) per account |

---

#### 🔄 Pipeline Stages in Main Workflow

**Stage 1 — Image Generation (state: empty/new)**

```
Get keyword → If (KEYWORDS not empty) → limits return (cap to nbrAccont)
→ Loop Over Items2 → Code1 (build vdata: promp/token/port per account)
→ Sub-Workflow (get="v") → line generation wait → Loop back
```

- Reads all rows where `state` is empty (no filter)
- Filters to `nbrAccont` rows maximum
- Loops through each row, builds a `vdata` array with each account's token, port, and Midjourney suffix prompt
- Calls the sub-workflow with `get="v"` to trigger Midjourney `/imagine` for each account
- After each sub-workflow completes, a `line generation` wait node (no timeout = resume immediately) feeds back into the loop

**Stage 2 — Upscaling (state: "u")**

```
get data for variation (filter state="u") → Loop Over Items (batchSize 1)
→ SplitInBatches → Wait2 (3s between accounts) → get data for variation
→ get all v id (build vdata with valid v-hashes) → Sub-Workflow (get="u")
→ between line upscale wait → Loop back
```

- Reads rows where `state = "u"` (Midjourney generation done, ready to upscale)
- For each row, collects all valid `v{n}` hashes (skipping accounts that already have a `U2 N` URL)
- Calls sub-workflow with `get="u"` to trigger upscaling (U1–U4) per account
- A 3-second wait between batch items prevents Discord rate-limiting

**Stage 3 — Combo Building + Placid Rendering (state: "p")**

```
Get row(s) in sheet (state="p") → limits return1 → get all possibility
→ Loop Over Items1 → splite images u (build vdata with combos + prompts + templates)
→ Sub-Workflow (get="p") → between line placeid (1s wait) → Loop back
```

- Reads rows where `state = "p"` (upscaling done, ready for Placid rendering)
- `get all possibility`: builds every valid image pair combination (U1+U2, U1+U3, U1+U4, U2+U3, etc.) for each template index
- `splite images u`: maps each combo to a random prompt variant (title/description/tagline/board - 4 variants per account), the Placid API key, and the correct template UUID
- Sub-workflow renders each combo into a Placid image and generates AI content

---

#### 🧩 Supporting Config Nodes

**`set place id`** — Stores Placid API credentials and template UUIDs per account:

```json
{
  "apikey": "placid-v1opm0th6cy06brc-fwfvvukommcttgpn",
  "template1": "ucrl74ybl0ov5",
  "template2": "be9qjo6esdk70",
  "template3": "czz6uxb9rk3tw",
  "template4": "twtjzcuh9hfk9",
  "template5": "tlilcybvxx2ku",
  "template6": "pd0yedlfv1ofr"
}
```
(Currently identical across all 10 accounts — can be per-account)

**`promps`** — 10 different Pinterest copywriting prompt sets (accounts 1–10). Each set contains 4 variants of:
- `title 1`–`title 4` — Hook-based Pinterest title formulas (under 90–100 chars, starting with keyword)
- `description 1`–`description 4` — SEO-optimized descriptions (max 450–500 chars)
- `tagline 1`–`tagline 4` — Short punchy taglines (max 3 words)
- `board 1`–`board 4` — Board classifier (from a fixed list of 20 Pinterest food boards)
- `midjourny` — Midjourney prompt suffix

---

### Workflow 2 — Sub-Workflow (Loop Api Link Strawberry Discourd Brave)

**File:** Sub-Workflow JSON  
**Purpose:** Executes one of three actions depending on the `get` parameter received from the main workflow.

---

#### Entry Point

```
When Executed by Another Workflow
  → urls (set base URL: http://82.197.93.19)
  → Switch (route based on get: "v" / "u" / "p")
```

**Input parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `get` | string | Action to perform: `"v"`, `"u"`, or `"p"` |
| `vdata` | array | Array of account objects (token, port, index, prompt/images) |
| `keyword` | string | The recipe keyword |
| `row_number` | number | Google Sheets row number for updating |
| `nbr` | array | Selected account IDs |

---

#### Branch A — `get="v"` (Midjourney Image Generation)

```
splite v tax → code (set index) → get first variation1 (POST /imagine)
→ check done (GET /status) → Wait (poll loop) → If1 (done?)
  ├── YES → data for img1 → Update row in sheet5 (save image URL)
  │         → data for vr1 → Update v id1 (save hash)
  │         → between photo (1s wait) → make picture done
  │         → Update row in sheet4 (set state="u")
  └── NO  → Wait → check done (loop back)
```

**Process:**
1. Each item in `vdata` is split into individual items
2. A POST to `/imagine` endpoint sends: `keyword + midjourney_suffix` as the prompt, with a webhook URL for result notification
3. The workflow polls `/status?hash=X` every ~1 second until `status === "done"`
4. On completion, saves the grid image URL to `image generated Account {N}` and the job hash to `v{N}` in Google Sheets
5. Sets `state = "u"` to signal the main workflow that generation is complete

**API Calls:**
```
POST http://82.197.93.19:{port}/imagine
  Headers: X-API-Token: {token}, Content-Type: application/json
  Body: { "prompt": "{keyword} {promp}", "webhook_url": "...", "webhook_type": "result" }

GET http://82.197.93.19:{port}/status?hash={hash}
  Headers: X-API-Token: {token}
  Response: { "status": "done"|"pending", "result": { "imageUrl": "...", "hash": "..." } }
```

---

#### Branch B — `get="u"` (Midjourney Upscaling U1–U4)

```
splite u tax → codev (set imagegeneratefrom, index) → U1 (POST /upscale #2)
→ Wait2 (1s) → check done2 → If (done?)
  ├── YES → for u image → Update row in sheet (save U2 N URL)
  │         → HTTP Request4 (POST /upscale #1)
  │         → Wait3 (1s) → check done3 → If3
  │           ├── YES → for u image1 → Update row in sheet6 (save U1 N URL)
  │           │         → HTTP Request6 (POST /upscale #3)
  │           │         → Wait5 → check done5 → If5
  │           │           ├── YES → for u image3 → Update row in sheet8 (save U3 N URL)
  │           │           │         → HTTP Request5 (POST /upscale #4)
  │           │           │         → Wait4 → check done4 → If4
  │           │           │           ├── YES → for u image2 → Update row in sheet7 (save U4 N URL)
  │           │           │           │         → brtween account (wait) → for state → Update row in sheet2 (state="p")
  │           │           │           └── NO  → Wait4 (loop)
  │           │           └── NO  → Wait5 (loop)
  │           └── NO  → Wait3 (loop)
  └── NO  → Wait2 (loop)
```

**Process:**
1. Triggers upscales for all 4 variations (U1, U2, U3, U4) of the Midjourney grid sequentially
2. Each upscale polls until done, then saves the resulting URL to `U{N} {account_index}` in Google Sheets
3. After all 4 are saved, sets `state = "p"` to proceed to Placid rendering

**API Calls:**
```
POST http://82.197.93.19:{port}/upscale
  Body: { "jobId": "{v_hash}", "upscale_number": 1|2|3|4 }
```

**Column mapping:**

| Upscale | Google Sheets column |
|---------|---------------------|
| U1 | `U1 {account}` |
| U2 | `U2 {account}` |
| U3 | `U3 {account}` |
| U4 | `U4 {account}` |

---

#### Branch C — `get="p"` (Placid Rendering + AI Content Generation)

```
splite u tax2 → Loop Over Items2 (batch through combos)
  ├── for state1 → Update row in sheet3 (set state="d" = done)
  └── generate title (GPT-4.1 Mini) → sae title
      → generate description → description
      → tagline → tag line
      → generate board (structured output) → Edit Fields
      → get templite1 (POST Placid API)
      → place id photo (wait 2s) → HTTP Request (GET Placid image status)
      → Switch1:
          ├── image_url exists → set data for output → Append row in sheet (account sheet)
          ├── error → set data for output (skip)
          └── not ready → place id photo (retry loop)
      → Loop Over Items2 (next combo)
```

**Process:**
1. Each combo in `vdata` (containing U1, U2, keyword, title prompt, description prompt, tagline prompt, board prompt, accountapi, template UUID) is processed one by one
2. GPT-4.1 Mini generates:
   - **Title**: A Pinterest hook title under 90–100 characters
   - **Description**: SEO-optimized description up to 480–500 characters
   - **Tagline**: A 3-word max punchy phrase
   - **Board**: One of 20 predefined Pinterest food boards
3. Placid API is called to render the template with U1, U2 images, the keyword as title, and the AI tagline
4. The workflow polls Placid until the rendered image URL is ready
5. Outputs are appended to the per-account sheet: `account {N}` (columns: Placid url, Board, Title, Description)
6. The main row state is set to `"d"` (done)

**Placid API Call:**
```json
POST https://api.placid.app/api/rest/images
Authorization: Bearer {apikey}
{
  "template_uuid": "{template_id}",
  "layers": {
    "U1": { "image": "{image1_url}" },
    "U2": { "image": "{image2_url}" },
    "Title": { "text": "{keyword}" },
    "Tag Line": { "text": "{ai_tagline}" }
  }
}
```

**Per-account output sheet columns:**

| Column | Source |
|--------|--------|
| `Placid url` | Rendered Placid image URL |
| `Board` | GPT-assigned Pinterest board |
| `Title` | GPT-generated pin title |
| `Description` | GPT-generated pin description |
| `keyword` | Original keyword |
| `link` | Article link |

---

## 🤖 Account System

The pipeline supports up to 10 accounts, each independently configurable:

| Setting | Per-Account Storage |
|---------|-------------------|
| **Midjourney Token** | `account tokens` Set node → `account {N}` |
| **API Port** | `account port` Set node → `account {N}` |
| **Midjourney Suffix Prompt** | `midourny promps` Set node → `account {N}` |
| **Pinterest Copywriting Prompts** | `promps` Set node → `promp accont {N}` |
| **Placid API Key + Templates** | `set place id` Set node → `account {N}` |

**Active accounts are selected via:**
- `config.nbrAccont` — limits how many rows are processed at once
- `config["accont selected"]` — array of which account IDs are active (e.g. `[1,2,3,5,6]`)
- Per-row `account` column — which accounts are assigned to this specific keyword row

---

## 🎨 Prompt Variants System

Each account (1–10) has a unique **copywriting personality** with 4 variants of title/description/tagline/board prompts. A random number 1–4 is selected per run, ensuring variety.

**Account Personalities:**
| Account | Style |
|---------|-------|
| 1 | Hook-phrase system (For Moms, Under $20, Lazy, Aesthetic, etc.) |
| 2 | Benefit-first, curiosity-driven, sensory language |
| 3 | Comfort memories, impress guests, foolproof, healthy |
| 4 | Game-changer, nostalgic, expert secret, trending |
| 5 | Same as account 3 (comfort/easy/healthy) |
| 6 | Seasonal/holiday, chef's secret, 15-minute meal, budget family |
| 7 | Gourmet/luxurious, nostalgic, dinner party, health-conscious |
| 8 | Viral/trending, shocking combos, cooking challenge, FOMO urgency |
| 9 | Beginner-friendly, budget under $10, one-pan, 5-ingredient |
| 10 | Crowd-pleaser, better-than-takeout, cozy winter, grandma's secret |

**Fixed Pinterest Board List (20 boards):**
Dinner Recipes, Chicken Recipes, Pasta Recipes, Breakfast Recipes, Healthy Recipes, Soup Recipes, Salad Recipes, Shrimp Recipes, Dessert Recipes, Cake Recipes, Bread Recipes, Appetizer Recipes, Beef Recipes, Seafood Recipes, Slow Cooker Recipes, Air Fryer Recipes, Instant Pot Recipes, Side Dishes, Easy Recipes, Casserole Recipes

---

## 🔁 State Machine

Each Google Sheets row moves through the following states:

```
[empty] → image generation (Midjourney /imagine per account)
    ↓
  "u"   → upscaling (Midjourney /upscale U1–U4 per account)
    ↓
  "p"   → Placid rendering + AI content generation
    ↓
  "d"   → done (all pins created and saved to account sheets)
```

---

## 🧱 Key Technical Components

### Self-Hosted Midjourney API
- Base URL: `http://82.197.93.19`
- One port per account (e.g. 5006, 5007, ...)
- Endpoints:
  - `POST :{port}/imagine` — submit generation job
  - `GET :{port}/status?hash={hash}` — poll job status
  - `POST :{port}/upscale` — upscale one of 4 grid images
- Authentication: `X-API-Token` header

### Placid (Template Rendering)
- REST API: `https://api.placid.app/api/rest/images`
- 6 template UUIDs per account (template1–template6)
- Template layers: `U1` (image), `U2` (image), `Title` (text), `Tag Line` (text)
- Bearer auth

### OpenAI (Content Generation)
- Model: `gpt-4.1-mini`
- Sequential chain: title → description → tagline → board
- Board uses `Structured Output Parser` for reliable JSON output

### Google Sheets (State + Storage)
- Main sheet (`Feuille 1`): keyword rows, state tracking, all image URLs
- Per-account sheets (`account 1`, `account 2`, ...): final pin data for publishing

---

## 🚀 How to Run

1. **Prepare the spreadsheet:**
   - Add keyword rows to the main sheet
   - Fill in `KEYWORDS`, `link`, `account` (e.g. `1,3`), `templite` (e.g. `1,2,3`)
   - Leave `state` empty

2. **Configure accounts:**
   - Update `account tokens`, `account port`, `midourny promps` Set nodes with real values
   - Update `set place id` with your Placid API keys and template UUIDs
   - Set `config.nbrAccont` to control parallelism
   - Set `config["accont selected"]` to your active account IDs

3. **Run the main workflow:**
   - Click "Execute workflow" (manual trigger)
   - The pipeline will process rows through all 3 stages automatically

4. **Monitor progress:**
   - Watch the `state` column in Google Sheets advance from empty → `u` → `p` → `d`
   - Final pin data appears in the per-account sheets

---

## ⚠️ Important Notes & Limitations

- **Rate limiting:** The workflow uses Wait nodes (1–3 seconds) between accounts to avoid Discord/Midjourney bans
- **Sequential upscaling:** U1–U4 are done sequentially per account (not parallel) due to Midjourney API constraints
- **Retry logic:** HTTP Request nodes for Midjourney have `retryOnFail: true` with 5-second delays
- **Placid retry:** If a Placid image isn't ready, the workflow loops back to check again (Switch1 → output 3)
- **Error handling:** Most Google Sheets update nodes and Placid calls use `onError: continueRegularOutput` to avoid stopping the whole pipeline on one failure
- **Account 2 token** is currently set to `"2"` (placeholder — needs real token)
- **Accounts 4, 5** ports are currently `"4"`, `"5"` (placeholders — need real port numbers)
- **All Placid templates** currently share one API key — differentiate per account if needed

---

## 📦 Dependencies & Services

| Service | Role |
|---------|------|
| **n8n** | Workflow automation platform |
| **Google Sheets** | Data source, state store, output destination |
| **Self-hosted Midjourney API** | Image generation via Discord |
| **Placid** | Template-based image rendering |
| **OpenAI GPT-4.1 Mini** | SEO title, description, tagline, board generation |

---

## 🗺️ File Reference

| Item | ID / Value |
|------|-----------|
| Main spreadsheet | `1-IV5--hOOorr8ndeREXnktOrVeKI21Ja0DARc-rmL-Y` |
| Sub-workflow ID | `W1PpkWNTEJutazoY` |
| Midjourney base URL | `http://82.197.93.19` |
| Placid API endpoint | `https://api.placid.app/api/rest/images` |
| OpenAI model | `gpt-4.1-mini` |
| n8n webhook (imagine result) | `.../webhook/8954db7c-bf8b-417a-85c3-0c0d08bc95f5` |
| Google Sheets credential | `Google Sheets account 3` (ID: `KtJ0AOOeD4DpObI5`) |
| OpenAI credential | `OpenAi account` (ID: `AOGEFNkcTLTrdEo7`) |
| Placid credential | `Bearer Auth account` (ID: `UqeyCyVhC8SGBLb9`) |

---

*Generated documentation for AutoPinEngine — a fully automated Pinterest content factory.*
