# Junba 維護 DISC 人際風格測驗：維護與交接說明

給未來接手的人或其他 AI 模型：先讀完本文件，再動手改 `index.html`。
最後更新：2026-11-25（第三次更新）

## 0. 修改原則（務必遵守）
1. **只新增、不更動既有功能**。除非使用者明確要求，不要改動計分、代碼編碼、周哈里窗、PDF、他評流程、Google 試算表欄位順序。
2. 每次修改後都要重跑第 9 節的驗證清單，確認舊功能沒壞。
3. `index.html` 維持單一檔案、無建置步驟；後端欄位順序若要動，必須同步改前端與 Apps Script。
4. 不要把主持人密碼、使用者聯絡資料寫進程式、文件或對話。

## 1. 專案目的
繁體中文的 DISC 人際風格測驗網頁，流程共三步：
1. 自評：5 組描述，每組把 1、3、5、7 各用一次。
2. 請重要它人評價：同樣格式，可多位，取平均。
3. 比對：自評 vs 他評，並用周哈里窗（開放我、盲目我、隱藏我、未知我）分析。

另有：變色龍型判定、回顧與 PDF（瀏覽器列印）、Google 試算表收集課堂資料、主持人統計頁。

## 2. 架構與位置
| 項目 | 內容 |
|---|---|
| 前端 | 單一檔案 `index.html`（原生 JS，無框架，無建置步驟） |
| 託管 | GitHub Pages，儲存庫 `b52tw/junba-d.i.s.c-test`，main 分支，根目錄 |
| 網址 | 以 Pages 設定頁顯示的網址為準 |
| 後端 | Google Apps Script 網頁應用程式 + Google 試算表（自評、他評、統計 三個分頁） |
| Apps Script 網址 | 程式內常數 `API`（`.../exec`） |
| 外部程式庫 | 只有 QR 產生：cdnjs `qrcode-generator@1.4.4`，載入失敗時頁面會退回顯示文字連結 |
| 原始製作 | Claude 網頁版 artifact（私人連結，其他人打不開，不要當成來源） |

注意：
- 原本的 Claude 連線對該 GitHub 儲存庫只有讀取權限，更新要由使用者手動到 GitHub 上傳覆蓋 `index.html`。
- 主持人密碼 `HOST_KEY` 只存在使用者的 Apps Script 裡，本文件沒有，也不應該被寫進前端程式。

## 3. 計分規則（不要隨意改，會連動多處）
- 每組 4 個描述，順序固定為 D、I、S、C（老虎、孔雀、無尾熊、貓頭鷹），分數 1、3、5、7 各用一次。
- 每欄加總為該類型分數，範圍 5–35，四項總和固定 80；每項平均 20。
- 因為 5 個奇數相加必為奇數，單人分數永遠不會等於 20，所以周哈里窗用「>20」判斷「明顯」沒有平手問題。
- 主型：最高分；同分則並列。副型：第二高且與最高差 ≤4。
- 變色龍型：最高與最低相差 ≤8（此門檻為作者自訂）。
- 分數文字說明：7 最像、5 大部分有像、3 只有部分像、1 最不像。
  - 使用者曾寫成「2 分」，但按鈕是 1/3/5/7，目前文字採 3。若要真的改成 1、2、5、7，要同步改：計分範圍與總和、長條圖分母 35、變色龍門檻、周哈里窗門檻、代碼編碼（見下）。

## 4. 他評代碼（離線備援）
- 一個 grid（5 組）編成 6 個字母：前 5 個字母各代表該組 [1,3,5,7] 的排列（24 種），字母表 `ABCDEFGHJKLMNPQRSTUVWXYZ`（去掉 I、O）；第 6 個字母是檢查碼（5 個索引總和 mod 24）。
- 函式：`enc(grid)` 編碼、`decg(code)` 還原 grid、`dec(code)` 還原四項分數。
- 代碼可還原完整 5 組作答，所以他評的「5 組對照表」是從代碼還原，不另存。

