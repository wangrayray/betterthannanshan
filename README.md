# 班務系統（退餐紀錄／外食訂購）— Firebase 版部署說明

已把原本用 Claude artifact `window.storage` 的單檔版本，改寫成用 **Firebase
Firestore + Authentication** 的版本，可以直接放上 GitHub Pages。

檔案說明：

- `index.html` — 完整前端，已內建你的 firebaseConfig（專案 `betterthannanshan`）。
- `firestore.rules` — 安全規則，真正在後端強制角色權限（不是只有前端擋）。
- `firebase.json` / `.firebaserc` / `firestore.indexes.json` — 給 Firebase CLI
  部署規則用的設定檔。

## 資料結構

```
settings/config              全班共用設定（座號數、是否含「師」）
students/{seatNo}            全班座號名冊（伙委在設定頁存檔時自動同步）
staff/{email}                伙委名單，文件 id = 該伙委的登入 email
eventCoordinators/{seatNo}   外食訂購人名單，效期 14 天，過期自動失效
entries/{date}                每日訂餐紀錄
  └ lunch_choice/{seatNo}      該日午餐每人的選擇，一人一份文件
                               {status: meat|veg|cancel, note, updatedBy, updatedAt}
  └ dinner_choice/{seatNo}     該日晚餐每人的選擇，格式同上
events/{eventId}              外食訂購活動
  └ menu/{itemId}              菜單品項（名稱、金額，由伙委/外食訂購人設定）
  └ orders/{seatNo}            每人的訂購內容（品項＋數量，由同學自己選取）
```

之所以把「每人的選擇」「訂單」拆成每人一份文件（子集合），是因為 Firestore
規則只能檢查「整份文件」的讀寫權限，沒辦法檢查「陣列裡的某一筆」。拆成
子集合後，才能讓 Firestore 規則真正做到「同學只能寫自己座號那一份」。

### 退餐紀錄的選擇邏輯

同學介面一開始沒有預設值（一片空白），要自己按下「🍖 葷食」「🥦 素食」「🚫
退餐」其中一個按鈕才會產生紀錄，也可以再按一次同一個按鈕取消選擇（回到空白）；
選好之後可以額外填一段文字備註（例如「不吃蛋」）。伙委介面的座號格會即時反映
每個人目前的狀態，並自動加總「共訂幾人（葷／素）」「退餐幾人」「尚未選擇幾人」，
不需要再手動輸入訂餐人數。伙委也可以直接點座號格幫忘記選的同學代為切換狀態
（未選 → 葷 → 素 → 退餐 → 未選，循環切換）。

同學登入用的是「座號」（跟退餐點選格的號碼一樣，例如 05、師），不是正式學號；
輸入數字時系統會自動補成兩位數（例如輸入 5 會存成 05）以對齊點選格。

同學按「🍖／🥦／🚫」只是先在本機暫存選擇（畫面會顯示「尚未送出」），實際寫入
Firestore、讓伙委端看到，要再按一次「確定送出」才會生效；這樣同學可以先點來
點去比較，不會每點一下都送出一次請求。伙委端每個日期卡片除了顏色格子，也會
額外列出「尚未選擇（N）：座號清單」「退餐：座號清單」，不用逐格辨識顏色。

### 重要修正紀錄：collectionGroup 查詢與安全規則

app 用 `collectionGroup()` 一次讀取「所有日期」底下的 `lunch_choice` /
`dinner_choice`（以及所有活動底下的 `menu` / `orders`），這種查詢**只吃寫在
最外層、用 `{path=**}` 萬用字元的規則**；規則如果巢狀寫在 `entries/{date}`
或 `events/{eventId}` 裡面，只對「已知完整路徑」的單一文件讀寫有效，對
collectionGroup 查詢會直接被拒絕。而且因為前端用的是即時監聽
（`onSnapshot`），這種被拒絕預設不會跳出明顯錯誤，看起來就像「資料明明寫
進去了，畫面卻永遠不會更新」。這是系統從一開始就存在的根本問題，已在目前
的 `firestore.rules` 修正（見檔案內 `{path=**}` 開頭那幾條規則），修正後
已實際測試過即時更新。日後如果要新增其他用 collectionGroup 讀取的子集合，
記得比照辦理、規則要寫在最外層。

