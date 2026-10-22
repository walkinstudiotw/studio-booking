# WalkIn 水星錄音室預約系統

單檔 vanilla JS（`index.html`）+ Firebase Realtime Database + Google Apps Script 同步後端。
自 [meeting-room](https://github.com/walkinstudiotw/meeting-room)（會議室預約）fork，資料與後端完全獨立。

- 線上網址：https://walkinstudiotw.github.io/studio-booking/
- 本機模擬模式（假資料、不連 Firebase）：`index.html?mock=1`

## 功能總覽

| 客戶類型 | 功能 |
|---|---|
| 單次預約（訪客） | 看可預約時段（訪客開放時段）→ 填姓名／Email／電話／節目名稱 → 匯款（NT$/hr，勾「開發票」則總額 +5% 並填統編／抬頭）＋填帳號末五碼 → 送出申請 → 管理員核對後成立；單次最長 8 小時 |
| 新節目儲值 / 一般方案儲值（會員） | 手機號碼查詢登入 → 看儲值時數批次與到期日 → 填節目名稱、用時數預約（**免審核**，約 1 分鐘內自動確認；姓名／電話／Email 直接帶會員資料，建會員時 Email 必填）；時數不足改匯款（依類型時薪，可勾開發票 +5%） |
| 管理員 | Firebase 登入 → 審核匯款單（顯示金額＋末五碼）、會員 CRUD、儲值時數（有期限的批次，預設帶該類型方案時數）、系統設定（開放時段/各類型方案時數與價格、不足改匯款時薪、預設期限/日曆 ID/事件格式） |

線上儲值：會員在「會員專區」按「儲值時數」→ 顯示該類型方案（時數＋價格）、使用期限提醒、可勾開發票（統編＋抬頭，+5%）、匯款資訊與末五碼 → 送出申請；管理員在「待審核 → 儲值申請」核對款項後「核准入帳」（自動發放一筆批次，期限依類型設定）或「婉拒」，Apps Script 寄通知。

取消規則：審核中的預約客人可直接取消。**已確認**的預約客人可線上「申請取消」（預設開始前 24 小時內不可申請，後台「設定」可改），管理員在「待審核 → 取消申請」同意（釋出時段、退回時數、移除日曆事件、寄通知）或駁回（預約維持、通知客人）。

時數規則：**沒有每月額度**，全部是儲值批次。每筆儲值有使用期限，預設＝儲值當天＋該類型「儲值預設使用期限」（出廠 365 天，後台「設定」可改）；儲值時可逐筆改日期，事後也可編輯。扣抵先用快到期的批次；預約日必須落在批次有效期內。

## 首次啟用步驟

### 1. Firebase 規則（必做，否則整個系統無法讀寫）

Firebase Console → talkloudpm 專案 → Realtime Database → 規則：
把 `rules.walkin-studio.json` 裡的 `walkin-studio` 區塊**合併**進既有 rules JSON（與 `walkin-pm`、`meeting-room` 節點並列），發布。

### 2. 初始化設定

用管理員帳號（walkinstudiotw@gmail.com）打開網站 → 管理後台登入。
首次登入會自動寫入預設設定；到「設定」調整：匯款帳戶資訊（必改）、單次預約時薪、兩種儲值類型的時薪與預設期限、開放時段、日曆 ID。

### 3. Google 日曆

為錄音室另建一個 Google 日曆（不要共用會議室日曆），把日曆 ID 填到後台「設定」。
日曆若不屬於 Apps Script 專案的擁有者帳號，需分享給該帳號並給「進行變更」權限。

### 4. Apps Script 同步後端（獨立專案，與會議室的分開）

1. 用 walkinstudiotw@gmail.com 登入 [script.google.com](https://script.google.com) 新增專案，貼上 `appscript.gs`
2. 專案設定 → 指令碼屬性：`FB_EMAIL` / `FB_PASSWORD` / `FB_API_KEY` / `DB_URL` / `ADMIN_EMAIL`（值見 appscript.gs 開頭註解）
3. 專案設定 → 時區 **Asia/Taipei**
4. 編輯器執行 `testConnection` 完成授權並確認連線（記錄應顯示 settings 與日曆名稱）
5. 執行 `installTrigger` 安裝每 1 分鐘觸發器

### 5. 部署

Push 到 `main` → GitHub Pages（Settings → Pages → Deploy from branch → main / root）。
驗收記得硬刷新（⌘+Shift+R）。

## 信任模型（已知取捨）

- **手機號碼＝會員登入憑證**（無密碼）：知道號碼者可查該會員時數並用其額度預約；濫用時管理員可取消預約並停用會員。
- 開放時段/時數由前端＋Apps Script 在確認時強制；惡意 client 可寫入 out-of-hours 的 pending，會在約 1 分鐘內被自動婉拒（會員單）或由人工審核擋下（匯款單）。
- 未登入者可 create pending 申請（欄位有白名單＋長度上限、不佔用已確認時段）；被灌爆時再加訪客通行碼。
- 時段表（無姓名，只有起迄與狀態）全球可讀。
- 管理員 Firebase 密碼存於 Apps Script 指令碼屬性（信任層級同 walkin-pm 寄信 webhook）。
- 日曆手動事件只匯入起迄時間，**不**公開事件標題。

## 資料結構（`/walkin-studio/`）

```
settings                        公開讀；roomName/notice/slotMinutes/bookingWindowDays/
                                openHours{visitor,member}/
                                payment{hourlyRate,bankInfo,types{program|general:{pkgHours,pkgPrice,hourlyRate,grantDays}}}/
                                calendar{calendarId,eventTitle,eventDesc}
schedule/{date}/{bid}           公開讀 {s,e,st,k,cr?}（分鐘制；k: m|v|cal；cr=true 取消申請中，客人可對 confirmed 寫入）
bookings/{bid}                  admin 讀；完整預約（含 PII、payLast5、calEventId、通知戳記）
                                show 節目名稱；invTaxId/invTitle 開發票（有則 amount 已含 5% 稅）
                                cancelReq{at,reason,notifiedAt}：客人對 confirmed 預約可建立一次（取消申請）；cancelDenied{at,reason,notifiedAt}：駁回紀錄
members/{code}                  admin；完整會員資料（type: program|general；quota 恆為 0）
membersPublic/{code}            按 key 讀 {name,quota,active,type}
memberIndex/{手機數字}          按 key 讀 → code
memberGrants/{code}/{gid}       按 code 讀 {h,start,exp,note,at}（儲值批次）
memberBookings/{code}/{ym}/{bid} 按 code 讀 {date,s,e,st,pay,c}（會員端算餘額）
topups/{tid}                    admin 讀；會員線上儲值申請 {code,type,hours,price,amount,invTaxId?,invTitle?,payLast5,status,createdAt,grantId?,rejectReason?}（會員只能 create pending）
memberTopups/{code}/{tid}       按 code 讀 {h,amount,st,c,reason?}（會員端看申請狀態）
meta                            admin；lastSyncAt/lastError
```

寫入權限：非管理員只能「create status=pending」與「刪除自己的 pending」（push id 即取消憑證，存 localStorage 的 `ws.*` 鍵）；`confirmed` 只有管理員憑證（網頁後台或 Apps Script）能寫。
