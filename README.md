<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:0f0c29,50:302b63,100:24243e&height=160&section=header&text=OSS%20Analysis%20Findings&fontSize=36&fontColor=ffffff&fontAlignY=40&desc=Real%20bugs%20found%20in%20actively%20maintained%20open%20source%20projects&descAlignY=60&descColor=c0c0ff" width="100%"/>

</div>

<br/>

<div align="center">

[![Repos Analyzed](https://img.shields.io/badge/Repos%20Analyzed-9+-0f0c29?style=flat-square)](.)
[![Total Reported](https://img.shields.io/badge/Total%20Reported-8-302b63?style=flat-square)](.)
[![Confirmed Fixed](https://img.shields.io/badge/Confirmed%20Fixed-2-2ea44f?style=flat-square)](.)
[![PRs Open](https://img.shields.io/badge/PRs%20Open-1-f0a500?style=flat-square)](.)

</div>

---

## 📌 Methodology

Each finding comes from deep commit-level diff analysis — examining what changed between revisions, identifying callers that weren't updated, and verifying the behavioral contract break is real.

**What I look for:**
- Silent return value changes (`null` / empty instead of expected data)
- Exception scope widening (`except AttributeError` → `except Exception`)
- Parameter removals that break callers without a compile-time error
- Unreachable code masking intended behavior
- Wrong entity type passed to external APIs

---

## ✅ Confirmed & Fixed

### 1 · MoneyPrinterTurbo — Qwen empty `choices[]` crash
**Repo:** [harry0703/MoneyPrinterTurbo](https://github.com/harry0703/MoneyPrinterTurbo) · **22K+ ⭐**
**Issue:** [#984](https://github.com/harry0703/MoneyPrinterTurbo/issues/984) · **Fix PR:** [#994](https://github.com/harry0703/MoneyPrinterTurbo/pull/994) ✅ Merged · **Date:** Jun 4, 2026

When the Qwen API returns an empty `choices[]` array (rate-limit or quota exhausted), the code attempts `response.choices[0]` with no guard — raising an unhandled `IndexError` with zero diagnostic context. Users had no way to distinguish a quota issue from a code bug.

```python
# Before — crashes silently
content = response.choices[0].message.content

# After — clear diagnostic
if not response.choices:
    raise ValueError("Qwen returned empty choices — check API key / quota")
content = response.choices[0].message.content
```

**Response:** Maintainer acknowledged the diagnostics gap, requested a focused PR → merged same day. A community contributor also offered to help.

---

### 2 · MoneyPrinterTurbo — Groq model unvalidated on list-fetch failure
**Repo:** [harry0703/MoneyPrinterTurbo](https://github.com/harry0703/MoneyPrinterTurbo) · **22K+ ⭐**
**Issue:** [#1013](https://github.com/harry0703/MoneyPrinterTurbo/issues/1013) · **Fix PR:** [#1014](https://github.com/harry0703/MoneyPrinterTurbo/pull/1014) ✅ Merged · **Date:** Jun 10, 2026

When the Groq model list endpoint fails (network error, auth issue), the UI silently falls back to the first entry in a hardcoded list. If that entry is stale, every generation call uses the wrong model — no error, no warning.

**Response:** Maintainer agreed, requested PR → merged.

---

## ⏳ Open / Pending

### 3 · MoneyPrinterTurbo — CLI `--video-source local` validation gaps
**Repo:** [harry0703/MoneyPrinterTurbo](https://github.com/harry0703/MoneyPrinterTurbo) · **22K+ ⭐**
**Issue:** [#1032](https://github.com/harry0703/MoneyPrinterTurbo/issues/1032) · **PR:** [#1033](https://github.com/harry0703/MoneyPrinterTurbo/pull/1033) ⏳ Open · **Date:** Jun 13, 2026

The new `cli.py` accepts `--video-source local` without requiring `--video-materials`, and accepts `--stop-at terms` with local sources even though term generation is skipped for local sources. Both cause failures deep into the run with unhelpful errors.

**Fix:** Early validation in `parse_args()` with clear error messages.

---

### 4 · medusajs/medusa — Race condition in `compensatePaymentIfNeededStep`
**Repo:** [medusajs/medusa](https://github.com/medusajs/medusa) · **28K+ ⭐**
**Discussion:** [#15550](https://github.com/medusajs/medusa/discussions/15550) ⏳ Watching · **Date:** Jun 4, 2026

Async workflow step `compensatePaymentIfNeededStep` has a potential race condition where concurrent order fulfillment flows could trigger duplicate payment compensation — leading to double refunds or inconsistent payment state.

---

### 5 · MoneyPrinterTurbo — `>=` comparison risk in duration check
**Repo:** [harry0703/MoneyPrinterTurbo](https://github.com/harry0703/MoneyPrinterTurbo) · **22K+ ⭐**
**Issue:** [#985](https://github.com/harry0703/MoneyPrinterTurbo/issues/985) ⏳ Community PR expected · **Date:** Jun 4, 2026

Duration boundary check uses `>=` where `>` is semantically correct — edge-case videos at exact boundary duration may be silently rejected or accepted incorrectly.

**Response:** Community contributor (Sushanth012) offered to submit a fix.

---

## 🔍 Recently Reported

### 6 · midjourney-api — `ChannelId` used as `ServerId` in Discord guild API
**Repo:** [erictik/midjourney-api](https://github.com/erictik/midjourney-api) · **1.8K ⭐**
**Issue:** [#294](https://github.com/erictik/midjourney-api/issues/294) 🔍 Open · **Date:** Jun 14, 2026

```typescript
// getCommand() and allCommand() — src/command.ts lines 68-72
let serverId = this.config.ServerId;
if (!serverId) {
  serverId = this.config.ChannelId;  // ⚠️ ChannelId ≠ guild ID
}
const url = `.../api/v9/guilds/${serverId}/application-command-index`;
```

`ServerId` is optional in config. When omitted, `ChannelId` is used as the guild ID — but Discord's endpoint requires a **server (guild) ID**. Channel IDs and guild IDs are different entity types in Discord's API. The call either returns 404 or wrong data, causing all command operations to fail.

---

### 7 · midjourney-api — Dead code in `cacheCommand()` — full cache never populated
**Repo:** [erictik/midjourney-api](https://github.com/erictik/midjourney-api) · **1.8K ⭐**
**Issue:** [#295](https://github.com/erictik/midjourney-api/issues/295) 🔍 Open · **Date:** Jun 14, 2026

```typescript
// src/command.ts lines 35-45
async cacheCommand(name: CommandName) {
  if (this.cache[name] !== undefined) return this.cache[name];
  const command = await this.getCommand(name);
  this.cache[name] = command;
  return command;        // ← exits here

  this.allCommand();     // ← dead code: never reached
  return this.cache[name]; // ← dead code: never reached
}
```

`allCommand()` was meant to bulk-fetch all commands on first miss, populating the full cache. The early `return` makes it unreachable — every subsequent `cacheCommand()` call hits Discord's API individually, causing unnecessary requests and potential rate limiting.

---

## 📊 Analysis Log

| Date | Repo | Commits Analyzed | Outcome |
|:-----|:-----|:----------------:|:--------|
| Jun 14 | apify/crawlee-python | 3 | Behavioral findings — silent URL filtering, exception propagation change |
| Jun 14 | tox-dev/tox | 1 | Config override namespace risk identified |
| Jun 14 | gptme/gptme | 1 | LLM routing refactor — HIGH risk noted |
| Jun 14 | erictik/midjourney-api | 1 | 2 confirmed bugs → issues #294 #295 opened |
| Jun 13 | harry0703/MoneyPrinterTurbo | cli.py | Issue #1032 + PR #1033 |
| Jun 10 | harry0703/MoneyPrinterTurbo | groq fix | Issue #1013 → PR #1014 merged ✅ |
| Jun 4 | medusajs/medusa | payment step | Discussion #15550 |
| Jun 4 | harry0703/MoneyPrinterTurbo | qwen fix | Issue #984 → PR #994 merged ✅ |

---

<div align="center">

*Commit-level diff analysis · Cross-module caller tracking · Behavioral contract verification*

<br/>

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:24243e,50:302b63,100:0f0c29&height=100&section=footer" width="100%"/>

</div>
