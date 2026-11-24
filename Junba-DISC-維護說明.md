# Junba 維護 DISC 人際風格測驗：維護與交接說明

給未來接手的人或其他 AI 模型：先讀完本文件，再動手改 `index.html`。
最後更新：2026-11-24（第二次更新）

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
- 狀態：全域 `st`，主要欄位 `s`（目前畫面）、`self`、`gs`（自評 grid）、`go`（目前評價 grid）、`others`（他評分數陣列）、`og`（他評 grid 陣列）、`codes`、`prof`（課堂資料模式與欄位）、`id`（參與者編號）、`sent`。
- 畫面 `st.s`：`intro`、`self`、`selfRes`、`invite`、`other`、`otherRes`、`cmp`、`host`（主持人）、`rintro`、`rres`（給評價者）。
- 渲染：`render()` 以 innerHTML 重畫整頁；事件用單一 `click` 委派，按鈕以 `data-a`（動作）、`data-g/r/c/v`（作答）區分。
- `BACK` 對照表控制「← 回上一頁」。
- 儲存：`localStorage` 鍵 `disc-v1`（自評、作答、他評、課堂資料設定、編號）；評價者模式不儲存。
- 模式判斷（網址）：
  - `?r=編號&c=課堂代號`：評價者模式，完成後自動送到雲端並顯示代碼。
  - `#rater`：評價者模式（未選上傳時用代碼回傳）。
  - `?host=1&c=課堂代號`：主持人頁。
  - `?c=課堂代號`：學員報名，自動帶入課堂代號，並略過「輸入課堂代號」頁。
- PDF：用 `window.print()`，列印樣式 `@media print` 會隱藏按鈕並加報告標題（含日期與名字）。

## 6. 進入流程與課堂資料三種模式（`st.prof.mode`）
**進入流程（課堂代號門檻）**：
1. 沒有課堂代號時（`st.prof.cls` 為空，且網址沒有 `?c=`），首頁只顯示「先輸入課堂代號」，看不到後面的選項與測驗說明。
2. 輸入代號（按「進入課堂」）或掃描帶 `?c=` 的 QR 後，才顯示：課堂資料（選填）卡片 → 測驗說明 → 開始自評。
3. 課堂資料卡片頂端顯示「課堂代號：xxx」與「更換」按鈕（`chgcls`，清空代號回到輸入頁）。
4. 輸入框外都有文字標籤：課堂代號、姓名或代號、Email、電話、LINE ID。
5. 評價者模式（`?r=`、`#rater`）與主持人頁不受此門檻影響。
6. 目前代號只是分組標籤，沒有白名單驗證；任何人輸入任何代號都能進入。

**三種模式**：
| 值 | 畫面文字 | 行為 |
|---|---|---|
| 0 | 1. 留名字／匿名（只在裝置上使用，不上傳） | 不送任何資料；名字只用在 PDF 標題 |
| 1 | 2. 用名字／匿名（供課堂統計，只上傳分數） | 按「送出」才上傳分數與名字（可空白＝匿名）；填了名字須勾同意 |
| 2 | 3. 提供姓名（留下聯絡資料，可進一步得到聯絡分析機會） | 另外上傳 Email、電話、LINE ID（皆選填）；須勾同意 |
- 預設為 0。送出動作 `send` 由使用者按鈕觸發，不會自動上傳。

## 7. Google 試算表與 Apps Script
工作表：
- `自評` 欄位：時間、課堂代號、編號、姓名或代號、Email、電話、LINE ID、D、I、S、C、滿意度、建議
- `他評` 欄位：時間、課堂代號、被評者編號、D、I、S、C、作答代碼
- `統計`：每個課堂代號的人數與四項平均（公式）

API：
- `POST`（前端用 `fetch(..., {mode:'no-cors'})` 送 JSON 文字）：
  - `{type:'self', cls, id, name, email, phone, line, score:[D,I,S,C], rating, note}`
  - `{type:'other', cls, id, score:[D,I,S,C], code}`
- `GET ?a=others&id=編號`：回傳該編號收到的他評陣列 `[{score, code}]`。
- `GET ?a=host&key=密碼&cls=課堂代號`：回傳 `{n, avg:[4], main:[4], rate, rows:[{id,name,score:[4],rating,note}], others:[{id,n,avg:[4]}]}`；密碼錯誤回 `{error:'denied'}`。**不含 Email、電話、LINE ID**。
  - 前端 `ext(j)` 會用這些欄位顯示「自評名單」「他評名單」「建議與回饋」；若伺服器是舊版（沒有 rows/others），前端仍可只顯示統計卡片，不會出錯。
  - 更新 Apps Script 後必須重新部署新版本才會生效。

Apps Script 程式碼（密碼請使用者自行設定，勿寫進前端）：

