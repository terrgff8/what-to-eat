# 「要吃什麼」實作計畫書 v1.3

> **v1.3 變更（使用者 2026-07-04 指示）**：選項上限由 12 → **40**；預設改為 40 種料理；palette 由 12 → 20 色（`i % PALETTE.length` 循環，相鄰含首尾繞回皆異色）；字級表加大 N 級距；localStorage KEY 升 `v2`。下列各節「12」凡指「選項總數上限」者一律讀作 40；「名稱最長 12 字」不變。

> 本文件是唯一規格來源（single source of truth）。
> 給 AI 實作者（Sonnet 角色）與審查者（Opus 角色）閱讀執行。
> 所有技術決策已鎖定：**不得自行更換方案、不得新增本文件未寫的功能**。
> 規格有衝突或缺漏時：停下回報，不要自行猜測。

---

## 0. 一句話摘要

一頁式網頁「要吃什麼」：轉盤抽食物類別 → 彈出結果 → 按鈕開 Google Maps 搜「附近的○○」。零後端、零框架、零外部資源，單一 `index.html`，部署 GitHub Pages，PC 與手機瀏覽器都能用。

---

## 1. 範圍

### 1.1 要做
1. Canvas 繪製的轉盤 + 減速旋轉動畫。
2. 預設 5 個選項：`日式料理`、`燒烤`、`泰式`、`火鍋`、`便當`。
3. 選項可自訂新增/刪除（下限 2 個、上限 40 個），存 localStorage，重新整理不消失。
4. 抽中後顯示結果視窗（modal），內有「開地圖」按鈕 → 新分頁開 Google Maps 搜尋。
5. RWD：手機直式（最窄 375px，見 §4.4）到桌機都可正常操作。

### 1.2 不做（NEVER — 審查者看到即退回）
- 不做後端、API、帳號、資料庫。
- 不用 `navigator.geolocation` 抓 GPS（Google Maps 開啟後會自己用使用者定位排最近）。
- 不引入任何框架 / 函式庫 / CDN / Google Fonts / 外部圖片。頁面載入時，**除瀏覽器自動發出的 `favicon.ico` 外**，Network 不得有其他外部請求。為吸收 favicon 請求，`<head>` 內須放一個 `data:` URI 的 `<link rel="icon">`（例如 `data:,` 空 favicon 或內嵌 emoji SVG），使 Network 只剩 `index.html`。
- 不做店家清單、評分、抽選歷史、分享功能。
- 不用 `alert()` / `confirm()`（手機體驗差），一律用頁內訊息與 modal。

---

## 2. 已鎖定技術決策

| # | 項目 | 決策 | 理由 |
|---|------|------|------|
| D1 | 架構 | 單一 `index.html`，CSS/JS 全部內嵌 | 免 build、GitHub Pages 直接服務 |
| D2 | 轉盤繪製 | Canvas 2D 畫扇形與文字；旋轉靠整個 `<canvas>` 元素的 CSS `transform: rotate()` | 每幀只改 transform，不重繪 canvas，效能好且簡單 |
| D3 | 中獎邏輯 | **先隨機抽 winnerIndex，再反算目標角度**讓轉盤停在該扇形 | 避免「從最終角度反推中獎者」的對角 bug |
| D4 | 動畫 | `requestAnimationFrame` + easeOutCubic，時長 4000ms，基礎 5 圈 + delta（含 ±jitter，實際約 4.8~5.5 圈） | 不依賴 CSS transition 結束事件，好控制 |
| D5 | 開地圖時機 | 動畫結束**不直接** `window.open`；先彈結果 modal，由使用者**點按鈕**才開 | `window.open` 必須在使用者手勢的同步呼叫內，否則被 popup blocker 擋 |
| D6 | Maps URL | `https://www.google.com/maps/search/?api=1&query=` + `encodeURIComponent('附近的' + 選項名)` | Google 官方 Maps URL API；手機會自動喚起 Maps App；Maps 依使用者定位排最近 |
| D7 | 儲存 | localStorage，key = `whatToEat.items.v1`，值 = JSON 字串陣列 | |
| D8 | 部署 | GitHub Pages，`gh` CLI 建 repo + 開 Pages（見 §10） | |
| D9 | 語系 | `<html lang="zh-Hant">`，介面全繁體中文 | |

