# GitHub Pages + Firebase 安全角色管理範例

這是一個**不需自架後端、不需打包**的靜態網站。使用 GitHub Pages 託管 HTML/CSS/JavaScript，Firebase Authentication 驗證帳密，Firestore Security Rules 控制資料存取。

功能：

- 使用 Email + Password 註冊、登入、登出。
- 新註冊的帳號一律建立為 `student`。
- Student 可查看公告；Admin 可新增、修改、刪除公告。
- **只有初始開發者的 Firebase UID** 可查看完整使用者清單、授予或撤銷 Admin。
- 其他 Admin **不能**變更使用者角色。
- 完全不使用 Cloud Functions 或 Cloud Storage；維持 Firebase Spark 免費方案。

## 0. 重要前提

這是實際權限控管範例，不是只隱藏按鈕。`firestore.rules` 必須在 Firebase 正式發布後才具有保護作用。網站的 HTML/JS 仍然公開，**不要將密碼、金鑰、學生私人檔案、機密教材放在 GitHub Pages 上**。

此範例適合小型課程站、班級公告。當需求擴充成付款、檔案上傳、複雜審核、處理機密資訊時，須重新評估架構。

## 1. 建立 Firebase 專案（只選 Spark）

1. 開啟 [Firebase Console](https://console.firebase.google.com/) 並建立新專案。
2. **保持 Spark (No-cost)**，不要連結 Cloud Billing，也不要升級 Blaze。
3. 進入 **Build → Authentication → Sign-in method**，啟用 **Email/Password**。
4. 進入 **Build → Firestore Database**，建立資料庫，選擇 **Production mode（正式模式）**。Firestore Standard 即可。
5. 在 Authentication → Settings → **Authorized domains** 新增你的 GitHub Pages 主機名稱（例如 `andyyoung1219.github.io`，**不包含** `https://` 和 repo 路徑）。若本機測試登入受到網域限制，再新增 `localhost`。

## 2. 先建立初始開發者 Admin（最重要）

因為所有新註冊者預設為 Student，所以第一位 Admin 必須**從 Firebase Console 手動建立**。

1. 在 **Authentication → Users → Add user**，新增你自己的 Email + Password 帳號。
2. 找到這位使用者的 **User UID**，例如 `aBc123xyz...`，複製它。
3. 在 **Firestore Database → Data**，新增 collection：`users`。
4. 新增文件，**文件 ID 必須等於上一步的 UID**，欄位如下：

   | 欄位 | 類型 | 值 |
   | --- | --- | --- |
   | `displayName` | string | 你的名字 |
   | `email` | string | 你的 Email |
   | `role` | string | `admin` |
   | `createdAt` | timestamp | 建立當下時間（可選，但建議） |

5. 使用文字編輯器打開 **`firestore.rules`**，將其中的 `PASTE_DEVELOPER_UID` 換成你的 UID。
6. 到 **Firestore Database → Rules**，將整份 `firestore.rules` 複製進去並按 **Publish**。

> 保留自己的 Firebase Console 存取權。若忘記管理員密碼，可在 Firebase Auth 重設；若 UID 改變，須同步更新兩個位置的 UID。

## 3. 填入網站設定

1. 在 Firebase Console → **Project settings（齒輪）→ General → Your apps**，建立 **Web app（`</>`）**。
2. 複製顯示的 `firebaseConfig`，貼入專案的 `firebase-config.js`。最少要有：

   ```js
   export const firebaseConfig = {
     apiKey: "...",
     authDomain: "你的專案.firebaseapp.com",
     projectId: "你的專案",
     appId: "..."
   };

   export const DEVELOPER_UID = "你自己的 Firebase Auth UID";
   ```

3. 注意：Firebase 的**網頁設定物件不是管理員私密金鑰**，可以在前端使用；絕不能將服務帳戶 JSON、`private_key` 或密碼寫進任何前端檔案。
4. `firebase-config.js` 和 `firestore.rules` 內的 Developer UID **必須一致**。前者只決定前端顯示什麼；後者才是安全授權依據。

## 4. 先在本機測試

此網站使用 ES modules，不能直接雙擊 `index.html` 用 `file://` 開啟。

進入專案目錄，在 PowerShell 執行：

```powershell
python -m http.server 5500
```

再開啟 <http://localhost:5500>。

1. 用你在 Firebase Console 建立的開發者帳號登入，畫面應顯示 `Developer · Admin`，並出現「使用者權限管理」。
2. 登出後，用另一個 Email 註冊，這個新帳號應只具有 Student 權限。
3. 再登入開發者帳號，在使用者列表找到該學生，按「授予 Admin」。
4. 用學生帳號重新登入或保持在線，介面應更新為 Admin，能新增公告。
5. 撤銷 Admin 後，該帳號再次寫入公告應被 Security Rules 拒絕。

> 為避免誤授權給冒名者，本範例的註冊者電子郵件**尚未強制完成驗證**。開發者在授予 Admin 前，應先透過可信管道確認該 UID／帳號確實屬於目標使用者。正式對外使用，建議補上 Email verification 或使用受信任的 Google 登入。

## 5. 部署 GitHub Pages

1. 在 GitHub 建立一個 Repository，例如 `course-hub`（公開倉庫即可）。
2. 將以下檔案推送到 Repository 根目錄：

   ```text
   index.html
   styles.css
   app.js
   firebase-config.js
   firestore.rules
   firebase.json
   README.md
   package.json
   tests/
   ```

3. 進入 Repository → **Settings → Pages**。
4. 選擇 **Deploy from a branch → main → /(root)** 並儲存。
5. 等待部署完成，網站網址通常是 `https://你的帳號.github.io/course-hub/`。
6. GitHub Pages 網頁設定和 Firebase 安全規則是**兩個獨立的部署項目**；修改 `firestore.rules` 後，還要再到 Firebase Console 發布新規則。


官方文件：

- [GitHub Pages 靜態網站限制](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site)
- [Firebase Authentication 密碼登入](https://firebase.google.com/docs/auth/web/password-auth)
- [Firestore 安全規則](https://firebase.google.com/docs/firestore/security/get-started)
- [Firestore 欄位級別規則](https://firebase.google.com/docs/firestore/security/rules-fields)
- [Firebase Spark / Blaze](https://firebase.google.com/docs/projects/billing/firebase-pricing-plans)