### 安全性強化：伺服器時間戳記 + 同一份文件的寫入節流

`lunch_choice`／`dinner_choice`／`orders` 這三個 collection 因為同學可以直接
寫自己那一份，如果有人打開瀏覽器 F12、複製 `firebaseConfig`，自己寫一支腳本
用 Firebase SDK 狂洗自己座號那份紀錄（例如 1 秒內送出上萬次「退餐/不退餐」），
安全規則原本只檢查「是不是自己的座號」，擋不住這種本人對自己資料的濫用，
可能在短時間內產生大量計費讀寫、刷爆 Firebase 免費額度，害全班當天都不能
點餐。目前已補上兩層防護：

1. **一律用 `serverTimestamp()`**：前端送出 `updatedAt` 時不再用瀏覽器本機的
   `new Date().toISOString()`（可以被偽造），改用 Firestore 提供的
   `serverTimestamp()`，寫入時由伺服器決定真正的時間，並在 `firestore.rules`
   強制 `request.resource.data.updatedAt == request.time`，偽造時間或乾脆不
   帶這個欄位都會直接被拒絕。
2. **同一份文件節流**：`firestore.rules` 裡新增 `isThrottled()`，同一份文件
   距離上次更新不到 1 秒就拒絕這次寫入（`create` 沒有「上一次」可比較，不受
   此限，只限制 `update`／`delete`）。這樣即使有人狂打同一位同學的紀錄，一秒
   內最多只有 1 次真的寫進去、算進計費用量，其餘全部在規則層被拒絕、不花錢
   也不影響額度。這個規則只設計給 `lunch_choice`／`dinner_choice`／`orders`
   （同學能直接自己寫的高風險 collection），其他 collection（`entries`／
   `events`／`settings`／`students`／`staff`／`eventCoordinators`）本來就只有
   伙委或外食訂購人能寫，不在這次調整範圍內。
   舊資料（這次規則上線前寫入的文件）`updatedAt` 還是字串，規則裡有做相容判斷
   （`is timestamp` 為 false 時視為沒有節流資訊、直接放行），不用手動搬移。

**這次改動除了換一版 `index.html`，`firestore.rules` 也有改，兩個都要重新
部署**（`index.html` 推上 GitHub Pages 會自動生效，但 `firestore.rules` 要照
下面「部署安全規則」那步再跑一次 `firebase deploy --only firestore:rules`，
或去 Firebase Console 手動貼上新版規則，光是更新網頁檔案不會自動套用新規則）。

## 角色權限（Firestore Rules 強制）

- **伙委（staff）**：帳號 email 存在於 `/staff/{email}`，可讀寫所有 collection。
- **外食訂購人（event coordinator）**：伙委在設定頁指定某個座號，效期 14 天。
  期間內可以完整管理 `events` 及其底下的 `menu`／`orders`（建立活動、編輯菜單、
  新增或修改任何人的訂單），但不能碰退餐紀錄、名冊、伙委名單。到期後規則會
  自動視為失效，不需要手動清除資料。這個角色完全在網頁內設定/移除，
  不需要進 Firebase Console。
- **同學（student）**：帳號 email 是 `座號@yourclass.local` 格式，只能寫入
  `lunch_choice/dinner_choice/orders` 底下「文件 id＝自己座號」的那一份，且該
  座號必須已經在 `/students` 名冊中（伙委存座號設定時會自動建立）。外食訂購
  的品項（menu）只有伙委／外食訂購人能新增或修改，同學只能寫自己那份訂單
  （orders），沒有辦法幫別人下單，伙委端也只能檢視座號同學的訂購狀況、無法
  代填，這是刻意設計，避免資料被覆蓋掉。
- 其他讀取（例如看當日退餐人數、活動總金額）對所有已登入使用者開放，這是
  為了讓班務資訊能互相參考；如果你想更嚴格（例如同學看不到別人退了誰），
  可以再跟我說，這需要把讀取也拆得更細。

## 部署前置作業

### 1. 確認 Firestore / Authentication 已啟用

