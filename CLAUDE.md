# Explorer AI Proofreader — Project Memory

## What this project is
Single-file browser app for professional AI-powered proofreading. Powered by Anthropic Claude API (direct browser → API calls). No backend, no server — pure client-side HTML/JS/CSS.

**Live repo:** github.com/shakkeer-cloud/ai-proofreader  
**Branch:** master  
**Current version:** v3.0.1

---

## Files

| File | Purpose | Lines |
|---|---|---|
| `index.html` | Main app — all logic, UI, CSS in one file | ~3000 |
| `share.html` | Review workflow — load a JSON report, reviewer marks decisions | ~1300 |
| `review.html` | Older review page (not actively maintained) | ~1160 |

**RULE: Only ever edit `index.html` unless explicitly asked to change another file.**

---

## Architecture (index.html)

- **PDF.js** (cdnjs 2.16.105) for client-side PDF text extraction
- **Mammoth** for .docx extraction
- **jsPDF** + **docx.js** for PDF and Word exports
- **Anthropic API** called directly from browser with `anthropic-dangerous-direct-browser-access: true`
- **localStorage** for: API key, user name, history (20 entries), last report (24h), cost stats, glossary, page texts (retry), theme
- No framework — vanilla JS throughout

---

## Key constants (index.html ~line 638)

```js
CHUNK_SIZE = 45000        // chars per chunk (~16k tokens)
CHUNK_OVERLAP = 150       // chars of overlap at chunk boundaries
RATE_LIMIT_WAIT_S = 75    // seconds between chunks
```

## Models
```js
claude-sonnet-4-6          // default, fast, recommended for RTL/complex scripts
claude-opus-4-7            // best quality, slow, expensive
claude-haiku-4-5-20251001  // fastest, cheapest, good for large batch docs
```

## MODEL_PRICING (per 1M tokens)
```js
sonnet-4-6:  { in: 3.00,  out: 15.00 }
opus-4-7:    { in: 15.00, out: 75.00 }
haiku-4-5:   { in: 0.80,  out: 4.00  }
```

---

## State object (docInfo)

```js
docInfo = {
  filename, ext, fileSize,
  pages,          // int — PDF page count
  pageTexts,      // string[] — per-page text (used for page number attribution)
  wordCount, charCount, paraCount,
  images, graphics, tables,
  language,       // detected language string
  isSpread,       // bool — InDesign double-page spread detected (page width > 750pt)
  scriptDir,      // 'ltr' | 'rtl'
}
```

---

## v3.0.0 / v3.0.1 — What was built (May 2026)

### PDF extraction improvements
- **Column-aware extraction** (`buildColumnText`): for spread PDFs, text items split at page midpoint, left column extracted first then right — prevents word interleaving
- **Spread auto-detection**: first page viewport width > 750pt = spread; user can override with Page Layout select
- **RTL word-order fix** (`fixRTLWordOrder`): pdf.js returns Arabic/Hebrew in visual L→R order; words are reversed per line when RTL script detected
- **NFC normalization**: fixes Bengali/Arabic/Hindi glyph fragments after extraction
- **Decorative text removal** (`fixDecorativeText`): strips InDesign shadow/echo text like "KK IISSAA HH"
- **Text compression** (`compressExtractedText`): collapses whitespace, trims blank lines
- **Quality warning** (`checkExtractionQuality`): alerts user when < 10 words/page (likely scanned)
- **Urdu advisory**: warns user that Nastaliq fonts use proprietary glyph maps — paste manually

### New UI controls
- **Page Layout** select: Auto-detect / Single page / Double-page spread
- **Reading Direction** select: Auto-detect / LTR / RTL
- **⚡ Quick** preset check button (Typos + Spelling + Grammar + Punctuation) — fastest/cheapest
- **Cost Tracker** card in Setup page (session, all-time, avg/run, Reset)
- **Confirm modal**: now shows per-1k-words cost + Layout/Direction info row + smarter model tip (RTL → Sonnet, large → Haiku)
- **Report cost line**: shows per-1k-words cost alongside total
- **Analyse panel**: shows Script Direction + Page Layout stats

### Language support
- `detectLanguage()` now covers: Hebrew, Persian, Hindi, Bengali, Tamil, Thai, Japanese, Korean (script-range detection) + Latin-script scoring (Indonesian, Malay, French, Spanish, German, English)
- Dropdown added: Hebrew, Japanese, Korean, Persian, Traditional Chinese
- `RTL_LANGS = new Set(['arabic','hebrew','persian','urdu'])`
- System prompt: RTL note, Thai note (no word-spaces), spread note — all injected per-document