## 5. 程式結構（`index.html` 內的 script）
- 資料常數：`T`（四種類型）、`W`（5×4 描述文字）、`TIP`（類型相處建議 4×4）、`CHAM`（變色龍說明）、`JO`（周哈里窗文字）。
- 狀態：全域 `st`，主要欄位 `s`（目前畫面）、`self`、`gs`（自評 grid）、`go`（目前評價 grid）、`others`（他評分數陣列）、`og`（他評 grid 陣列）、`codes`、`prof`（課堂資料模式與欄位）、`id`（參與者編號）、`sent`、`opt`（第一頁是否已按「確定」）、`sel`（第一頁目前選的選項）、`words`（別人給你的一句話）、`hist`（歷次自評紀錄，最多 12 筆）、`tmeta`（本次作答時間摘要）、`tm`／`nc`／`tl`（作答計時用的暫存）。
- 畫面 `st.s`：`intro`、`self`、`selfRes`、`invite`、`other`、`otherRes`、`cmp`、`host`（主持人）、`rintro`、`rmsg`（評價者的留言與滿意度，只在 `?r=` 模式出現）、`rres`（給評價者）。
- 渲染：`render()` 以 innerHTML 重畫整頁；事件用單一 `click` 委派，按鈕以 `data-a`（動作）、`data-g/r/c/v`（作答）區分。
- `BACK` 對照表控制「← 回上一頁」。
- 儲存：`localStorage` 鍵 `disc-v1`（自評、作答、他評、課堂資料設定、編號）；評價者模式不儲存。重新開啟網頁時**一律從第一頁開始**；若瀏覽器有舊紀錄，第一頁上方出現「繼續上次進度／從頭開始（清除紀錄）」卡片（`st.resume`、動作 `resume` 與 `fresh`），不會自動跳到後面的步驟。
- 模式判斷（網址）：
  - `?r=編號&c=課堂代號`：評價者模式，完成後自動送到雲端並顯示代碼。
  - `#rater`：評價者模式（未選上傳時用代碼回傳）。
  - `?host=1&c=課堂代號`：主持人頁。
  - `?c=課堂代號`：學員報名，自動帶入課堂代號，並略過「輸入課堂代號」頁。
- PDF：用 `window.print()`，列印樣式 `@media print` 會隱藏按鈕並加報告標題（含日期與名字）。

## 5-1. 解析引擎（自評結果頁與比對頁的深入解析）
資料來源：類型說明（標語、描述、優勢、留意處、壓力反應、成長方向、職涯方向）與六種組合型名稱，取自使用者自己的另一個專案 `b52tw/DISC-Career`（`index.html` 內的 `TYPES`、`PAIRS`），放在本檔的 `X`、`PR`；相處對照（`PI`）、微練習（`MICRO`）、作答節奏說明（`TX`）、周哈里窗正反向說明（`JQ`）是本專案新寫的。

函式與用途：
- `rich()`：自評結果頁的「你的深入解析」「作答節奏」「與上次比較」「這次給你的建議」「可以探索的方向」。內容依實際資料產生：組合型（最高與第二高相差 ≤6 才顯示）、分數強度（`band()`）、你給 7 分與 1 分的特質原文、7 分是否集中在主型、跨類型的 7 分。
- `timeCard(m)`：作答時間分析。總秒數分級（`tband`：<45 很快、<90 偏快、<180 中等、<300 偏慢、其餘很慢），再依主型與快慢給不同詮釋；單組停留 ≥15 秒且 ≥ 中位數 1.8 倍時指出「最難取捨」的一組；修改次數（多餘點擊數）≥4 視為反覆調整、≤1 視為篤定。以上門檻是作者暫訂的經驗值，需用實際資料校準，只當線索不是診斷。
- `rel(a)`：與四種類型的相處。每種類型含：方向性相處建議（`TIP`）、強連結、摩擦點、橋接做法（`PI`，依類型關係分同型、相鄰、對立）、依自己該類型分數給的「你的條件」；依自己最低分的類型排序，並標示「你最需要練習」。
- `joEnrich()`：周哈里窗深入解讀（比對頁）。以 20 個特質格子計算平均差距，一致度 % = 100×(1−平均差/6)，≥80 強、≥65 中、其餘弱；各象限（用與 `cmp()` 相同的 >20 門檻）附正向與反向解讀；列出強連結（雙方都≥5 且差≤1.5）、弱連結（差≥2.5，分「別人看得比你多」與「你自認有、別人較少看到」）；兩位以上評價者時計算評價者之間一致度；依結果給展現、請教、招牌三種行動建議。
- 變化與不重複：輸出是「同樣資料同樣結果」（以分數、作答時間、紀錄次數做雜湊決定要挑哪幾條），資料不同、作答節奏不同、或第幾次測驗不同，就會挑到不同的條目；建議與微練習會隨紀錄次數輪替。