---

## 3. 檔案結構

```
what-to-eat/
├── index.html   ← 唯一程式產出物（實作者只准改這個檔）
└── PLAN.md      ← 本文件（實作者不得修改）
```

---

## 4. UI 規格

### 4.1 版面（由上到下、置中排列）

> 註：本節四個小節以 §4.1.1~§4.1.4 標號，供全案交叉引用。

**§4.1.1 標題區**：`🍽️ 要吃什麼？`（h1），副標 `選擇困難救星`（小字灰色）。

**§4.1.2 轉盤區**：
   - 指針：固定在轉盤正上方（12 點鐘）、朝下的三角形（DOM 元素，不隨轉盤轉動），z-index 高於轉盤。
   - **指針三角形寫死 spec**：純 CSS border 三角形——`width:0; height:0; border-left:12px solid transparent; border-right:12px solid transparent; border-top:18px solid #FF6B35;`（尖端朝下），置於轉盤正上緣、水平置中、尖端壓在圓周上緣（`margin-bottom:-4px` 使尖端略入圓內）。顏色 `#FF6B35`（主色）。
   - 轉盤：正方形 canvas（正圓由繪製填滿），邊長 `size` 由 JS 計算（見下），CSS 寬高由 `size` 決定。
   - **`size` 唯一計算式（實作者照抄，§5.3 DPR 與此共用同一 `size`）**：
     ```js
     const size = Math.min(Math.round(window.innerWidth * 0.9), 400); // px
     ```
     首次於 `DOMContentLoaded` 求值，`resize` debounce 後重算（見 §6.5）。
   - canvas 幾何中心對位：`transform-origin: center;`（或不設，預設即 50% 50%）；canvas **不得**有會位移幾何中心的 `border` / 非對稱 `margin` / `padding`（否則旋轉軸偏移、指針指錯扇形）。置中用外層容器處理。

**§4.1.3 轉動按鈕**：`🎲 轉！`，寬 ≥ 200px、高 ≥ 52px、字 ≥ 20px。

**§4.1.4 編輯區**：`<details id="editPanel">` 摺疊面板（`#editPanel` 掛在 `<details>` 本身），summary 顯示 `⚙️ 編輯選項（N 個）`，**N 隨選項數即時更新**（v1.3 初始 40 項即顯示「⚙️ 編輯選項（40 個）」），內含：
   - 選項列表：每列 = 選項名 + `✕` 刪除鈕（刪除鈕觸控面積 ≥ 44×44px）。
   - 新增列：文字輸入框（placeholder `新增選項，例如：拉麵`）+ `新增` 按鈕。
   - `恢復預設` 按鈕（次要樣式）。
   - 訊息列 `#msg`：顯示驗證錯誤（紅字，3 秒後自動清除）。

### 4.2 固定元素 id（實作者必須照用，供測試與審查對照）
`#wheel`（canvas）、`#pointer`、`#spinBtn`、`#resultModal`、`#resultName`、`#mapsBtn`、`#againBtn`、`#editPanel`、`#itemList`、`#newItemInput`、`#addBtn`、`#resetBtn`、`#msg`

### 4.3 配色
- 背景 `#FFF8F0`（暖米白）、主文字 `#333`、主按鈕底 `#FF6B35` 白字、圓角 12px。
- 轉盤扇形循環用以下 12 色（全部可配白色粗體字）：
  ```
  ['#FF6B6B','#F08C00','#40C057','#22B8CF','#4D8AF0','#9775FA',
   '#F06595','#12B886','#E8590C','#7048E8','#E64980','#3B5BDB']
  ```
  v1.3 起 palette 擴為 20 色（見 index.html），第 i 個扇形用 `palette[i % PALETTE.length]`；選項上限 40 時顏色循環重複，但相鄰扇形（含第 40 與第 1 繞回）皆不同色。

