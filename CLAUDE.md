# KHHouse 專案筆記(給 Claude Code)

高雄重點行政區建案儀表板。前台為純靜態 `data.json` + HTML/JS(`index.html`、`daliao.html`
等分區頁),後台 `admin.html` 直接用 GitHub API 讀寫 `data.json`(main 分支)。

## 工作流程慣例

- **PR 可以直接 merge,不用等使用者確認。**(使用者於 2026-09-09 明確表示「以後可以都直接
  MERGE」)包含這個репo 目前僅有的 workflow 是 GitHub Pages 的自動部署,沒有針對 PR 的 CI,
  所以正常情況下 PR 一開好、無 merge conflict 就可以直接 merge。
- 新 session 開始時,先看這份檔案 + `git log --oneline -20`,不要對已經討論過的結論重新提問
  (例如:大寮頁欄位缺口、豐鉅成家查無此案等)。

## 資料 schema(data.json → projects[])

新增欄位時,同時要改三個地方:`data.json` 每筆資料、`admin.html` 表單(`renderForm()` 的
`fld()` 呼叫 + `set()` 綁定清單 + `num[]` 陣列(如果是數字)+ `addProject()` 預設值)、
以及各分區頁(`index.html`/`daliao.html`)的渲染邏輯。三者有一個沒改,就會出現「後台看得到
資料、前台卻沒顯示」或「後台編輯不到某個前台有顯示的欄位」的情況。

欄位標準順序(照 admin.html 現有慣例):
```
name, dev, status, addr, district, zone, permit, floor,
avgPrice, minPrice, maxPrice, count, askPrice, units, layout,
r1, r2, r3, r4, loan, handover, kit, bath, other, amenities,
notes, analysis, vs, vsPublic, tagMode, tagManual, warn, warnReason,
lat, lng, s591, s958, sleju, floorPlans,
coverage, mgmtFee, parkingClean, supplement,
landPrice, launchDate, parkingPrice, parkingSize, collectFee,
totalMin, totalMax   ← 2026-09-09 補上,admin.html 對應在「價格與成交」區塊
```

## 已知 bug / 已修過的坑

1. **`minPrice`/`maxPrice` 為 null 但 `avgPrice` 有值時,modal 會顯示「null ~ null」。**
   已在 daliao.html 修成只在兩值都非 null 時才顯示單價區間(2026-09-09)。若之後改
   index.html 或其他分區頁,要檢查同一個 pattern 有沒有一樣的問題。
2. **`totalMin`/`totalMax`(總價區間,萬)曾經是「資料存在但沒有任何頁面顯示」的孤兒欄位**
   ——有人直接寫進 data.json,但 admin.html 表單和 daliao.html 渲染邏輯都沒有對應。
   已於 2026-09-09 補齊三處。以後新增欄位時要記得同步三個地方(見上面 schema 段落)。
3. **GitHub Pages 部署會偶發失敗,不是資料或程式碼的問題。**
   2026-09-08 晚間連續兩次 push 之後,「pages build and deployment」workflow 在
   `Upload artifact` 步驟收到 `403 Forbidden: Error from intermediary`(GitHub 服務端
   暫時性錯誤),導致網站停在舊版本、admin.html(直接讀 GitHub API)卻已經是新資料
   ——兩邊「不一致」不代表資料沒存到。判斷方式:用
   `mcp__github__actions_list` (method: list_workflow_runs) 看最新幾次
   「pages build and deployment」的 conclusion,再用 `get_job_logs` 看失敗那次的
   `Upload artifact` 步驟。修法:推一個新 commit 觸發全新部署(重跑舊的失敗 run 需要
   的權限,目前這個 GitHub App 整合沒有,`rerun_workflow_run` 會 403)。
4. **大寮區(daliao.html)資料完整度明顯低於楠梓/仁武**:15 案(截至 2026-09-09)完全沒有
   `lat`/`lng`,地圖分頁是空的;`analysis`、`vs` 幾乎全空;`kit`/`bath`/`other`/`amenities`
   填寫率也偏低。這是資料尚未補齊,不是程式邏輯問題。
5. **這個 remote 執行環境連不到 591 / 樂居(leju.com.tw) / house958.com / 高雄房地王
   (housetube.tw)等房地產網站(WebFetch 會被 egress proxy 擋掉,回傳 EGRESS_BLOCKED),
   也連不到任何地理編碼服務(如 Nominatim,同樣被 proxy 擋)。** 只能用 WebSearch
   拿到的摘要拼湊資料,精確度有限,而且完全沒辦法幫建案標座標。遇到需要座標或需要逐頁
   核對 591/樂居/958 資料的任務,要老實跟使用者說清楚這個限制,不要用猜的座標或憑空編造
   的資料填欄位——尤其是地圖座標這種一旦錯了會誤導看房路線的資訊。真的需要座標時,建議
   使用者用 `admin.html` 內建的「位置編輯」(拖曳圖釘 / 貼 Google Maps 連結解析)手動標定。
6. **「豐鉅成家」這個案名(使用者曾要求新增)在 591/樂居/958/高雄房地王都查不到**,只查到
   同一建商「豐鉅開發建築」在大寮的其他案子(如「豐鉅二三」)。使用者已確認「先跳過」
   (2026-09-09)。之後若使用者再提到這個案名,先確認是否已經拿到正確連結或案名,不要
   直接照搬歷史資料。

## 分區頁維護

- `daliao.html` 是完整複製 `index.html`(高大特區樣板)的排版與互動邏輯,只在載入後
  filter `district==='大寮'`。修 bug 或加欄位顯示時,兩份檔案的同一段邏輯通常要一起檢查
  (是否也要同步修改),但目前兩者是各自獨立的檔案,沒有共用模組,不會自動同步。