Firebase Console → 你的專案 `betterthannanshan`：
- **Firestore Database**：建立資料庫（正式模式即可，規則等等會覆蓋）。
- **Authentication → Sign-in method**：啟用「電子郵件/密碼」。

### 2. 部署安全規則

方法 A（用 Firebase CLI，推薦）：

```bash
npm install -g firebase-tools
firebase login
# 在放這幾個檔案的資料夾下執行：
firebase deploy --only firestore:rules
```

方法 B（手動）：打開 Firebase Console → Firestore Database → 規則，把
`firestore.rules` 的內容整份貼上、發布。

### 3. 建立第一位伙委帳號

Firestore 規則設計上，沒有人能透過網頁自己把自己設成伙委（避免有人冒充），
所以第一位伙委需要手動開通：

1. 打開 `index.html`（本機打開即可測試，不用等部署），在「伙委登入」輸入你的
   Email 和自訂密碼，按「還沒有帳號？註冊新帳號」→ 會建立 Firebase Auth 帳號。
2. 到 Firebase Console → Firestore Database → 開始收集資料 → 新增集合
   `staff` → 文件 ID 填你剛剛註冊的 **Email**（例如
   `wang0919451952@gmail.com`）→ 隨便加一個欄位，例如 `name`（字串，填你的
   稱呼）→ 儲存。
3. 回到網頁重新整理，就會以伙委身份登入。

之後這位伙委可以在網頁右上角齒輪⚙️「座號設定」裡的「新增其他伙委」欄位，
直接用 email 新增其他伙委（前提是對方要先用該 email 登入過一次建立帳號）。

這是唯一一個需要手動進 Firebase Console 的步驟（第一位伙委的開通），因為
Firestore 規則設計上不允許任何人自己把自己設成伙委，避免有人冒充。這一步
你已經做過了，之後不管是新增伙委、設定座號名冊、還是指定外食訂購人，
全部都可以在網頁裡完成，不用再碰 Firebase Console。

### 4. 設定班級座號、建立名冊

伙委登入後，點右上角⚙️，填入班級總座號數、是否含「師」，按儲存。這一步會
自動把 `students/{座號}` 名冊寫進 Firestore——同學要能登入寫入自己的資料，
座號必須先出現在這份名冊裡。

同一個設定面板裡也可以指定「外食訂購人」：輸入座號、按設定，該座號的帳號
接下來 14 天內就能在網頁裡完整管理外食訂購功能（建立活動、編輯菜單、修改
任何人的訂單），時間到了規則會自動失效；也可以隨時點清單裡的 × 提前移除。

## 部署到 GitHub Pages

1. 建一個新的 GitHub repo（public 或 private 皆可，Pages 免費方案 public
   repo 較單純）。
2. 把 `index.html` 放進 repo 根目錄（或 `/docs` 資料夾，看你 Pages 設定）。
   `firestore.rules` 等檔案也可以一起放進 repo，方便之後用 CLI 更新規則，
   但它們不影響網頁本身。
3. GitHub repo → Settings → Pages → Source 選擇你放 `index.html` 的分支/
   資料夾 → 儲存，會拿到一個網址，例如：
   `https://你的帳號.github.io/repo名稱/`

### ⚠️ 重要：把 Pages 網域加進 Firebase 已授權網域

Firebase Authentication 只允許「已授權網域」呼叫登入 API，否則會出現
`auth/unauthorized-domain` 錯誤：

Firebase Console → Authentication → Settings → Authorized domains →
新增網域 → 貼上你的 GitHub Pages 網域（只要網域，不用完整路徑），例如：

```
你的帳號.github.io
```

## 已知限制 / 後續可以強化的地方

- **密碼無法在前端重設**：同學或伙委忘記密碼時，需要管理員到 Firebase
  Console → Authentication 手動重設密碼或刪除帳號讓對方重新註冊。
- 目前「讀取」對所有已登入使用者是開放的（伙委和同學都能看到彼此的退餐/
  訂購狀況），只有「寫入」被規則限制在本人資料。如果需要同學之間互相看不到，
  請再告訴我，我可以幫你把讀取規則也收緊。