### 4.4 RWD 要求
- `<meta name="viewport" content="width=device-width, initial-scale=1">` 必加。
- **最窄支援寬度 = 375px**（全案基準，與 AC-9 同）：無水平捲軸、轉盤完整可見、所有按鈕可點且高 ≥ 44px、編輯區列表 + 刪除鈕不擠出水平捲軸。
- 結果 modal：置中卡片、寬 `min(85vw, 340px)`、背後半透明遮罩。

---

## 5. 功能規格（含關鍵程式碼，實作者可直接採用）

### 5.1 狀態

```js
const KEY = 'whatToEat.items.v2';                 // v1.3：v1→v2 讓新 40 種預設生效
const MAX = 40;                                    // 選項總數上限
const DEFAULTS = [ /* 40 種，見 index.html；前 5 = 日式料理/燒烤/泰式/火鍋/便當 */ ];
const state = {
  items: loadItems(),   // string[]
  rotation: 0,          // 累積旋轉角度（度），只增不減
  spinning: false,
};
```

### 5.2 讀取 / 儲存（必須防壞資料）

```js
function loadItems() {
  try {
    const arr = JSON.parse(localStorage.getItem(KEY));
    if (Array.isArray(arr)) {
      const clean = arr
        .filter(x => typeof x === 'string' && x.trim().length > 0)
        .map(x => x.trim())
        .slice(0, 12);
      if (clean.length >= 2) return clean;
    }
  } catch (e) { /* 壞資料 → 落到預設 */ }
  return [...DEFAULTS];
}
function saveItems() {
  try { localStorage.setItem(KEY, JSON.stringify(state.items)); } catch (e) {}
}
```

### 5.3 轉盤繪製（角度約定：**12 點鐘 = 0 度、順時針為正**，全案統一）

- 第 `i` 個扇形（0-based）：從「12 點鐘順時針偏移 `i * seg` 度」畫到 `（i+1) * seg` 度，`seg = 360 / N`。
- Canvas 座標系 0 弧度在 3 點鐘，所以畫弧時起角 = `(-90 + i*seg) * Math.PI/180`。
- 高解析度處理（必做，否則手機模糊）：

```js
const dpr = window.devicePixelRatio || 1;
canvas.width  = size * dpr;
canvas.height = size * dpr;
canvas.style.width  = size + 'px';
canvas.style.height = size + 'px';
ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
```

- 扇形文字：沿半徑放（radial）。作法：`ctx.translate(中心)` → `ctx.rotate((-90 + i*seg + seg/2) * Math.PI/180)` → `textAlign='right'`、`textBaseline='middle'` → `fillText(繪製字串, 半徑*0.88, 0)`。左半邊文字會倒向圓心，屬正常轉盤樣式，可接受。
- **字級表（依 N 決定，寫死不得自創）**：
  | N（選項數） | 字級 |
  |---|---|
  | 2–6 | `bold 16px` |
  | 7–8 | `bold 15px` |
  | 9–12 | `bold 13px` |
  | 13–20 | `bold 12px` |
  | 21–40 | `bold 11px` |
  字型固定 `system-ui, sans-serif`，文字一律白色。
- **長名稱截斷（寫死）**：繪製字串 = 名稱以「code point」計（`[...name]`）超過 8 個字元時，取前 7 個 code point + `…`（省略號）。此截斷**只作用於轉盤繪製**；結果 modal 與 Maps 查詢一律用完整名稱。此規則使最長顯示 8 字（含省略號），在 0.88 半徑內不溢出。
- 中心畫一個半徑 ~8% 的白色圓蓋（裝飾）。

### 5.4 轉動與中獎（核心，公式照抄）