計時與紀錄：
- 只計「自評」格子（`data-g='gs'`）的點擊。兩次點擊間隔超過 60 秒以 60 秒計，避免離開頁面被算進去；`start` 與回到自評頁時重新計時。
- 按「計算我的結果」時產生 `st.tmeta = {sec, rows[5], rev}`，並寫入 `st.hist`；分數與上一筆相同時取代而不重複新增。
- 「重新開始」保留 `hist`（供下次比較）；「從頭開始／清除紀錄」才清除。第一頁在有紀錄時會顯示對應卡片。
- 計時與紀錄只存在該裝置的瀏覽器，目前不會上傳到試算表。

## 6. 進入流程與課堂資料三種模式（`st.prof.mode`）
**進入流程（兩層：課堂代號 → 第一頁 → 第二頁）**：
1. 沒有課堂代號（`st.prof.cls` 為空，且網址沒有 `?c=`）時，只顯示「先輸入課堂代號」與「我是主持人」按鈕。
2. 有課堂代號後，**第一頁只顯示「課堂資料（選填）」**：課堂代號與「更換」、三個選項（預設都不選）。選了一項後才出現「姓名或代號」；選項 3 另外出現 Email、電話、LINE ID；選項 2、3 出現同意勾選；最後按「確定，下一步」（`ok`）。
3. 按「確定」後，第一頁縮成一列摘要（課堂代號、資料方式、名字，旁邊有「修改」按鈕 `chgopt`），並顯示第二頁內容：測驗說明與「開始自評」。
4. 驗證：沒選項會被擋；選項 3、或選項 2 且有填名字，必須勾同意，失敗時已填的內容與選擇會保留。
5. 評價者模式（`?r=`、`#rater`）與主持人頁不受此流程影響。
6. 目前代號只是分組標籤，沒有白名單驗證。
7. 所有輸入框外都有文字標籤；說明文字都註明「每一個詞都是一項特質」。

**三種模式**：
| 值 | 畫面文字 | 行為 |
|---|---|---|
| 0 | 1. 留名字／匿名（只在裝置上使用，不上傳） | 不送任何資料；名字只用在 PDF 標題 |
| 1 | 2. 用名字／匿名（供課堂統計，只上傳分數） | 按「送出」才上傳分數與名字（可空白＝匿名）；填了名字須勾同意 |
| 2 | 3. 提供姓名（留下聯絡資料，可進一步得到聯絡分析機會） | 另外上傳 Email、電話、LINE ID（皆選填）；須勾同意 |
- 第一頁不預選任何項目。送出動作 `send` 由使用者按鈕觸發，不會自動上傳。選項 1（mode 0）不使用雲端，因此沒有「收取網路上的評價」，評價者回傳走代碼。

## 7. Google 試算表與 Apps Script
工作表：
- `自評` 欄位：時間、課堂代號、編號、姓名或代號、Email、電話、LINE ID、D、I、S、C、滿意度、建議
- `他評` 欄位：時間、課堂代號、被評者編號、D、I、S、C、作答代碼、滿意度、一句話（滿意度與一句話是第 9、10 欄，舊版 8 欄的表會在下次寫入時自動補上標題）
- `統計`：每個課堂代號一列（人數與四項平均，公式指向該班分頁），第一次出現新代號時自動新增。
- `班-<課堂代號>`：**每個課堂代號自動一個分頁**（同一份試算表）。內容：自評資料列（含主型、滿意度、建議）、他評人數與平均、他評主型、他評滿意度、他評的一句話；右側 W:Z 欄是統計區（自評主型人數、自評與他評平均、人數、平均滿意度）與兩張圖表（圓餅圖：自評主型分布；直條圖：自評與他評平均）。分頁每次有新資料就整頁重寫，**請不要在這些分頁手動修改資料**，要修改請改「自評」「他評」總表再執行 `rebuildAll`。
- 課堂代號含 `[ ] * ? : \ / '` 時，分頁名稱會把這些字元換成 `_`。