### Cost reduction
- **Prompt caching**: system sent as `[{type:'text', text:systemPrompt, cache_control:{type:'ephemeral'}}]` with `anthropic-beta: prompt-caching-2024-07-31` header — ~80% cost saving on multi-chunk runs
- **Adaptive max_tokens**: Haiku 1500–4096, Sonnet 2000–8000 (sized per chunk word count)
- **CHUNK_OVERLAP**: reduced from 400 → 150 chars
- **CJK sentence boundaries** in `splitIntoChunks()`: `。！？；`

### Export fixes (v3.0.1)
- `buildCtxParts(context, original, replacement)` — plain-text equivalent of `buildCtx()`; returns `{before, highlight, after}`
- **Word export**: `cellCtx()` builds multi-TextRun cells — shows full context sentence with original in red, corrected in green; surrounding context in grey
- **PDF export**: uses `buildCtxParts()` for full context strings; columns widened (38→46mm / 44→52mm); page-break guard for long rows
- Column headers renamed: "Original (in context)" / "Corrected (in context)"

### Fonts
- Noto Sans Bengali, Arabic, Devanagari, Hebrew, Tamil, Thai, Nastaliq Urdu loaded via Google Fonts
- Multi-script CSS font stack on `.context-snippet, td`

---

## Key functions to know

| Function | What it does |
|---|---|
| `extractText(file)` | Dispatches to PDF/DOCX/TXT extractor |
| `buildColumnText(items, isSpread)` | Column-aware pdf.js item → text |
| `fixRTLWordOrder(text)` | Reverses word order in RTL lines |
| `fixDecorativeText(text)` | Strips InDesign shadow-layer echoes |
| `compressExtractedText(text)` | Collapses whitespace |
| `checkExtractionQuality(text, pages)` | Returns warning string or null |
| `detectLanguage(text)` | Returns language name string |
| `isRTLLang(lang)` | Returns bool |
| `splitIntoChunks(text, size)` | Latin + CJK sentence boundaries |
| `addChunkOverlap(chunks, n)` | Adds n-char tail from previous chunk |
| `callApiWithRetry(payload, key, …)` | Fetch with 3× retry + rate-limit wait |
| `proofreadOne(file, paste, key, show)` | Full single-document proofreading flow |
| `renderReport(r)` | Renders report page from report object |
| `buildCtx(ctx, orig, cls, repl)` | HTML context snippet with mark |
| `buildCtxParts(ctx, orig, repl)` | Plain-text {before,highlight,after} for exports |
| `renderCostTracker()` | Updates Setup cost tracker card |
| `trackCost(usd)` | Adds to session + localStorage stats |
| `saveToHistory(report)` | Persists to proof_history (max 20) |
| `loadHistoryItem(id)` | Restores report from history |
| `saveLastReport(report)` | Persists to proof_last_report (24h TTL) |
| `showConfirmModal(…)` | Pre-run cost estimate modal |
| `retryFailedChunks()` | Continues partial run from _retryState |

---

## API call format (proofreadOne)

```js
{
  model,
  max_tokens: adaptiveMaxTok,   // 1500–8000 based on chunk word count
  temperature: 0,
  system: [{
    type: 'text',
    text: systemPrompt,
    cache_control: { type: 'ephemeral' }
  }],
  messages: [{ role:'user', content: `Proofread this text (part N of M):\n\n${chunk}` }]
}
// Headers include: anthropic-beta: prompt-caching-2024-07-31
```

---

## System prompt structure (proofreadOne)

1. Language line (auto-detect instruction OR `The document is in ${resolvedLang}`)
2. Checks line (`Check ONLY these error types: …`)
3. 8 error type definitions (Typos, Spelling, Grammar, Punctuation, Capitalization, Style, Consistency, Clarity)
4. Severity rules
5. 4 important rules (exact text, page markers, explanation format, certainty)
6. Optional: glossary line (from localStorage)
7. Optional: RTL note (when scriptDir === 'rtl')
8. Optional: Thai note (when language === Thai)
9. Optional: spread note (when isSpread === true)
10. JSON schema for response

---

## share.html key facts

- Loads report from localStorage key `proof_last_report`
- Uses Supabase for persisting shared reviews (key in localStorage: `proof_supabase_*`)
- Reviewer can mark each error: Accept / Reject / Note
- Exports: PDF, Word, CSV of review decisions
- Bug fixes in current version: `_origReviewHTML` captured at init; `_syncBtns` queries correct selector; removed dead `row.className` line in `setDecision()`

---

## Commit conventions

Use descriptive commit messages. Always add:
```
Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

## Things that do NOT work / known limitations
- Urdu Nastaliq fonts: proprietary glyph maps — pdf.js cannot extract readable text; user is warned to paste manually
- Scanned PDFs: no OCR support; quality warning shown when < 10 words/page
- Very large files (>300k chars): user warned; Haiku recommended for rate-limit safety
- Opus 4.7 on large docs: very slow + expensive; confirm modal warns