```js
function spin() {
  if (state.spinning || state.items.length < 2) return;
  state.spinning = true;
  spinBtn.disabled = true;

  const n   = state.items.length;
  const seg = 360 / n;
  const winnerIndex = Math.floor(Math.random() * n);      // 先決定贏家
  const centerOffset = winnerIndex * seg + seg / 2;       // 該扇形中心相對 12 點鐘的順時針角
  const jitter = (Math.random() - 0.5) * seg * 0.7;       // ±0.35 seg，保證不壓線
  const current = state.rotation;
  // delta：讓「扇形中心」轉到 12 點鐘所需的增量（0~360）
  const delta = ((360 - centerOffset) - (current % 360) + 720) % 360;
  const target = current + 5 * 360 + delta + jitter;

  animate(current, target, 4000, () => {
    state.rotation = target;
    state.spinning = false;
    spinBtn.disabled = false;
    showResult(state.items[winnerIndex], winnerIndex);
  });
}

function animate(from, to, dur, done) {
  const t0 = performance.now();
  (function frame(now) {
    const t = Math.min((now - t0) / dur, 1);
    const p = 1 - Math.pow(1 - t, 3);                     // easeOutCubic
    canvas.style.transform = `rotate(${from + (to - from) * p}deg)`;
    if (t < 1) requestAnimationFrame(frame); else done();
  })(performance.now());
}
```

**驗算範例（審查者必核對）**：N=5、seg=72、winnerIndex=2（第 3 項「泰式」）、current=0、jitter=0
→ centerOffset = 2×72+36 = 180 → delta = (360−180−0+720)%360 = 180 → target = 1800+180 = **1980**。
轉 1980 度後，扇形 2 的中心（原本在順時針 180 度）位於 (180+1980)%360 = 0 度 = 12 點鐘指針處 ✅

### 5.5 結果 modal

- 內容：小字 `今天吃` + 大字選項名（`#resultName`，≥ 28px 粗體）+ 兩顆按鈕：
  - 選項名寫入 `#resultName` **一律用 `textContent`（禁用 `innerHTML`）**，避免含 `<`、`&` 的名稱造成顯示錯亂或注入。
  - `showResult(item, winnerIndex)` 同時設定 `resultName.dataset.idx = winnerIndex`（供 AC-2 程式化斷言「名稱 = `state.items[data-idx]`」）。
  - `#mapsBtn`：`🗺️ 找最近的○○`（主按鈕）→ 點擊時**同步**執行（`item` = 完整名稱，非截斷後）：
    ```js
    window.open(
      'https://www.google.com/maps/search/?api=1&query=' +
        encodeURIComponent('附近的' + item),
      '_blank', 'noopener');
    ```
  - `#againBtn`：`🔄 再轉一次` → 關閉 modal 後呼叫 `spin()`。
- 點半透明遮罩也可關閉 modal。
- modal 開啟期間 `#spinBtn` 與編輯區操作無效（modal 蓋住即可）。

### 5.6 選項管理

**字數計法（全案統一）**：名稱長度一律以 code point 計，即 `[...name].length`。單一 emoji（surrogate pair）算 1，避免 `.length` 把 emoji 誤判超長。§5.3 截斷、下表「≤ 12 字」、AC-6 測資皆用此定義。

| 操作 | 規則 | 違規時 `#msg` 顯示 |
|------|------|--------------------|
| 新增 | `trim()` 後非空 | `請輸入名稱` |
| 新增 | 不得與現有重複 | `「○○」已存在` |
| 新增 | `[...name].length ≤ 12` | `名稱最長 12 字` |
| 新增 | 總數 ≤ 40 | `最多 40 個選項` |
| 刪除 | 總數 > 2 才可刪；**剩 2 個時 `#itemList` 內所有刪除鈕加 `disabled` 屬性** | —（鈕直接失效） |
| 恢復預設 | `state.items = [...DEFAULTS]` | — |