API：
- `POST`（前端用 `fetch(..., {mode:'no-cors'})` 送 JSON 文字）：
  - `{type:'self', cls, id, name, email, phone, line, score:[D,I,S,C], rating, note}`
  - `{type:'other', cls, id, score:[D,I,S,C], code, rating, msg}`（`msg` 前端限 100 字）
- `GET ?a=others&id=編號`：回傳該編號收到的他評陣列 `[{score, code, rating, msg}]`；前端把 `msg` 收成「別人給你的話」。
- `GET ?a=host&key=密碼&cls=課堂代號`：回傳 `{n, avg:[4], main:[4], rate, rows:[{id,name,score:[4],rating,note}], others:[{id,n,avg:[4]}]}`；密碼錯誤回 `{error:'denied'}`。**不含 Email、電話、LINE ID**。
  - 前端 `ext(j)` 會用這些欄位顯示「自評名單」「他評名單」「建議與回饋」；若伺服器是舊版（沒有 rows/others），前端仍可只顯示統計卡片，不會出錯。
  - 更新 Apps Script 後必須重新部署新版本才會生效。

Apps Script 程式碼（密碼請使用者自行設定，勿寫進前端）：

```js
const HOST_KEY = '請改成你自己的主持人密碼';
const H_SELF = ['時間','課堂代號','編號','姓名或代號','Email','電話','LINE ID','D','I','S','C','滿意度','建議'];
const H_OTHER = ['時間','課堂代號','被評者編號','D','I','S','C','作答代碼','滿意度','一句話'];
const H_CLASS = ['時間','編號','姓名或代號','Email','電話','LINE ID','D','I','S','C','主型','滿意度','建議',
  '他評人數','他評D','他評I','他評S','他評C','他評主型','他評滿意度','他評的話'];
const TYPES = ['D','I','S','C'];

function sh_(name, head) {
  const ss = SpreadsheetApp.getActive();
  let s = ss.getSheetByName(name);
  if (!s) { s = ss.insertSheet(name); s.appendRow(head); s.setFrozenRows(1); }
  else if (s.getLastColumn() < head.length) s.getRange(1, 1, 1, head.length).setValues([head]);
  return s;
}
function cname_(cls) { return ('班-' + String(cls).replace(/[\[\]*?:\\\/']/g, '_')).slice(0, 90); }
function main_(sc) { return TYPES[sc.indexOf(Math.max.apply(null, sc))]; }
function round1_(x) { return Math.round(x * 10) / 10; }

function setup() {
  sh_('自評', H_SELF); sh_('他評', H_OTHER);
  sh_('統計', ['課堂代號','人數','D平均','I平均','S平均','C平均']);
}

function syncClass_(cls) {
  if (cls === '' || cls == null) return;
  const ss = SpreadsheetApp.getActive(), name = cname_(cls);
  let s = ss.getSheetByName(name);
  if (!s) s = ss.insertSheet(name);
  const self = sh_('自評', H_SELF).getDataRange().getValues().slice(1).filter(r => String(r[1]) === String(cls));
  const oth = sh_('他評', H_OTHER).getDataRange().getValues().slice(1).filter(r => String(r[1]) === String(cls));
  const g = {};
  oth.forEach(r => { (g[r[2]] = g[r[2]] || []).push(r); });
  const rows = self.map(r => {
    const o = g[r[2]] || [], n = o.length;
    const av = [0, 1, 2, 3].map(i => n ? round1_(o.reduce((a, x) => a + Number(x[3 + i]), 0) / n) : '');
    const rt = o.filter(x => x[8] !== '' && x[8] != null).map(x => Number(x[8]));
    return [r[0], r[2], r[3], r[4], r[5], r[6], r[7], r[8], r[9], r[10], main_([r[7], r[8], r[9], r[10]]), r[11], r[12],
      n, av[0], av[1], av[2], av[3], n ? main_(av) : '',
      rt.length ? round1_(rt.reduce((a, b) => a + b, 0) / rt.length) : '',
      o.map(x => x[9]).filter(String).join(' ｜ ')];
  });
  s.getRange(1, 1, 1, H_CLASS.length).setValues([H_CLASS]);
  s.setFrozenRows(1);
  s.getRange('E:E').setNumberFormat('@');
  s.getRange(2, 1, Math.max(s.getMaxRows() - 1, 1), H_CLASS.length).clearContent();
  if (rows.length) s.getRange(2, 1, rows.length, H_CLASS.length).setValues(rows);
  stats_(s);
  statRow_(cls, name);
}

function stats_(s) {
  const L = ['D 老虎', 'I 孔雀', 'S 無尾熊', 'C 貓頭鷹'], sc = ['G', 'H', 'I', 'J'], oc = ['O', 'P', 'Q', 'R'];
  const v = [['類型', '自評主型人數', '自評平均', '他評平均']];
  for (let i = 0; i < 4; i++)
    v.push([L[i], '=COUNTIF($K$2:$K,"' + TYPES[i] + '")',
      '=IFERROR(AVERAGE(' + sc[i] + '2:' + sc[i] + '),0)', '=IFERROR(AVERAGE(' + oc[i] + '2:' + oc[i] + '),0)']);
  s.getRange('W1:Z5').setValues(v);
  s.getRange('W7:X8').setValues([['人數', '=COUNTA($B$2:$B)'], ['平均滿意度', '=IFERROR(AVERAGE($L$2:$L),"")']]);
  if (s.getCharts().length === 0) {
    try {
      s.insertChart(s.newChart().setChartType(Charts.ChartType.PIE).addRange(s.getRange('W1:X5')).setNumHeaders(1)
        .setPosition(10, 23, 0, 0).setOption('title', '自評主型分布').build());
      s.insertChart(s.newChart().setChartType(Charts.ChartType.COLUMN).addRange(s.getRange('W1:W5'))
        .addRange(s.getRange('Y1:Z5')).setNumHeaders(1).setPosition(25, 23, 0, 0)
        .setOption('title', '自評與他評平均').build());
    } catch (err) { /* 圖表建立失敗不影響資料 */ }
  }
}

function statRow_(cls, name) {
  const t = sh_('統計', ['課堂代號', '人數', 'D平均', 'I平均', 'S平均', 'C平均']);
  const ids = t.getRange(1, 1, Math.max(t.getLastRow(), 1), 1).getValues().map(r => String(r[0]));
  if (ids.indexOf(String(cls)) >= 0) return;
  t.appendRow([String(cls)]);
  const r = t.getLastRow(), q = "'" + name + "'!";
  t.getRange(r, 2, 1, 5).setValues([[
    '=COUNTA(' + q + 'B2:B)',
    '=IFERROR(AVERAGE(' + q + 'G2:G),0)', '=IFERROR(AVERAGE(' + q + 'H2:H),0)',
    '=IFERROR(AVERAGE(' + q + 'I2:I),0)', '=IFERROR(AVERAGE(' + q + 'J2:J),0)']]);
}

function rebuildAll() {
  const cs = {};
  [['自評', H_SELF], ['他評', H_OTHER]].forEach(x =>
    sh_(x[0], x[1]).getDataRange().getValues().slice(1).forEach(r => { if (r[1] !== '') cs[r[1]] = 1; }));
  Object.keys(cs).forEach(syncClass_);
}

function out_(o) {
  return ContentService.createTextOutput(JSON.stringify(o)).setMimeType(ContentService.MimeType.JSON);
}
function doPost(e) {
  const d = JSON.parse(e.postData.contents), now = new Date();
  const lock = LockService.getScriptLock();
  lock.waitLock(20000);
  try {
    if (d.type === 'self')
      sh_('自評', H_SELF).appendRow([now, d.cls || '', d.id, d.name || '', d.email || '',
        d.phone ? "'" + d.phone : '', d.line || '', ...d.score, d.rating || '', String(d.note || '').slice(0, 300)]);
    else if (d.type === 'other')
      sh_('他評', H_OTHER).appendRow([now, d.cls || '', d.id, ...d.score, d.code || '',
        d.rating || '', String(d.msg || '').slice(0, 200)]);
  } finally { lock.releaseLock(); }
  if (lock.tryLock(5000)) {
    try { syncClass_(d.cls); } catch (err) { } finally { lock.releaseLock(); }
  }
  return ContentService.createTextOutput('ok');
}
function doGet(e) {
  const p = e.parameter;
  if (p.a === 'others') {
    const rows = sh_('他評', H_OTHER).getDataRange().getValues().slice(1).filter(r => r[2] === p.id);
    return out_(rows.map(r => ({ score: r.slice(3, 7), code: r[7], rating: r[8], msg: r[9] })));
  }
  if (p.a === 'host' && p.key === HOST_KEY) {
    const rows = sh_('自評', H_SELF).getDataRange().getValues().slice(1).filter(r => !p.cls || r[1] === p.cls);
    const n = rows.length, main = [0, 0, 0, 0];
    const avg = [0, 1, 2, 3].map(i => n ? rows.reduce((a, r) => a + r[7 + i], 0) / n : 0);
    rows.forEach(r => { const sc = r.slice(7, 11); main[sc.indexOf(Math.max(...sc))]++; });
    const rated = rows.filter(r => r[11] !== '');
    const rate = rated.length ? rated.reduce((a, r) => a + Number(r[11]), 0) / rated.length : 0;
    const g = {};
    sh_('他評', H_OTHER).getDataRange().getValues().slice(1)
      .filter(r => !p.cls || r[1] === p.cls)
      .forEach(r => { (g[r[2]] = g[r[2]] || []).push(r.slice(3, 7)); });
    const others = Object.keys(g).map(id => ({ id: id, n: g[id].length,
      avg: [0, 1, 2, 3].map(i => g[id].reduce((a, s) => a + s[i], 0) / g[id].length) }));
    return out_({ n, avg, main, rate, rows: rows.map(r => ({ id: r[2], name: r[3], score: r.slice(7, 11), rating: r[11], note: r[12] })), others });
  }
  return out_({ error: 'denied' });
}
```
寫入使用 `LockService`；總表寫入成功後才同步班級分頁，同步失敗不會影響總表。若同時很多人送出，班級分頁可能暫時落後，可手動執行 `rebuildAll`，或在 Apps Script「觸發條件」建立每 5 分鐘執行 `rebuildAll` 的時間驅動觸發。
修改 Apps Script 後，必須「部署」→「管理部署作業」→ 編輯 → 新版本，`/exec` 網址才會生效。

