# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Structure

This repo (`miyossy75-alt/omurice-date2`) hosts two projects:
- `Omurice.html` — Kyoto omurice restaurant guide (static)
- `receipt-scanner.html` — Source file for the receipt scanner PWA (deployed separately)

**Active branch:** `claude/setup-receipt-scanner-rzdz1`

## Receipt Scanner System

A purchase tracking system with two components: a PWA web app and a Google Apps Script for auto-importing EC site orders from Gmail.

### 1. PWA Web App

- **Live URL:** `https://miyossy75-alt.github.io/receipt-scanner/`
- **Source:** `receipt-scanner.html` in this repo → manually copied to `miyossy75-alt/receipt-scanner/index.html` via GitHub web editor (MCP cannot push to that repo)
- Single-file HTML/CSS/JS with Claude Haiku Vision API for receipt OCR
- PWA files in `miyossy75-alt/receipt-scanner/`: `sw.js` (v3, network-first), `manifest.json`, `icon.svg`

**Key constants:**
```javascript
const GAS_URL = 'https://script.google.com/macros/s/AKfycbw5jHAWHMdDzmuCUrfToDWHW1Ck3zmKiprBqC4GtkHxlQPNwPmrS46hy0thSMgIJbkYHw/exec';
const STORAGE_KEY = 'receipts_v1';
const API_KEY_STORAGE = 'anthropic_api_key';
```

**Categories:** ゲーム機、ゲームソフト、PC周辺機器、美容家電、一般家電、その他

**Critical patterns:**
- `saveReceipt()` must be synchronous with fire-and-forget fetch (`mode: 'no-cors'`, `.catch(() => {})`) — async/await causes the button to freeze on "送信中…"
- Never store base64 images in localStorage — causes QuotaExceededError
- Always use `todayJST()` for dates, never `new Date().toISOString()` (UTC vs JST mismatch)
- Service worker must use network-first strategy; cache-first causes stale content

**JST date function:**
```javascript
function todayJST() {
  const d = new Date();
  return `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
}
```

**Service worker (sw.js) — must stay network-first:**
```javascript
const CACHE = 'receipt-scanner-v3';
self.addEventListener('install', e => { self.skipWaiting(); });
self.addEventListener('activate', e => {
  e.waitUntil(caches.keys().then(keys => Promise.all(keys.map(k => caches.delete(k)))));
  self.clients.claim();
});
self.addEventListener('fetch', e => {
  e.respondWith(fetch(e.request).catch(() => caches.match(e.request)));
});
```

### 2. Google Apps Script

**Spreadsheet ID:** `1tCcbj35ufLwGPYdb6BA5Tu9VqkMcJuRjYyhYaN8f4TY`

**Columns (A–L):** タイムスタンプ, ASIN, JAN, 商品名, カテゴリ, コンディション, 仕入れ日, 仕入れ先, 仕入れ価格, 数量, 想定販売価格, メモ

**Functions:**
| Function | Purpose |
|---|---|
| `doPost(e)` | Receives scanner POST and appends row |
| `importOrders()` | Main Gmail import — run manually or via daily trigger |
| `setDailyTrigger()` | Sets 8am daily auto-run (run once to register) |
| `fetchAmazon()` / `parseAmazonBody()` | Amazon email parser |
| `fetchRakuten()` | 楽天市場 + 楽天ブックス |
| `fetchYahoo()` | Yahoo!ショッピング (forwarded from Outlook) |
| `fetchYodobashi()` | ヨドバシ.com with Gold Points skip logic |
| `resetAmazonLabels()` | Remove Gmail labels from Amazon emails for re-import |
| `resetYodobashiLabels()` | Remove Gmail labels from Yodobashi emails for re-import |
| `debugAmazon()` | Logs email body to help debug price extraction |

**EC site email addresses:**
| Store | From address |
|---|---|
| Amazon | `amazon.co.jp` (domain match) |
| 楽天市場 | `order@rakuten.co.jp` |
| 楽天ブックス | `books-order@mail.books.rakuten.co.jp` |
| ヨドバシ.com | `thanks_gochuumon@yodobashi.com` |
| Yahoo!ショッピング | `shopping-order-master@mail.yahoo.co.jp` (forwarded Outlook→Gmail) |

**Deduplication:** Gmail label `仕入れ記録済み` — once applied, emails are excluded from all future `importOrders()` runs even if spreadsheet rows are manually deleted.

**Special logic:**

Yodobashi — skip Gold Points full-payment orders (likely personal purchases):
```javascript
const totalM = body.match(/今回のお買い物合計金額[^\d]*([\d,]+)/);
const pointM = body.match(/ゴールドポイントでのお支払い[^\d]*([\d,]+)/);
if (totalM && pointM && totalM[1] === pointM[1]) continue; // skip
```

Amazon — skip cancellation/refund/shipping notification emails:
```javascript
if (body.includes('キャンセルされました') ||
    body.includes('返金') ||
    body.includes('発送しました')) return [];
```

Amazon — price extraction tries multiple patterns:
```javascript
const pricePatterns = [
  /注文合計[：:\s]*[¥￥]([\d,]+)/,
  /ご注文合計[：:\s]*[¥￥]([\d,]+)/,
  /お支払い金額[：:\s]*[¥￥]([\d,]+)/,
  /合計金額[：:\s]*[¥￥]([\d,]+)/,
  /合計[：:\s]*[¥￥]([\d,]+)/,
  /(\d[\d,]+)\s*JPY/,
];
```

## Deployment Workflow

**Updating the PWA:**
1. Edit `receipt-scanner.html` locally and commit to `claude/setup-receipt-scanner-rzdz1`
2. User manually pastes into `miyossy75-alt/receipt-scanner/index.html` via GitHub web editor (split front/back halves due to file length)
3. User clears browser cache or reinstalls PWA to pick up changes

**Updating sw.js:** bump CACHE version string (currently `receipt-scanner-v3`) to force cache invalidation on all clients.

**GAS changes:** edit directly in the Google Spreadsheet's Apps Script editor — not stored in this repo.

## Constraints

- MCP GitHub tools are restricted to `miyossy75-alt/omurice-date2` — cannot push to `miyossy75-alt/receipt-scanner`
- GAS script lives only in the Google Spreadsheet's Apps Script editor
- Claude API key is stored in the browser (localStorage `anthropic_api_key`) and entered by the user — never hardcode it