- **刪除 UX（寫死）**：單擊刪除鈕立即生效、**無確認對話框**（§1.2 禁 confirm）、**不顯示 `#msg` 訊息**；刪除不影響 `#newItemInput` 內既有文字。
- 任何變更後：`saveItems()` → 重繪轉盤 → 更新列表與 summary 數字。`state.rotation` 保留不歸零。
- `state.spinning === true` 時：新增/刪除/恢復預設一律不動作（直接 return）。
- 新增成功後清空輸入框；輸入框按 Enter 等同按「新增」。
- 列表內顯示選項名一律用 `textContent`（同 §5.5，禁 `innerHTML`）。

---

## 6. 邊界與錯誤處理清單

> 註：本節條目以 §6.1~§6.5 標號，供全案交叉引用。

**§6.1** localStorage 值是壞 JSON / 不是陣列 / 少於 2 個有效字串 → 回預設 5 項，**不得拋出未捕捉例外**。

**§6.2** localStorage 不可用（隱私模式）→ `saveItems` 的 try/catch 吞掉，功能照常（只是不持久）。

**§6.3** 連點「轉！」→ `spinning` flag 擋掉，只轉一次。

**§6.4** 選項名含特殊字元（空格、`&`、`<`、emoji）→ 顯示用 `textContent`、URL 用 `encodeURIComponent`，兩路都正確。

**§6.5** 視窗縮放（resize / 手機轉向）→ debounce 200ms 後重算 `size`（§4.1.2 公式）並重繪 canvas。**動畫進行中（`state.spinning === true`）時：resize 只記錄、不重繪，延到動畫結束再套用新尺寸**，避免打斷 CSS transform 旋轉。參考實作：

```js
let resizeTimer, pendingResize = false;
window.addEventListener('resize', () => {
  clearTimeout(resizeTimer);
  resizeTimer = setTimeout(() => {
    if (state.spinning) { pendingResize = true; return; } // 動畫中先掛旗標
    drawWheel();                                          // 內部重算 size + DPR + 重繪
  }, 200);
});
// animate 的 done callback 尾端補：if (pendingResize) { pendingResize = false; drawWheel(); }
```

---

## 7. 驗收標準（AC — 全數通過才算完成）

| # | 驗收項 | 驗證方法 |
|---|--------|----------|
| AC-1 | 首次開啟：轉盤呈現 40 個扇形、內容與 DEFAULTS 一致且 40 種不重複、相鄰扇形（含首尾繞回）異色 | 讀 `state.items`（數量=40、`new Set` 去重）；顏色判準 = 填色用 `palette[i % PALETTE.length]`，逐一驗 `palette[i%L] !== palette[((i+1)%40)%L]`（讀常數，**不從 canvas 截圖取色**） |
| AC-2 | 按「轉！」：減速動畫後停止，結果 modal 名稱 = `state.items[winnerIndex]`（**程式化驗證**，非目視旋轉截圖） | ① showResult 把 winnerIndex 寫入 `#resultName` 的 `data-idx`；斷言 `#resultName.textContent === state.items[Number(#resultName.dataset.idx)]`。② 動畫時長判準 = 讀程式碼 `animate(...)` 傳入 `dur === 4000` 且用 easeOutCubic（**不碼錶量牆鐘時間**，避免 rAF 節流噪音）。③ 用 §5.4 驗算範例數學核對公式。連測 3 次 |
| AC-3 | 按「找最近的○○」：新分頁網址 = `https://www.google.com/maps/search/?api=1&query=` + encodeURIComponent(`附近的` + 完整名稱) | 攔截 `window.open` 讀第 1 引數 |
| AC-4 | 未達上限時新增有效並持久：刪 1 項（→39）後新增「宵夜」→ 40 段；reload 後仍含「宵夜」且 40 段 | 操作 + reload，讀 `state.items.length` 與內容 |
| AC-5 | 刪到剩 2 個 → `#itemList` 內所有刪除鈕 `disabled` | 操作，查 disabled 屬性 |
| AC-6 | 四種輸入各被拒且 `#msg` 顯示指定文字：① 空白→`請輸入名稱`；② 重複既有項→`「○○」已存在`；③ 13 個 code point 的名稱→`名稱最長 12 字`；④ 已達上限（40 項）時再新增→`最多 40 個選項` | 逐項操作，斷言 `#msg.textContent` |
| AC-7 | 動畫中連點「轉！」→ 只轉一次；動畫中新增/刪除無效 | 操作 |
| AC-8 | 執行 `localStorage.setItem('whatToEat.items.v1','not json')` 後重整 → 回預設 5 項，console 無未捕捉錯誤 | DevTools / console_logs |
| AC-9 | 375×667 視窗：無水平捲軸；canvas 未被裁切或被其他元素遮蓋（**允許頁面垂直捲動**，轉盤不必全落在首屏 667px 內）；`#spinBtn` 與刪除鈕高 ≥ 44px | preview_resize：查 `document.documentElement.scrollWidth <= 375`、canvas `getBoundingClientRect` 完整、按鈕 `offsetHeight >= 44` |
| AC-10 | 頁面載入：Network 排除瀏覽器自動 `favicon.ico` 後，只有 `index.html` 一個請求（`<head>` 已內嵌 data: favicon 吸收之） | DevTools / preview_network |