## 8. 主持人頁與分享功能
- 主持人頁（手機與電腦同一個頁面）：輸入課堂代號與密碼，一次顯示：統計（人數、四項平均、**自評主型圓餅圖**與各類型人數與百分比）、自評名單（名字或編號、DISC 分數、主型、滿意度）、他評名單（**含他評主型圓餅圖**、被評者、評價數、他評平均）、建議與回饋；表格可左右捲動。圓餅圖由前端 `pie()` 以 SVG 繪製，不依賴外部程式庫。
- 主持人頁下方有學員報名 QR（帶課堂代號）與主持人入口 QR，學員報名 QR 附分享列。
- 分享列 `shareRow(url, qr索引, 文字)`：顯示可全選的連結欄，以及按鈕：複製連結（`cplink`）、分享連結（`sharelink`，用系統分享 `navigator.share`，不支援時改複製）、用 LINE 傳送（連結 `https://line.me/R/msg/text/?...`）、下載 QR 圖片（`dlqr`，SVG 轉 canvas 轉 PNG）、分享 QR 圖片（`shareqr`，`navigator.share({files})`，不支援時提示改用下載）。
- 分享列用在：第二步「請重要它人評價你」頁（評價連結）與主持人頁（學員報名連結）。
- 第二步頁面已移除舊的「連結（可修改後按更新）」輸入欄與「更新 QR」按鈕（原本用來手動改 QR 的基底網址，已被自動帶入的連結與分享列取代）。
- 評價者留言：`?r=` 的評價者填完 5 組後，會先到 `rmsg` 步驟（給對方的一句話、滿意度，皆選填）再一次送出；一句話由被評者在「收取網路上的評價」時收到，顯示在被評者頁、他評頁與比對頁（也會出現在 PDF）。
- 提示訊息用 `toast()`（不重繪畫面）。