```js
const HOST_KEY = '請改成你自己的主持人密碼';
const H_SELF = ['時間','課堂代號','編號','姓名或代號','Email','電話','LINE ID','D','I','S','C','滿意度','建議'];
const H_OTHER = ['時間','課堂代號','被評者編號','D','I','S','C','作答代碼'];

function sh_(name, head) {
  const ss = SpreadsheetApp.getActive();
  let s = ss.getSheetByName(name);
  if (!s) { s = ss.insertSheet(name); s.appendRow(head); s.setFrozenRows(1); }
  return s;
}
function setup() {
  sh_('自評', H_SELF); sh_('他評', H_OTHER);
  const s = sh_('統計', ['課堂代號','人數','D平均','I平均','S平均','C平均']);
  s.getRange('A2').setValue('範例班');
  s.getRange('B2').setFormula("=COUNTIF('自評'!B:B,A2)");
  ['H','I','J','K'].forEach((c, i) =>
    s.getRange(2, 3 + i).setFormula("=IFERROR(AVERAGEIF('自評'!B:B,$A2,'自評'!" + c + ":" + c + "),\"\")"));
}
function out_(o) {
  return ContentService.createTextOutput(JSON.stringify(o)).setMimeType(ContentService.MimeType.JSON);
}
function doPost(e) {
  const d = JSON.parse(e.postData.contents), now = new Date();
  if (d.type === 'self')
    sh_('自評', H_SELF).appendRow([now, d.cls||'', d.id, d.name||'', d.email||'',
      d.phone ? "'" + d.phone : '', d.line||'', ...d.score, d.rating||'', d.note||'']);
  else if (d.type === 'other')
    sh_('他評', H_OTHER).appendRow([now, d.cls||'', d.id, ...d.score, d.code||'']);
  return ContentService.createTextOutput('ok');
}
function doGet(e) {
  const p = e.parameter;
  if (p.a === 'others') {
    const rows = sh_('他評', H_OTHER).getDataRange().getValues().slice(1).filter(r => r[2] === p.id);
    return out_(rows.map(r => ({ score: r.slice(3, 7), code: r[7] })));
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
修改 Apps Script 後，必須「部署」→「管理部署作業」→ 編輯 → 新版本，`/exec` 網址才會生效。

## 8. 主持人頁與分享功能
- 主持人頁（手機可用）：輸入課堂代號與密碼，一次顯示：統計（人數、四項平均、主型人數）、自評名單（名字或編號、DISC 分數、主型、滿意度）、他評名單（被評者、評價數、他評平均）、建議與回饋；表格可左右捲動。
- 主持人頁下方有學員報名 QR（帶課堂代號）與主持人入口 QR，學員報名 QR 附分享列。
- 分享列 `shareRow(url, qr索引, 文字)`：顯示可全選的連結欄，以及按鈕：複製連結（`cplink`）、分享連結（`sharelink`，用系統分享 `navigator.share`，不支援時改複製）、用 LINE 傳送（連結 `https://line.me/R/msg/text/?...`）、下載 QR 圖片（`dlqr`，SVG 轉 canvas 轉 PNG）、分享 QR 圖片（`shareqr`，`navigator.share({files})`，不支援時提示改用下載）。
- 分享列用在：第二步「請重要它人評價你」頁（評價連結）與主持人頁（學員報名連結）。
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
- 輸入框外的文字標籤是否都在（課堂代號、姓名或代號、Email、電話、LINE ID）。
- 分享列：複製、系統分享、LINE 連結內容正確、QR 圖片下載與分享（需在實機驗證）。
- 主持人頁：統計＋自評名單＋他評名單＋建議同時顯示；舊版伺服器回應（無名單）也不會壞；回應中不得出現 Email、電話、LINE ID。
- Apps Script 邏輯可用 Node 模擬 `SpreadsheetApp` 與 `ContentService` 測試（本專案做過：他評讀取、課堂隔離、密碼錯誤、無個資外洩）。
- 自動化測試做法（本專案用過）：Node + jsdom 載入 `index.html`，stub `fetch`、`print`、`scrollTo`，以 `click()` 走完整流程。

## 10. 已知限制
- Apps Script 網址在公開頁面原始碼中可見，任何人都能亂送資料；主持人密碼只擋「讀取統計」，擋不住灌水。
- 主持人密碼目前以 GET 參數傳送（網址會被記錄），不要在公用電腦使用。
- 以短網址包裝帶有 `?r=` 的連結可能失效。
- PDF 依賴瀏覽器列印，iPhone 需手動兩指放大預覽再儲存；各手機中文排版尚未全面實測。
- 個資：目前已有告知與勾選同意，但正式課堂使用前請專業人士確認是否符合個資法。

- 課堂代號沒有白名單驗證，也沒有主持人預先建立班級的機制。
- 主持人頁會顯示學員自填的名字（可能是真名）、DISC 分數、滿意度與建議；Email、電話、LINE ID 只存在試算表，不會顯示在網頁。
- QR 圖片下載與分享依賴瀏覽器的 canvas 與 `navigator.share`，各手機支援度不同，需實機驗證。

## 11. 待辦與待確認
- 確認計分文字：是否真的要「2 分」，或維持 1/3/5/7（目前為 3）。
- 實機驗證主持人頁（含自評名單、他評名單）與 QR 圖片分享；試算表「他評」寫入已實測成功。
- 可考慮：課堂白名單（主持人先建立班級代號）。
- 可考慮：以隨機簽章降低灌水風險、課堂代號白名單、主持人頁改用 POST 送密碼。

## 12. 變更紀錄
- 初版：自評、他評代碼、變色龍、周哈里窗。
- 加入：回上一頁、作答回顧、三份 PDF 報告（列印）。
- 加入：Google 試算表收集、課堂代號、掃碼評價自動回傳、主持人統計頁。
- 2026-11-24（第二次更新）：首頁改為先輸入課堂代號（或掃 QR）才顯示後續內容；課堂資料卡片移到說明之前並加上文字標籤；主持人頁新增自評名單、他評名單、建議與滿意度（Apps Script 同步更新）；第二步與主持人頁新增複製連結、系統分享、LINE 傳送、下載與分享 QR 圖片。實測確認：Google 試算表「他評」分頁已成功收到掃碼評價資料，主持人頁可正常讀取。
- 2026-11-24：標題改為「Junba 維護 DISC 人際風格測驗」；引導文字改為「請重要它人評價」；評分說明新增 5 與 3 的文字；課堂資料三選項改名並加入名字欄；修正切換選項時欄位不顯示與勾同意後內容被洗掉的問題。