---

## 8. 實作步驟（給實作者 Sonnet，依序做，每階段自驗後才進下一階段）

- **Phase 1 — 骨架與轉盤**：HTML 結構（§4.2 全部 id）、CSS、canvas 繪製（含 DPR、resize 重繪）。
  結束條件：AC-1、AC-9、AC-10 通過。
- **Phase 2 — 轉動與地圖**：spin/animate（§5.4 照抄）、結果 modal、Maps 按鈕（§5.5）。
  結束條件：AC-2、AC-3、AC-7 通過。
- **Phase 3 — 選項管理**：CRUD + 驗證（§5.6）、localStorage（§5.2）。
  結束條件：AC-4、AC-5、AC-6、AC-8 通過。
- **Phase 4 — 總驗收**：AC-1 ~ AC-10 全跑一遍，修到全過。

實作者鐵則：
1. 只准建立/修改 `index.html`。
2. §5.4 公式與 §5.3 角度約定**逐字採用**，不得改寫成別的角度系統。
3. 發現規格衝突/缺漏 → 停下，輸出「規格問題：…」回報，不自行決定。
4. 不加規格外功能（~~音效~~、震動、深色模式、動畫花樣）。**例外（使用者 2026-07-04 override）：金屬音效已核准** — Web Audio 合成（零外部音檔）：轉動每過扇形邊界播金屬 tick、落定播金屬鐘鳴；`#soundBtn` 開關存 localStorage `whatToEat.sound.v1`。震動/深色模式/動畫花樣仍禁。

## 9. 審查清單（給審查者 Opus，逐條打勾）