## 8-1. 更新流程
1. 取得目前的 `index.html`（從 GitHub 儲存庫下載）。
2. 在本地修改，檔名維持 `index.html`，保持單一檔案、無建置步驟。
3. 在 GitHub 儲存庫：Add file → Upload files → 覆蓋 `index.html` → Commit changes。
4. 等 1–2 分鐘，手機重新整理（必要時清快取）。
5. 若動到 Apps Script，另做新版本部署（見上）。

## 9. 驗證清單（改完請至少檢查）
- 自評：每組 1/3/5/7 重複選擇時，舊分數被清掉；5 組填滿才能計算。
- 回上一頁：作答保留；重新整理後資料還在。
- 評價者：`?r=` 與 `#rater` 兩種進入方式；完成後出現 6 字母代碼；貼回代碼可加入，重複或錯誤代碼會被擋。
- 三種課堂資料模式：切換選項時欄位顯示正確；需要同意時沒勾會被擋，且不會洗掉已填內容。
- 送出：資料出現在試算表；匿名時名字欄為空。
- 主持人頁：密碼錯誤顯示拒絕；正確時顯示統計，無聯絡資料。
- 列印 PDF：按鈕隱藏、深色模式也是白底黑字。
- 實機：iPhone Safari、Android Chrome 的排版與列印。
- 課堂代號門檻：沒代號只看到輸入頁；輸入空白被擋；`?c=` 會略過；「更換」可回到輸入頁。
- 第一頁：只有課堂資料；不預選；沒選就按確定被擋；選了才出現欄位；需同意時被擋且內容保留；按確定後縮成摘要並出現第二頁；「修改」可回來。
- 解析引擎：同樣輸入結果完全相同；不同作答、作答節奏、紀錄次數會有不同內容；多次隨機測試不得出現 undefined、NaN、null；時間分級邊界、修改次數、停留最久的一組、紀錄比較、重新開始保留紀錄與清除紀錄、周哈里窗一致度數值（需與獨立計算相符）。
- 重新開啟：有舊紀錄時先看到第一頁與續做卡片；按「繼續上次進度」回到結果頁或第二步；按「從頭開始」清空紀錄但保留課堂代號。
- 評價者：`?r=` 有留言步驟、按送出前沒有任何 POST；`#rater` 沒有留言步驟。
- 主持人頁：兩張圓餅圖、舊版伺服器回應（無名單）仍可顯示、空班級顯示「尚無資料」。
- Apps Script 模擬測試（Node 模擬 Sheets，約 17 項）：班級分頁、統計列、圖表只建一次、舊 8 欄他評表升級、危險分頁名稱、無個資外洩。
- 輸入框外的文字標籤是否都在（課堂代號、姓名或代號、Email、電話、LINE ID）。
- 分享列：複製、系統分享、LINE 連結內容正確、QR 圖片下載與分享（需在實機驗證）。
- 主持人頁：統計＋自評名單＋他評名單＋建議同時顯示；舊版伺服器回應（無名單）也不會壞；回應中不得出現 Email、電話、LINE ID。
- Apps Script 邏輯可用 Node 模擬 `SpreadsheetApp` 與 `ContentService` 測試（本專案做過：他評讀取、課堂隔離、密碼錯誤、無個資外洩）。
- 自動化測試做法（本專案用過）：Node + jsdom 載入 `index.html`，stub `fetch`、`print`、`scrollTo`，以 `click()` 走完整流程。

