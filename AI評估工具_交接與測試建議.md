# 筋動人生 AI 評估工具 — 交接與測試建議

> **這份文件要給誰讀**
> 給正在協助建置「筋動人生」**官方網站**的 AI 或工程師。
>
> 寫這份文件的 AI 只看得到舊的原型專案（`index.html` + `mobile.html` 兩個靜態檔），**看不到官網的程式碼**。所以底下講的「現況」都是指原型；官網若已重構，請把結論對應到你手上的實際檔案，不要照抄路徑。
>
> **核心訴求：** 這是一個「傳統整復推拿」業者的網站，內含一個 AI 症狀評估工具。這個產業對「療效宣稱」與「疑似診斷」的用詞非常敏感，業主本人也很清楚（他自己在 prompt 裡列了禁語清單）。因此底下的建議**優先順序是照「生意風險」排的，不是照「工程完整度」排的**。

---

## 目錄

1. [原型現況速覽](#1-原型現況速覽)
2. [P0 — 禁語即時攔截（最優先，且與測試無關）](#2-p0--禁語即時攔截)
3. [P0 — 醫療安全攔截機制的測試](#3-p0--醫療安全攔截機制的測試)
4. [P1 — 已確認的分頁 Bug 與修正](#4-p1--已確認的分頁-bug-與修正)
5. [P2 — 兩個版本的邏輯歧異（官網整合前必須做決定）](#5-p2--兩個版本的邏輯歧異)
6. [P2 — 網路錯誤處理](#6-p2--網路錯誤處理)
7. [P3 — innerHTML 注入風險](#7-p3--innerhtml-注入風險)
8. [建議的測試環境與檔案結構](#8-建議的測試環境與檔案結構)
9. [給接手者的提醒](#9-給接手者的提醒)

---

## 1. 原型現況速覽

**測試覆蓋率：0%。** 沒有 `package.json`、沒有測試框架、沒有 CI。查過完整 git 歷史，從未存在過任何測試檔案。

**專案結構（原型）：**

```
index.html    427 行  — 桌機/A4 看板版，內嵌 JS 在 268-425 行
mobile.html   368 行  — 手機版，內嵌 JS 在 157-366 行
營業時間.jpg
筋動人生 (廣告看板 (直式)).png
```

**這個工具在做什麼：**

客人在表單勾選「職業型態 / 不適部位 / 感覺 / 特殊狀況」，前端組成 prompt 打到一個 Cloudflare Worker（`WORKER_URL`，兩個檔案都寫死同一個網址），Worker 轉呼叫 Claude 或 Gemini，回傳一段「人體力學分析」文字，前端渲染出來並附上 LINE 預約連結。

**關鍵邏輯位置（原型）：**

| 邏輯 | index.html | mobile.html |
|---|---|---|
| Worker URL（寫死） | 274 | 158 |
| System prompt（含禁語清單） | 346-362 | 160-206 |
| 主流程 `analyzeSymptoms()` | 314-408 | 269-349 |
| **醫療安全攔截** | 335-342 | 291-310 |
| 分頁 `splitIntoPages()` | 無 | 211-228 |
| 渲染 | 410-417 | 230-261 |
| 重試迴圈 | 無 | 318-338 |
| 特殊狀況提醒清單（寫死） | 386-388 | 240 |

---

## 2. P0 — 禁語即時攔截

### 問題

業主在 prompt 裡明確定義了禁語（`mobile.html:205-206`）：

```
【絕對禁止字眼】
神經、壓迫神經、骨頭、脊椎、椎間盤、矯正、復位、治療、治癒、發炎、診斷、無力、建議您
```

`index.html:359-362` 也有類似規定（禁止「神經」「壓迫神經」，嚴禁「矯正、復位、治癒、發炎、治療」）。

**但程式碼完全沒有檢查 AI 有沒有照做。**

`index.html:382` 與 `mobile.html:329` 都是直接把 `data.result` 拿來用，中間沒有任何驗證：

```js
let finalResponse = data.result.trim() + "\n\n";   // index.html:382
pages = splitIntoPages(data.result.trim());        // mobile.html:329
```

LLM 的輸出是機率性的，prompt 只是「強烈建議」不是「保證」。只要 AI 有一次回了「腰椎附近的**神經**受到**壓迫**」，這句話會**原封不動顯示給客人看**，而且沒有任何人會知道。

### 為什麼這是 P0

原型是自己測試用的頁面，風險有限。**官網不一樣** —— 對外公開、24 小時無人看守、會有真實客人、內容會被截圖。這是整個專案裡唯一「不做任何測試也應該立刻補」的項目，因為它是**即時防線**，不是測試。

### ⚠️ 實作時最容易踩的陷阱

**只能掃描 AI 回傳的那一段，絕對不要掃描整份最終文字。**

因為業主自己寫死的固定文案本身就含有禁語，這是合法且刻意的：

- `index.html:384`：「**我們不針對疾病做治療**」← 含「治療」
- `index.html:340`：「近期骨折或急性發炎的狀態」← 含「發炎」
- checkbox 選項值：「近期骨折/急性發炎」← 含「發炎」

如果對最終字串做過濾，會把業主自己的合規聲明誤判成違規。**過濾的對象只有 `data.result`。**

### 建議實作

```js
// 注意：「無力」與「建議您」只出現在 mobile 的禁語清單，
// 而 index.html 卻提供「無力」作為可勾選的感覺選項 —— 見第 5 節，需先做決定。
const BANNED_WORDS = [
  '神經', '壓迫神經', '骨頭', '脊椎', '椎間盤',
  '矯正', '復位', '治療', '治癒', '發炎', '診斷', '建議您'
];

/** 回傳命中的禁語陣列；空陣列代表乾淨 */
function findBannedWords(aiText) {
  return BANNED_WORDS.filter(word => aiText.includes(word));
}

/** 當 AI 輸出違規時顯示的安全備援文案 */
const SAFE_FALLBACK = `謝謝您提供的資訊！

依照您勾選的部位與感覺，建議由師傅在現場親自為您做更準確的評估，這樣我們才能針對您的實際狀況安排合適的手法與力道。

歡迎在官方 LINE 留言您方便的時段與地點，會有專人為您安排 😊`;
```

在取得 `data.result` 之後、渲染之前插入：

```js
const aiText = data.result.trim();
const hits = findBannedWords(aiText);

if (hits.length > 0) {
  // 建議：先重試一次（mobile 版已有重試迴圈可沿用），
  // 若重試後仍違規，改用安全備援文案，絕不顯示原始輸出。
  console.warn('[禁語攔截] AI 輸出含禁語，已改用備援文案：', hits);
  renderFinalResponse(SAFE_FALLBACK);   // 或 mobile 版對應的渲染函式
  return;
}

// 乾淨才走原本流程
```

**選配但建議：** 把命中事件回報給 Worker 記錄下來，讓業主知道違規發生的頻率。若頻率偏高，代表 prompt 需要調整，而不是靠過濾硬擋。

### 對應測試

```js
describe('禁語攔截', () => {
  it('乾淨的 AI 輸出不應被攔截', () => {
    expect(findBannedWords('▍【肩頸】\n狀況：像背帶越綁越緊')).toEqual([]);
  });

  it('含「神經」的輸出應被攔截', () => {
    expect(findBannedWords('腰椎附近的神經受到壓迫')).toContain('神經');
  });

  it('含「發炎」的輸出應被攔截', () => {
    expect(findBannedWords('局部有輕微發炎現象')).toContain('發炎');
  });

  it('違規時顯示備援文案，不得顯示原始 AI 輸出', async () => {
    mockWorkerResponse({ result: '您的脊椎神經受到壓迫，建議矯正治療' });
    await analyzeSymptoms();
    const shown = document.getElementById('ai-response').innerHTML;
    expect(shown).not.toContain('神經');
    expect(shown).toContain('現場親自為您做更準確的評估');
  });

  // 這條是防呆，避免未來有人把過濾套用到整份文字上
  it('業主自己的固定文案不受影響', async () => {
    mockWorkerResponse({ result: '▍【肩頸】\n狀況：像背帶越綁越緊' });
    await analyzeSymptoms();
    // index.html 的固定文案含「治療」二字，必須完整保留
    expect(document.getElementById('ai-response').innerHTML)
      .toContain('我們不針對疾病做治療');
  });
});
```

---

## 3. P0 — 醫療安全攔截機制的測試

### 現況

`analyzeSymptoms()` 會在**發出網路請求之前**攔截兩種狀況，直接回覆婉拒訊息並 `return`：

- 勾選「懷孕中」→ 不提供孕期調理，請諮詢婦產科
- 勾選「近期骨折/急性發炎」→ 請先以醫療院所診治為主

程式碼：`index.html:335-342`、`mobile.html:291-310`。

### 風險

判斷方式是**對硬編碼字串做精確比對**：

```js
if (warningsArray.includes('懷孕中')) { ... }
```

而 `warningsArray` 的值來自 HTML checkbox 的 `value` 屬性（`index.html:229`、`mobile.html:108`）。

**只要有人改動選項文字，攔截就會靜默失效。** 例如把「近期骨折/急性發炎」改成比較不嚇人的「近期受傷」，或在「懷孕中」後面加個 emoji —— 攔截條件不再成立，程式**不會報錯、畫面看起來一切正常**，但懷孕的客人從此會拿到一份肌肉力學分析，而不是就醫轉介建議。

這在官網階段特別危險，因為文案調整是最常見的改動。

### 對應測試

**關鍵原則：測試必須從 DOM 讀出實際的 checkbox 值，不可以在測試裡再寫一次字串。** 在測試裡重寫字串等於把同一個錯誤抄兩遍，改名時兩邊一起改就完全失去防護意義。

```js
describe('醫療安全攔截', () => {
  it('勾選懷孕中 -> 顯示婉拒訊息', async () => {
    checkWarning('懷孕中');
    await analyzeSymptoms();
    expect(document.getElementById('ai-response').innerHTML)
      .toContain('暫不提供孕期調理服務');
  });

  it('勾選懷孕中 -> 絕對不可呼叫 Worker', async () => {
    const spy = vi.spyOn(globalThis, 'fetch');
    checkWarning('懷孕中');
    await analyzeSymptoms();
    expect(spy).not.toHaveBeenCalled();   // 這條最重要
  });

  it('勾選近期骨折/急性發炎 -> 顯示就醫建議且不呼叫 Worker', async () => {
    const spy = vi.spyOn(globalThis, 'fetch');
    checkWarning('近期骨折/急性發炎');
    await analyzeSymptoms();
    expect(document.getElementById('ai-response').innerHTML).toContain('醫療院所');
    expect(spy).not.toHaveBeenCalled();
  });

  // ★ 防改名的核心測試：直接從 DOM 抓值，不寫死字串
  it('每一個應攔截的選項，其 DOM 值都必須真的觸發攔截', async () => {
    const values = Array.from(
      document.querySelectorAll('input[name="warnings"]')
    ).map(el => el.value);

    // 用語意特徵尋找，而非精確字串，這樣改文案時測試會指出問題而不是默默通過
    const shouldBlock = values.filter(v => /懷孕|骨折|急性發炎/.test(v));
    expect(shouldBlock.length).toBeGreaterThanOrEqual(2);   // 確保沒被整個刪掉

    for (const value of shouldBlock) {
      resetForm();
      const spy = vi.spyOn(globalThis, 'fetch');
      checkWarningByValue(value);
      await analyzeSymptoms();
      expect(spy, `選項「${value}」應觸發攔截但沒有`).not.toHaveBeenCalled();
      spy.mockRestore();
    }
  });
});
```

### 同類問題：特殊狀況提醒清單

`mobile.html:240` 另外硬編碼了一份清單：

```js
const specialWarnings = ['植入金屬物(鋼釘鋼板)', '骨質疏鬆', '心血管疾病/抗凝血劑'];
```

同樣的字串耦合，風險較低（只是少一則安全提醒，不是漏掉就醫轉介）。一條測試即可：

```js
it('特殊提醒清單必須是實際 checkbox 值的子集合', () => {
  const domValues = Array.from(
    document.querySelectorAll('input[name="warnings"]')
  ).map(el => el.value);
  specialWarnings.forEach(w => expect(domValues).toContain(w));
});
```

> **順帶一提：** `index.html:386-388` 的做法不同 —— 它把 `warningsArray` **全部**列進提醒，沒有另一份清單。官網整合時要決定採用哪一種。

---

## 4. P1 — 已確認的分頁 Bug 與修正

### 這是實際存在的 Bug，已用程式驗證

`mobile.html:211` 的 `splitIntoPages()` 用意是「每頁放 2 個部位段落」，但實作有缺陷：

```js
const rawParts = text.split(/(?=▍)/);   // ← 只以 ▍ 切分
```

AI 的輸出格式是：多個 `▍【部位】` 段落，最後接一個 `📌 您目前的感覺分析` 區塊。**但 `📌` 區塊前面沒有 `▍`**，所以它永遠不會被切開，而是黏在前一個 `▍` 段落後面。接著這一整塊因為 `trimmed.includes('📌')` 為真，被整個歸類成 `sensePart` —— **最後一個部位段落因此被誤判掉了**。

實測結果：

| AI 回傳段落數 | 實際頁數 | 應有頁數 |
|---|---|---|
| 1 | 1 | 1 |
| 2 | 1 | 1 |
| **3** | **1** | **2** |
| 4 | 2 | 2 |
| **5** | **2** | **3** |

內容沒有遺失，但**分頁一律少切一頁**。3 個部位會擠成一頁長長的捲動 —— 正是當初做分頁要避免的情況。

另外 `if (bodyParts.length <= 2) return [text]` 回傳的是**未經處理的原始字串**，另一分支回傳的是 **trim 過並重新 join 的內容**，兩條路徑輸出格式不一致。

### 修正版（已驗證 13 項測試全數通過）

```js
function splitIntoPages(text, perPage = 2) {
  // 先把 📌 感覺分析整段抽離（它不一定有 ▍ 前綴，這是原版的根本問題）
  const senseIdx = text.indexOf('📌');
  const body      = senseIdx >= 0 ? text.slice(0, senseIdx) : text;
  const sensePart = senseIdx >= 0 ? text.slice(senseIdx).trim() : '';

  const bodyParts = body.split(/(?=▍)/).map(s => s.trim()).filter(Boolean);

  // 完全沒有 ▍ 結構時退回單頁，避免內容消失
  if (bodyParts.length === 0) return [text];

  const pages = [];
  for (let i = 0; i < bodyParts.length; i += perPage) {
    pages.push(bodyParts.slice(i, i + perPage).join('\n\n'));
  }
  if (sensePart) pages[pages.length - 1] += '\n\n' + sensePart;
  return pages;
}
```

### 對應測試

```js
describe('splitIntoPages', () => {
  const makeText = n =>
    Array.from({ length: n }, (_, i) => `▍【部位${i + 1}】\n狀況${i + 1}`).join('\n\n')
    + '\n\n📌 您目前的感覺分析\n・痠痛/緊繃：xxx';

  it.each([[1, 1], [2, 1], [3, 2], [4, 2], [5, 3]])(
    '%i 個部位應分成 %i 頁', (sections, expectedPages) => {
      expect(splitIntoPages(makeText(sections))).toHaveLength(expectedPages);
    });

  it('感覺分析必須出現在最後一頁', () => {
    const pages = splitIntoPages(makeText(5));
    expect(pages[pages.length - 1]).toContain('📌');
  });

  it('任何部位內容都不可遺失', () => {
    const joined = splitIntoPages(makeText(5)).join('');
    for (let i = 1; i <= 5; i++) expect(joined).toContain(`部位${i}`);
  });

  // 異常輸入（AI 沒照格式回時不可炸掉或吃掉內容）
  it('沒有 ▍ 只有 📌 -> 原文單頁',   () => expect(splitIntoPages('📌 感覺\n・痠')).toHaveLength(1));
  it('空字串不可拋錯',                () => expect(splitIntoPages('')).toHaveLength(1));
  it('沒有 📌 也要能正常分頁',        () => expect(splitIntoPages('▍A\n1\n\n▍B\n2\n\n▍C\n3')).toHaveLength(2));
});
```

---

## 5. P2 — 兩個版本的邏輯歧異

兩個檔案實作同一功能，但已經產生差異。**有些看起來不像刻意設計，而是無意造成的。官網整合時每一項都要做決定：**

| 項目 | index.html | mobile.html | 備註 |
|---|---|---|---|
| 「感覺」是否必填 | **選填**，未選時預設 `未具體說明`（321） | **必填**，未選會 alert（277） | 需統一 |
| 上背部位的值 | `上背/膏肓` | `上背/肩胛骨內側` | 字串不同，影響 prompt 對照庫比對 |
| 感覺選項 | 痠 / 痛 / 麻 / 緊繃卡卡的 / **無力** | 痠痛緊繃 / 沉重感 | 差異很大 |
| **「無力」的矛盾** | 提供為可勾選項目（222） | **列在禁語清單**（206） | ⚠️ 直接衝突，必須擇一 |
| AI 模型 | 可切換 Claude / Gemini，預設 gemini（278） | 寫死 `claude`（313） | 需決定官網用哪個 |
| 重試機制 | **無** | 3 次 + 遞增退避（318-338） | 官網應統一採用有重試的版本 |
| 分頁 | 無 | 有 | 桌機版不需要可理解 |
| 特殊狀況提醒 | 列出**全部**勾選項（386） | 只列**指定 3 項**（240） | 需統一 |
| `max_tokens` | 未指定 | `2000`（313） | 需統一 |

**特別注意「無力」：** `index.html` 讓客人可以勾選「無力」，而 `mobile.html` 的 prompt 明文禁止 AI 輸出「無力」二字。若官網沿用禁語過濾（第 2 節），而客人勾了「無力」，AI 很可能會在回覆中複述這個詞而被攔截 —— 造成客人明明正常填寫卻一直拿到備援文案。**上線前務必先解決這個矛盾。**

### 建議做法

把共用邏輯抽成單一模組，桌機/手機只是不同的呈現層。然後寫一組共用行為測試同時跑在兩種呈現上 —— 任何差異都會以測試失敗的形式浮現，逼迫它變成一個明確的決定，而不是悄悄漂移。

---

## 6. P2 — 網路錯誤處理

`mobile.html:318-338` 的重試迴圈有幾條未被驗證的路徑：

1. **最後一次嘗試才遇到 503** —— 迴圈結束、`success` 為 false，顯示「伺服器流量過高」友善訊息。行為看起來正確但沒被驗證。
2. **非 503 錯誤** —— 立即 `break` 不重試。應確認這是刻意的。
3. **回應 `ok` 但 `data.result` 為 `undefined`** —— `data.result.trim()` 會在 `try` 內拋出 `TypeError`，被 catch 後 `break`，最後把 **`Cannot read properties of undefined`** 這種原始英文訊息顯示給中文客人看（346 行）。這是官網上很難看的失敗模式。

`index.html` 完全沒有重試（365-373 行單次 fetch），官網應統一採用有重試的版本。

```js
describe('錯誤處理', () => {
  it('連續 503 -> 顯示中文友善訊息，不顯示技術錯誤', async () => {
    mockWorkerResponse({ status: 503 }, { times: 3 });
    await analyzeSymptoms();
    const html = document.getElementById('ai-response').innerHTML;
    expect(html).toContain('流量過高');
    expect(html).not.toMatch(/undefined|TypeError|Cannot read/);
  });

  it('503 後成功 -> 應顯示正常結果', async () => {
    mockWorkerResponseSequence([{ status: 503 }, { result: '▍【肩頸】\n狀況：xxx' }]);
    await analyzeSymptoms();
    expect(document.getElementById('ai-response').innerHTML).toContain('肩頸');
  });

  it('回應 ok 但缺少 result 欄位 -> 不可顯示英文技術錯誤', async () => {
    mockWorkerResponse({ /* 沒有 result */ });
    await analyzeSymptoms();
    expect(document.getElementById('ai-response').innerHTML)
      .not.toMatch(/undefined|TypeError|Cannot read/);
  });
});
```

---

## 7. P3 — innerHTML 注入風險

以下位置都把未消毒的字串寫進 `innerHTML`：

- `index.html:413` — AI 輸出
- `index.html:401` — **錯誤訊息 `err.message`**
- `mobile.html:233` — AI 輸出
- `mobile.html:346` — 錯誤訊息

AI 輸出算半可信來源，但 `index.html` 會把客人自由輸入的文字（`user-extra` 第 324 行、`doctor-info` 第 323 行）送給模型，所以內容有機會被反射回來。官網若加入更多自由輸入欄位，風險會提高。

```js
it('AI 輸出中的標記應以純文字呈現，不得被解析為 HTML', async () => {
  mockWorkerResponse({ result: '▍【肩頸】\n狀況：<img src=x onerror=alert(1)>' });
  await analyzeSymptoms();
  expect(document.querySelectorAll('#ai-response img')).toHaveLength(0);
});
```

建議：改用 `textContent`，只針對明確需要的樣式（`**粗體**`、LINE 連結）做白名單轉換 —— 現有的 `renderFinalResponse`（`index.html:411-412`）已經是這個模式，把它推廣到所有渲染點即可。

---

## 8. 建議的測試環境與檔案結構

### 前提：需要先把邏輯抽出來

目前所有 JS 都內嵌在 HTML 的 `<script>` 裡，無法 import，也就無法直接測試。官網重構時建議：

```
src/
  assessment/
    banned-words.js      ← 禁語清單 + findBannedWords()
    safety-gate.js       ← 懷孕/骨折攔截判斷（純函式）
    split-pages.js       ← splitIntoPages()（純函式）
    api.js               ← fetch + 重試邏輯
    render.js            ← DOM 渲染
tests/
  banned-words.test.js
  safety-gate.test.js
  split-pages.test.js
  api.test.js
  integration.test.js    ← 用 jsdom 跑完整流程
```

抽成純函式後，前四項（禁語、攔截判斷、分頁、重試）都能用最單純的單元測試覆蓋，不需要 DOM。

### 建議工具

```bash
npm init -y
npm i -D vitest jsdom
```

`package.json`：

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest"
  }
}
```

`vitest.config.js`：

```js
import { defineConfig } from 'vitest/config';
export default defineConfig({
  test: { environment: 'jsdom' }
});
```

### 若不想動現有結構

替代方案：在測試裡用 jsdom 載入整份 HTML 檔，stub 掉 `fetch`，再直接呼叫全域的 `analyzeSymptoms()`。不需要改任何 production 程式碼，就能覆蓋第 2、3、6 節。缺點是測試較慢也較脆弱，但對兩頁的規模完全夠用。

再更簡單的選項：用 Playwright 直接對真實頁面跑端到端測試，只驗證安全攔截。程式碼一行都不用改。

---

## 9. 給接手者的提醒

1. **規模要相稱。** 這是小型商家網站，不需要追求高覆蓋率數字。把第 2、3 節做好（禁語 + 安全攔截），價值就已經達成八成。其餘視官網實際複雜度再決定。

2. **第 2 節（禁語攔截）跟「要不要寫測試」是兩件事。** 那是即時防線，不管測試策略如何都應該先做。

3. **不要把禁語過濾套用到業主自己寫死的文案上。** 詳見第 2 節的陷阱說明，「治療」「發炎」出現在合規聲明與攔截訊息裡是正常的。

4. **測試安全攔截時，值要從 DOM 讀出來，不要在測試裡重寫字串。** 否則改名時兩邊一起改，防護完全失效。

5. **第 5 節的歧異清單需要業主本人做決定**，特別是「無力」的矛盾。那是產品決策不是工程決策，不要自己選一個就實作下去。

6. **`WORKER_URL` 在兩個檔案裡都是寫死的明碼**（`index.html:274`、`mobile.html:158`）。官網階段建議改成設定檔或環境變數，方便日後切換測試/正式環境。

---

*本文件基於原型專案 `index.html` / `mobile.html` 的分析。文件中的 `splitIntoPages` 修正版與禁語過濾邏輯皆已實際執行驗證（13 項檢查全數通過）。*