- [ ] 角度系統一致：繪圖（§5.3）與中獎（§5.4）都以 12 點鐘為 0 度、順時針為正；用 §5.4 驗算範例核對程式碼。
- [ ] **視覺對位（以公式為準，非像素目視）**：canvas 為正方形、`transform-origin` 為 center、無位移幾何中心的 border/padding/非對稱 margin。pass 判準 = 上述結構條件 + §5.4 公式逐字符合（`-90` 起角、delta 公式、jitter ≤ ±0.35 seg）。截圖僅供 sanity check，**不以肉眼「差幾度」當 fail 依據**（rAF 停格與三角形尖端定義有天然誤差）。
- [ ] `window.open` 只出現在 `#mapsBtn` 的 click handler 內、同步呼叫（D5）。
- [ ] Maps URL 完全符合 D6（含 `?api=1` 與 `encodeURIComponent`）、且用完整名稱（非 §5.3 截斷後）。
- [ ] 顯示選項名（`#resultName`、`#itemList`）一律 `textContent`，無 `innerHTML` 拼字串。
- [ ] **零外部資源（語意判準，非純字串搜 http）**：無 `<link href=`、`<script src=`、`<img src=` 指向非 `data:` 來源；除 `#mapsBtn` handler 內的 Maps URL 外，全檔無 `http(s)://` 外部 URL；favicon 為 `data:` URI。
- [ ] `loadItems` 有 try/catch 且驗證陣列內容；`saveItems` 有 try/catch。
- [ ] `spinning` flag 防重入；動畫中編輯無效；動畫中 resize 不打斷（§6.5）。
- [ ] canvas 有 DPR 縮放與 resize 重繪；`size` 用 §4.1.2 唯一公式。
- [ ] 字級表（§5.3）與長名稱截斷（≤8 code point 顯示）已實作；字數一律 `[...s].length` 計。
- [ ] 全部 §4.2 的 id 存在且拼字正確；`#editPanel` 掛在 `<details>` 本身。
- [ ] 無 §1.2 禁止事項；無規格外功能 → 有就退回重做。**「規格外功能」硬退回定義（窮舉，只此兩類）**：① §1.2 明列禁項（後端/geolocation/CDN/外部資源/店家清單/評分/歷史/分享/alert/confirm）；② §8 鐵則 4 明列（音效/震動/深色模式/額外動畫花樣）。**可接受不退回的無害增強**（白名單）：`aria-label`/`role` 等無障礙屬性、`:focus` 樣式、`:hover` 效果、陰影/圓角等純裝飾 CSS、鍵盤操作（Enter 已在規格內）。判不準時歸「可接受」，不退回。
- [ ] AC-1 ~ AC-10 逐條有實測證據（screenshot / console / snapshot / network）。

## 10. 部署步驟（**等使用者說「部署」才執行**）

前提：`gh` CLI 已登入（`gh auth status` 確認）。

```bash
cd C:/Users/Trans-Am/.claude/FreeCase1/what-to-eat
git init -b main
git add index.html PLAN.md
git commit -m "what-to-eat: wheel food picker MVP"
gh repo create what-to-eat --public --source . --push
OWNER=$(gh api user -q .login)
# 注意：source[branch] 是巢狀欄位，必須用 -F（field），不能用 -f（raw string）。
# 用 -f 會送成扁平鍵 {"source[branch]":"main"}，Pages API 回 422 啟用失敗。實測 gh 2.89。
gh api -X POST "repos/$OWNER/what-to-eat/pages" -F "source[branch]=main" -F "source[path]=/"
```

驗證：等 1~3 分鐘，`https://<OWNER>.github.io/what-to-eat/` 回 HTTP 200 且轉盤可轉。
若 Pages API 回 409（已存在）視為成功，改用 `gh api repos/$OWNER/what-to-eat/pages` 查狀態。

## 11. 常見陷阱（實作者先讀完再動工）

1. **popup blocker**：動畫 callback 裡直接 `window.open` 必被擋 → 一律走 modal 按鈕（D5）。
2. **canvas 模糊**：忘記 DPR 縮放，手機上字糊 → §5.3 程式碼必用。
3. **對角 bug**：自創「從最終角度反推中獎者」公式最容易錯 → 禁止，照 D3/§5.4。
4. **jitter 壓線**：jitter 超過 ±0.5 seg 會讓指針壓在扇形交界 → 上限 ±0.35 seg 已寫死。
5. **中文 URL**：不 encode 會在部分瀏覽器產生錯誤查詢 → 必用 `encodeURIComponent`。
6. **rotation 取模**：`current % 360` 在 current 為累積正值時安全；不要讓 rotation 出現負值。
7. **iOS `<details>`**：iOS Safari 支援良好，可放心用；不需 polyfill。

## 12. 完成定義（loop 退出條件）

`index.html` 存在、AC-1 ~ AC-10 全數通過、審查清單 §9 全勾、無 blocker → **DONE**。