## 10. 已知限制
- 作答時間分級的門檻是暫訂值，且受閱讀速度、裝置、環境影響；之後可累積實際課堂資料再校準。
- 職涯方向只是參考，非職業判定。
- 解析文字是規則組合而成，不是即時由 AI 生成；想要更多變化需要增加文字庫。
- Apps Script 網址在公開頁面原始碼中可見，任何人都能亂送資料；主持人密碼只擋「讀取統計」，擋不住灌水。
- 主持人密碼目前以 GET 參數傳送（網址會被記錄），不要在公用電腦使用。
- 以短網址包裝帶有 `?r=` 的連結可能失效。
- PDF 依賴瀏覽器列印，iPhone 需手動兩指放大預覽再儲存；各手機中文排版尚未全面實測。
- 個資：目前已有告知與勾選同意，但正式課堂使用前請專業人士確認是否符合個資法。

- 課堂代號沒有白名單驗證，也沒有主持人預先建立班級的機制；大小寫或前後空白不同會被當成不同班（例如 `b52tw` 與 `B52tw` 會產生兩個分頁），建議之後統一轉小寫並去除空白。
- 班級分頁的圖表與公式是在 Google 端產生，無法在本機自動測試，部署後要人工檢查一次。
- 主持人頁會顯示學員自填的名字（可能是真名）、DISC 分數、滿意度與建議；Email、電話、LINE ID 只存在試算表，不會顯示在網頁。
- QR 圖片下載與分享依賴瀏覽器的 canvas 與 `navigator.share`，各手機支援度不同，需實機驗證。

