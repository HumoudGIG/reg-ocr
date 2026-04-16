# REG·OCR — Vehicle Registration Card OCR Tool
Internal underwriting tool for extracting fields from Kuwait vehicle registration cards.

---

## What this does

- Upload a photo of a registration card (JPG, PNG, WEBP)
- Claude Vision (Haiku) reads the card and extracts all fields
- Every field gets a confidence rating: **HIGH** / **MED** / **LOW**
- Medium and low confidence fields require manual confirmation before you can copy/export
- Exports to clipboard or CSV

**Cost:** ~$0.001 per scan (Claude Haiku pricing)

---

## Setup — 3 steps

### Step 1: Get an Anthropic API key
1. Go to [console.anthropic.com](https://console.anthropic.com)
2. Create an API key
3. Keep it somewhere safe

### Step 2: Deploy to GitHub Pages (free hosting)

```bash
# In Cursor terminal:
git init
git add .
git commit -m "feat: initial OCR tool"

# Create a repo on github.com, then:
git remote add origin https://github.com/YOUR_ORG/reg-ocr.git
git push -u origin main
```

Then on GitHub:
- Go to **Settings → Pages**
- Source: **Deploy from branch → main → / (root)**
- Your tool is live at: `https://YOUR_ORG.github.io/reg-ocr`

### Step 3: Share with the team
Give them the GitHub Pages URL. On first visit, the config modal will appear asking for the API key. Each user enters their own key — it stays in their browser's localStorage.

---

## Production upgrade: Cloudflare Worker (recommended)

Currently the API key is stored in each user's browser. For a more secure setup where one shared key is used server-side and never exposed to the browser, set up a Cloudflare Worker proxy in 15 minutes:

### 1. Create a Cloudflare Worker

```javascript
// worker.js — paste this into Cloudflare Workers dashboard
export default {
  async fetch(request, env) {
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': 'https://YOUR_ORG.github.io',
          'Access-Control-Allow-Methods': 'POST',
          'Access-Control-Allow-Headers': 'Content-Type',
        }
      });
    }

    if (request.method !== 'POST') return new Response('Method not allowed', { status: 405 });

    const body = await request.json();

    const response = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'x-api-key': env.ANTHROPIC_API_KEY,   // set in Worker env vars
        'anthropic-version': '2023-06-01',
      },
      body: JSON.stringify(body)
    });

    const data = await response.json();
    return new Response(JSON.stringify(data), {
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': 'https://YOUR_ORG.github.io',
      }
    });
  }
};
```

### 2. Set your API key as a Worker secret
In the Cloudflare dashboard → Workers → Your worker → Settings → Variables:
- Variable name: `ANTHROPIC_API_KEY`
- Value: `sk-ant-...`

### 3. Update index.html
Replace the fetch URL in the `runOCR()` function:
```javascript
// Change this line:
const response = await fetch('https://api.anthropic.com/v1/messages', {

// To your worker URL:
const response = await fetch('https://reg-ocr-proxy.YOUR_ACCOUNT.workers.dev', {
```

Also remove the `'anthropic-dangerous-direct-browser-access': 'true'` header — the worker handles auth now.

Now no one on the team needs to manage an API key.

---

## Fields extracted

| Field | Arabic | Notes |
|---|---|---|
| Plate number | رقم اللوحة | |
| Civil ID | الرقم المدني | Always verify — watermarks cause errors |
| Owner name | الاسم | |
| Chassis / VIN | رقم القاعدة | |
| Engine / model code | المحرك | |
| Year | سنة الصنع | |
| Expiry date | الانتهاء | |
| Insurance company | شركة التأمين | |
| Policy number | رقم الوثيقة | |
| Reference number | المرجع | |
| Passengers | الركاب | |
| Primary colour | اللون الأول | Often obscured by watermark |
| Secondary colour | اللون الثاني | Often obscured by watermark |
| Engine capacity (L) | الدرج | |
| Nationality | الجنسية | |
| Issue / print date | تاريخ الإصدار | |

---

## Confidence system

| Level | Meaning | Action required |
|---|---|---|
| **HIGH** | Clearly readable, Claude is certain | Auto-confirmed, no action needed |
| **MED** | Partially obscured or uncertain digit | User must type to confirm before export |
| **LOW** | Unreadable or completely obscured | User must enter manually before export |

**The "Copy all" and "Export CSV" buttons are blocked until every MED and LOW field has been manually confirmed.** This prevents unverified data from entering downstream systems.

---

## Tips for best results

- Photograph cards flat, not at an angle
- Avoid shadows across the card
- Good lighting — do not use flash directly on the card (causes glare)
- Higher resolution = better results
- If a scan gives bad results, use the "Re-scan" button (re-sends same image to Claude)

---

## Project structure

```
reg-ocr/
├── index.html      ← entire tool, single file
└── README.md       ← this file
```

Single-file design means zero build steps, zero dependencies, instant deployment.
