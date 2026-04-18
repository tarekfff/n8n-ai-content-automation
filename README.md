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

The system supports up to 10 Discord/Midjourney accounts simultaneously, each with its own port, token, prompt style, and Placid template — making it capable of producing tens or hundreds of unique Pinterest pins per keyword with minimal manual intervention.

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

**File:** `main_workflow.json`
**Purpose:** Master controller. Reads the spreadsheet, dispatches jobs to the sub-workflow, and manages pipeline state.

---

#### 🔧 Configuration Nodes

| Node | Type | Description |
|------|------|-------------|
| `config` | Set | Defines `nbrAccont` (max accounts to use) and `accont selected` (array of active account IDs e.g. `[1,2,3,5,6]`) |
| `midourny promps` | Set | Defines Midjourney suffix prompts per account (accounts 1–10) |
| `account tokens` | Set | Stores Discord/Midjourney API tokens per account — **replace with your own tokens** |
| `account port` | Set | Stores port numbers for the self-hosted Midjourney API per account |

> ⚠️ **Never hardcode tokens or ports in the workflow JSON when pushing to GitHub.** Use n8n credentials or environment variables. See [Setup](#️-setup--configuration) below.

---

#### 📊 Google Sheets Data Source

**Spreadsheet:** Your own Google Sheet (configure the spreadsheet ID in each Google Sheets node)
**Sheet:** Main sheet (e.g. `Sheet1`)

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

- Reads all rows where `state` is empty
- Filters to `nbrAccont` rows maximum
- Loops through each row, builds a `vdata` array with each account's token, port, and Midjourney suffix prompt
- Calls the sub-workflow with `get="v"` to trigger Midjourney `/imagine` for each account
- After each sub-workflow completes, a `line generation` wait node feeds back into the loop

**Stage 2 — Upscaling (state: "u")**

```
get data for variation (filter state="u") → Loop Over Items (batchSize 1)
→ SplitInBatches → Wait2 (3s between accounts)
→ get all v id (build vdata with valid v-hashes) → Sub-Workflow (get="u")
→ between line upscale wait → Loop back
```

- Reads rows where `state = "u"` (Midjourney generation done, ready to upscale)
- For each row, collects all valid `v{n}` hashes, skipping accounts that already have a `U2 N` URL
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
- `splite images u`: maps each combo to a random prompt variant (4 variants per account), the Placid API key, and the correct template UUID
- Sub-workflow renders each combo into a Placid image and generates AI content

---

#### 🧩 Supporting Config Nodes

**`set place id`** — Stores Placid API credentials and template UUIDs per account.
Replace placeholder values with your own:

```json
{
  "apikey": "YOUR_PLACID_API_KEY",
  "template1": "YOUR_TEMPLATE_1_UUID",
  "template2": "YOUR_TEMPLATE_2_UUID",
  "template3": "YOUR_TEMPLATE_3_UUID",
  "template4": "YOUR_TEMPLATE_4_UUID",
  "template5": "YOUR_TEMPLATE_5_UUID",
  "template6": "YOUR_TEMPLATE_6_UUID"
}
```

**`promps`** — 10 different Pinterest copywriting prompt sets (accounts 1–10). Each set contains 4 variants of:
- `title 1`–`title 4` — Hook-based Pinterest title formulas (under 90–100 chars, starting with keyword)
- `description 1`–`description 4` — SEO-optimized descriptions (max 450–500 chars)
- `tagline 1`–`tagline 4` — Short punchy taglines (max 3 words)
- `board 1`–`board 4` — Board classifier (from a fixed list of 20 Pinterest food boards)
- `midjourny` — Midjourney prompt suffix

---

### Workflow 2 — Sub-Workflow

**File:** `sub_workflow.json`
**Purpose:** Executes one of three actions depending on the `get` parameter received from the main workflow.

---

#### Entry Point

```
When Executed by Another Workflow
  → urls (set base URL: YOUR_MIDJOURNEY_SERVER_URL)
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
1. Each item in `vdata` is split into individual account items
2. A POST to `/imagine` sends: `keyword + midjourney_suffix` as the prompt, with a webhook URL for the result callback
3. The workflow polls `/status?hash=X` until `status === "done"`
4. On completion, saves the grid image URL to `image generated Account {N}` and the job hash to `v{N}` in Google Sheets
5. Sets `state = "u"` to signal the main workflow that generation is complete

**API shape:**
```
POST http://YOUR_SERVER_IP:{port}/imagine
  Headers: X-API-Token: {token}, Content-Type: application/json
  Body: { "prompt": "{keyword} {suffix}", "webhook_url": "{YOUR_N8N_WEBHOOK}", "webhook_type": "result" }

GET http://YOUR_SERVER_IP:{port}/status?hash={hash}
  Headers: X-API-Token: {token}
  Response: { "status": "done"|"pending", "result": { "imageUrl": "...", "hash": "..." } }
```

---

#### Branch B — `get="u"` (Midjourney Upscaling U1–U4)

```
splite u tax → codev (set imagegeneratefrom, index) → POST /upscale #2
→ Wait (1s) → check done2 → If (done?)
  ├── YES → save U2 → POST /upscale #1
  │         → Wait → check done3 → If3
  │           ├── YES → save U1 → POST /upscale #3
  │           │         → Wait → check done5 → If5
  │           │           ├── YES → save U3 → POST /upscale #4
  │           │           │         → Wait → check done4 → If4
  │           │           │           ├── YES → save U4 → set state="p"
  │           │           │           └── NO  → loop
  │           │           └── NO  → loop
  │           └── NO  → loop
  └── NO  → loop
```

**Process:**
1. Triggers upscales for all 4 variations (U1–U4) of the Midjourney grid sequentially
2. Each upscale polls until done, then saves the URL to `U{N} {account_index}` in Google Sheets
3. After all 4 are saved, sets `state = "p"`

**API shape:**
```
POST http://YOUR_SERVER_IP:{port}/upscale
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
  ├── for state1 → Update row in sheet3 (set state="d")
  └── generate title (GPT-4.1 Mini) → sae title
      → generate description → description
      → tagline → tag line
      → generate board (structured output) → Edit Fields
      → get templite1 (POST Placid API)
      → place id photo (wait 2s) → HTTP Request (GET Placid status)
      → Switch1:
          ├── image_url exists → set data for output → Append row in account sheet
          ├── error → skip
          └── not ready → retry loop
      → Loop Over Items2 (next combo)
```

**Process:**
1. Each combo in `vdata` (U1 URL, U2 URL, keyword, AI prompts, Placid API key, template UUID) is processed one by one
2. GPT-4.1 Mini generates sequentially:
   - **Title** — Pinterest hook title under 90–100 characters
   - **Description** — SEO description up to 480–500 characters
   - **Tagline** — 3-word max punchy phrase
   - **Board** — one of 20 predefined Pinterest food boards (via Structured Output Parser)
3. Placid API renders the template using U1, U2, the keyword, and the AI tagline
4. The workflow polls Placid until the image URL is ready
5. Outputs are appended to the per-account sheet

**Placid API shape:**
```json
POST https://api.placid.app/api/rest/images
Authorization: Bearer YOUR_PLACID_API_KEY
{
  "template_uuid": "YOUR_TEMPLATE_UUID",
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

| Setting | Where to Configure |
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
- Base URL: configure in the `urls` Set node
- One port per account (configure in `account port` Set node)
- Endpoints:
  - `POST :{port}/imagine` — submit a generation job
  - `GET :{port}/status?hash={hash}` — poll job status
  - `POST :{port}/upscale` — upscale one of 4 grid images
- Authentication: `X-API-Token` header

### Placid (Template Rendering)
- REST API: `https://api.placid.app/api/rest/images`
- 6 template UUIDs per account (template1–template6) — configure in `set place id` node
- Template layers: `U1` (image), `U2` (image), `Title` (text), `Tag Line` (text)
- Bearer auth via n8n credential

### OpenAI (Content Generation)
- Model: `gpt-4.1-mini`
- Sequential chain: title → description → tagline → board
- Board uses `Structured Output Parser` for reliable JSON output
- Configure via n8n OpenAI credential

### Google Sheets (State + Storage)
- Main sheet: keyword rows, state tracking, all image URLs
- Per-account sheets (`account 1`, `account 2`, ...): final pin data for publishing

---

## ⚙️ Setup & Configuration

### 1. n8n Credentials to Create

| Credential | Type | Used For |
|-----------|------|---------|
| Google Sheets OAuth2 | `googleSheetsOAuth2Api` | Reading/writing spreadsheet data |
| OpenAI API Key | `openAiApi` | GPT-4.1 Mini content generation |
| Placid Bearer Token | `httpBearerAuth` | Placid image rendering API |

### 2. Placeholders to Replace in the Workflow JSONs

| Placeholder | Replace With |
|-------------|-------------|
| `YOUR_SPREADSHEET_ID` | Your Google Sheets spreadsheet ID (from the URL) |
| `YOUR_MIDJOURNEY_SERVER_IP` | IP address of your self-hosted Midjourney API server |
| `YOUR_ACCOUNT_N_TOKEN` | Discord/Midjourney token for account N |
| `YOUR_ACCOUNT_N_PORT` | API port for account N (e.g. 5006, 5007, …) |
| `YOUR_PLACID_API_KEY` | Your Placid API key |
| `YOUR_TEMPLATE_N_UUID` | Placid template UUID for template slot N |
| `YOUR_N8N_WEBHOOK_URL` | Your n8n webhook URL for Midjourney result callbacks |
| `YOUR_SUB_WORKFLOW_ID` | n8n ID of the sub-workflow after importing it |

### 3. Google Sheet Structure

**Main sheet columns:**
```
KEYWORDS | link | state | account | templite | v1…v10 | image generated Account 1…10 | U1 1…U4 10
```

**Per-account output sheets** (one per account, named `account 1`, `account 2`, etc.):
```
Placid url | Board | Title | Description | keyword | link
```

### 4. Workflow Import Order

1. Import `sub_workflow.json` first — note its workflow ID in n8n
2. Replace `YOUR_SUB_WORKFLOW_ID` in `main_workflow.json` with the ID from step 1
3. Import `main_workflow.json`
4. Attach all credentials in both workflows

---

## 🚀 How to Run

1. **Prepare the spreadsheet** — add rows with `KEYWORDS`, `link`, `account`, and `templite` filled in; leave `state` empty
2. **Configure the Set nodes** — update tokens, ports, Placid API keys, template UUIDs
3. **Execute the main workflow** — click "Execute workflow" (manual trigger)
4. **Monitor via Google Sheets** — watch `state` advance: empty → `u` → `p` → `d`; final pins appear in per-account sheets

---

## ⚠️ Important Notes

- **Rate limiting:** Wait nodes (1–3 seconds) between accounts prevent Discord/Midjourney bans
- **Sequential upscaling:** U1–U4 are processed one at a time per account due to API constraints
- **Retry logic:** Midjourney HTTP requests use `retryOnFail: true` with 5-second delays
- **Placid retry:** If a rendered image isn't ready yet, the workflow loops back automatically
- **Error handling:** Most nodes use `onError: continueRegularOutput` so one failure doesn't halt the whole pipeline

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

## 🔒 Security Notes

- **Never commit real tokens, API keys, server IPs, or spreadsheet IDs** to version control
- Store all secrets via n8n credentials or environment variables
- The workflow JSON files in this repo should contain only placeholder values before committing
- Consider restricting your self-hosted Midjourney API server to accept requests only from your n8n server IP

---

*AutoPinEngine — a fully automated Pinterest content factory built on n8n.*
