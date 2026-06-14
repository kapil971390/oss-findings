<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:0f0c29,50:302b63,100:24243e&height=160&section=header&text=OSS%20Analysis%20Findings&fontSize=36&fontColor=ffffff&fontAlignY=40&desc=Real%20bugs%20found%20in%20actively%20maintained%20open%20source%20projects&descAlignY=60&descColor=c0c0ff" width="100%"/>

</div>

<br/>

<div align="center">

[![Total Findings](https://img.shields.io/badge/Total%20Findings-5-0f0c29?style=flat-square&logo=github)](.)
[![Confirmed Fixed](https://img.shields.io/badge/Confirmed%20Fixed-2-2ea44f?style=flat-square)](.)
[![PRs Open](https://img.shields.io/badge/PRs%20Open-1-f0a500?style=flat-square)](.)
[![Under Review](https://img.shields.io/badge/Under%20Review-2-302b63?style=flat-square)](.)

</div>

---

## 📌 Methodology

Each finding comes from deep commit-level analysis — examining what changed between two revisions, identifying callers that were not updated, and verifying the behavioral contract break is reproducible.

**What I look for:**
- Silent return value changes (`null` / empty instead of expected data)
- Exception scope widening (`except AttributeError` → `except Exception`)
- Parameter removals that break callers without a compile error
- Unreachable code masking intended behavior
- Wrong entity type passed to APIs (e.g. channel ID where guild ID expected)

---

## ✅ Confirmed & Fixed

### 1 · MoneyPrinterTurbo — Qwen empty `choices[]` crash
**Repo:** [harry0703/MoneyPrinterTurbo](https://github.com/harry0703/MoneyPrinterTurbo)  
**Issue:** [#984](https://github.com/harry0703/MoneyPrinterTurbo/issues/984) · **Fix PR:** [#994](https://github.com/harry0703/MoneyPrinterTurbo/pull/994) ✅ Merged

When the Qwen API returns an empty `choices[]` array (rate-limit or quota exhausted), the code attempts `response.choices[0]` with no guard — raising an unhandled `IndexError` with no diagnostic message. Users had no way to tell whether the problem was their API key, their quota, or the service itself.

```python
# Before — unguarded
content = response.choices[0].message.content

# After — with diagnostic
if not response.choices:
    raise ValueError("Qwen returned empty choices — check API key / quota")
content = response.choices[0].message.content
```

**Maintainer response:** Acknowledged, requested focused PR → merged same day.

---

### 2 · MoneyPrinterTurbo — Groq model unvalidated on list-fetch failure
**Repo:** [harry0703/MoneyPrinterTurbo](https://github.com/harry0703/MoneyPrinterTurbo)  
**Issue:** [#1013](https://github.com/harry0703/MoneyPrinterTurbo/issues/1013) · **Fix PR:** [#1014](https://github.com/harry0703/MoneyPrinterTurbo/pull/1014) ✅ Merged

When the Groq model list endpoint fails (network error, auth issue), the UI falls back silently to the first entry in a hardcoded list. If that entry is stale or wrong, every subsequent generation call uses the wrong model — no error, no warning.

**Maintainer response:** Agreed, requested PR → merged.

---

## ⏳ Pending

### 3 · MoneyPrinterTurbo — CLI local source validation gaps
**Repo:** [harry0703/MoneyPrinterTurbo](https://github.com/harry0703/MoneyPrinterTurbo)  
**Issue:** [#1032](https://github.com/harry0703/MoneyPrinterTurbo/issues/1032) · **PR:** [#1033](https://github.com/harry0703/MoneyPrinterTurbo/pull/1033) ⏳ Open

The new `cli.py` (merged in PR #1005) accepts `--video-source local` without requiring `--video-materials`, and accepts `--stop-at terms` with local sources even though term generation is skipped for local sources. Both issues cause failures deep into the run with unhelpful errors.

**Fix:** Early argument validation in `parse_args()` with clear error messages.

---

## 🔍 Under Review

### 4 · midjourney-api — `ChannelId` used as `ServerId` in guild API
**Repo:** [erictik/midjourney-api](https://github.com/erictik/midjourney-api)  
**File:** `src/command.ts` lines 47–50, 68–70

```typescript
// getCommand() and allCommand() both do:
let serverId = this.config.ServerId;
if (!serverId) {
  serverId = this.config.ChannelId;  // ⚠️ Wrong — ChannelId ≠ guild ID
}
const url = `.../api/v9/guilds/${serverId}/application-command-index`;
```

`ServerId` is optional in the config. When omitted, `ChannelId` is used as the guild ID — but Discord's `/guilds/{id}/application-command-index` endpoint requires a **server (guild) ID**, not a channel ID. These are different entity types. The API call either fails or returns commands from the wrong server.

---

### 5 · midjourney-api — Dead code in `cacheCommand()` — cache never fully populated
**Repo:** [erictik/midjourney-api](https://github.com/erictik/midjourney-api)  
**File:** `src/command.ts` lines 39–45

```typescript
async cacheCommand(name: CommandName) {
  if (this.cache[name] !== undefined) return this.cache[name];
  const command = await this.getCommand(name);
  this.cache[name] = command;
  return command;           // ← returns here
  this.allCommand();        // ← unreachable: never called
  return this.cache[name];  // ← unreachable
}
```

`allCommand()` fetches and caches **all** bot commands in one API call. It was intended to run after the first individual lookup to populate the full cache — but the early `return` makes it dead code. Every subsequent `cacheCommand()` call hits the Discord API individually instead of using the pre-populated cache, causing unnecessary API calls and potential rate limiting.

---

<div align="center">

*Analysis performed using commit-level diff inspection and cross-module caller tracking.*

<br/>

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:24243e,50:302b63,100:0f0c29&height=100&section=footer" width="100%"/>

</div>