## 11. 待辦與待確認
### 擬議但尚未實作：個人背景獨立一頁（使用者尚未決定）
想法：自評後多一頁選填問卷（家中排行、兄弟姊妹人數與性別、與誰較親、MBTI 代碼、星座），有填就在報告多一段「綜合參考」，沒填就跳過。
建議做法（若使用者同意）：
- 這些資料**不得改動 DISC 分數**，只作為獨立的「背景與對照」段落，措辭為參考與自我反思題，不下結論。
- 資料屬較私人資訊，預設只存在本機；只有選項 3 並另外勾同意才上傳；主持人頁只顯示彙整次數，不顯示個人答案。
- 研究面：出生序對人格的影響在大型研究中多半很小；星座在雙盲實驗中沒有預測力；MBTI 的分類穩定度與效度常被質疑。因此報告需標明「趣味與反思用」。
- MBTI 與 DISC 的對照只能當啟發式說明（例如外向偏高常與 D、I 並行），需寫明量測的面向不同，不一致不代表有錯。


- 確認計分文字：是否真的要「2 分」，或維持 1/3/5/7（目前為 3）。
- 實機驗證主持人頁（含自評名單、他評名單）與 QR 圖片分享；試算表「他評」寫入已實測成功。
- 可考慮：課堂白名單（主持人先建立班級代號）。
- 可考慮：以隨機簽章降低灌水風險、課堂代號白名單、主持人頁改用 POST 送密碼。

## 12. 變更紀錄
- 初版：自評、他評代碼、變色龍、周哈里窗。
- 加入：回上一頁、作答回顧、三份 PDF 報告（列印）。
- 加入：Google 試算表收集、課堂代號、掃碼評價自動回傳、主持人統計頁。
- 2026-11-24（第三次更新）：首頁改為先輸入課堂代號（或掃 QR）才顯示後續內容；課堂資料卡片移到說明之前並加上文字標籤；主持人頁新增自評名單、他評名單、建議與滿意度（Apps Script 同步更新）；第二步與主持人頁新增複製連結、系統分享、LINE 傳送、下載與分享 QR 圖片。實測確認：Google 試算表「他評」分頁已成功收到掃碼評價資料，主持人頁可正常讀取。
- 2026-11-24：標題改為「Junba 維護 DISC 人際風格測驗」；引導文字改為「請重要它人評價」；評分說明新增 5 與 3 的文字；課堂資料三選項改名並加入名字欄；修正切換選項時欄位不顯示與勾同意後內容被洗掉的問題。
