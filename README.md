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
students/{studentId}         全班學號名冊（伙委在設定頁存檔時自動同步）
staff/{email}                伙委名單，文件 id = 該伙委的登入 email
entries/{date}                每日退餐紀錄（訂餐人數、截圖）
  └ lunch_off/{studentId}      該日午餐退餐名單，一人一份文件
  └ dinner_off/{studentId}     該日晚餐退餐名單，一人一份文件
events/{eventId}              外食訂購活動
  └ menu/{itemId}              菜單品項（名稱、金額）
  └ orders/{studentId}         每人的訂購內容（品項＋數量）
```

之所以把「退餐名單」「訂單」拆成每人一份文件（子集合），是因為 Firestore
規則只能檢查「整份文件」的讀寫權限，沒辦法檢查「陣列裡的某一筆」。拆成
子集合後，才能讓 Firestore 規則真正做到「同學只能寫自己學號那一份」。

## 角色權限（Firestore Rules 強制）

- **伙委（staff）**：帳號 email 存在於 `/staff/{email}`，可讀寫所有 collection。
- **同學（student）**：帳號 email 是 `學號@yourclass.local` 格式，只能寫入
  `lunch_off/dinner_off/orders` 底下「文件 id＝自己學號」的那一份，且該學號
  必須已經在 `/students` 名冊中（伙委存座號設定時會自動建立）。
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
   Email 和自訂密碼，按「登入 / 註冊」→ 會建立 Firebase Auth 帳號。
2. 到 Firebase Console → Firestore Database → 開始收集資料 → 新增集合
   `staff` → 文件 ID 填你剛剛註冊的 **Email**（例如
   `wang0919451952@gmail.com`）→ 隨便加一個欄位，例如 `name`（字串，填你的
   稱呼）→ 儲存。
3. 回到網頁重新整理，就會以伙委身份登入。

之後這位伙委可以在網頁右上角齒輪⚙️「座號設定」裡的「新增其他伙委」欄位，
直接用 email 新增其他伙委（前提是對方要先用該 email 登入過一次建立帳號）。

### 4. 設定班級座號、建立名冊

伙委登入後，點右上角⚙️，填入班級總座號數、是否含「師」，按儲存。這一步會
自動把 `students/{學號}` 名冊寫進 Firestore——同學要能登入寫入自己的資料，
學號必須先出現在這份名冊裡。

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

- **截圖直接存在 Firestore 文件裡**（base64，已壓縮到約 900px 寬、jpeg
  0.65 品質），單一文件上限 1MB，正常使用不會超過，但如果想更穩妥，可以
  改用 Firebase Storage 存圖片、Firestore 只存網址，我可以之後幫你加。
- **密碼無法在前端重設**：同學或伙委忘記密碼時，需要管理員到 Firebase
  Console → Authentication 手動重設密碼或刪除帳號讓對方重新註冊。
- 目前「讀取」對所有已登入使用者是開放的（伙委和同學都能看到彼此的退餐/
  訂購狀況），只有「寫入」被規則限制在本人資料。如果需要同學之間互相看不到，
  請再告訴我，我可以幫你把讀取規則也收緊。
