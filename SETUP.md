# 旅行手帖：開啟多人即時共編（Firebase 設定）

大約 15 分鐘就能設定好，只要做一次，之後每趟旅行都會沿用。Firebase 的免費方案（Spark）對朋友出遊的用量綽綽有餘。

---

## 1. 建立 Firebase 專案

1. 打開 <https://console.firebase.google.com>，用你的 Google 帳號登入。
2. 按「新增專案」，名稱例如 `trip-desk`。Google Analytics 可以關掉。

## 2. 開啟 Firestore 資料庫

1. 左側選單 → **Build → Firestore Database** → 「建立資料庫」。
2. 位置選 **asia-east1（台灣）**。
3. 模式先選「正式版模式（production mode）」，第 5 步會貼上規則。

## 3. 開啟登入方式

左側 **Build → Authentication** → 「開始使用」→「Sign-in method」，啟用以下兩種：

- **匿名（Anonymous）**：旅伴打開連結就能自動登入，不必註冊。
- **Google**：只有主辦人（你）會用。

接著到 **Authentication → Settings → 授權網域（Authorized domains）**，新增你的網站網域，例如：

```
你的帳號.github.io
```

## 4. 取得設定碼，貼進 index.html

1. 專案首頁按「</>」（新增網頁應用程式），名稱隨意，不用勾 Hosting。
2. 畫面會出現一段 `const firebaseConfig = { ... }`，複製大括號 `{ ... }` 這一段。
3. 打開 `index.html`，搜尋 `window.FIREBASE_CONFIG = null;`，換成：

```js
window.FIREBASE_CONFIG = {
  apiKey: "……",
  authDomain: "trip-desk-xxxx.firebaseapp.com",
  projectId: "trip-desk-xxxx",
  storageBucket: "……",
  messagingSenderId: "……",
  appId: "……"
};
```

> 這段設定碼本來就是給瀏覽器用的公開資訊，放在 GitHub 上沒關係。真正控管權限的是下一步的安全規則。

## 5. 貼上安全規則

**Firestore Database → 規則（Rules）**，整段換成以下內容，然後按「發布」：

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /trips/{tripId} {
      // 有連結的人都能讀；只能列出自己建立的旅程
      allow get: if request.auth != null;
      allow list: if request.auth != null && resource.data.ownerUid == request.auth.uid;
      // 只有用 Google 登入的主辦人能建立旅程、修改旅程設定
      allow create: if request.auth != null
        && request.auth.token.firebase.sign_in_provider != 'anonymous'
        && request.resource.data.ownerUid == request.auth.uid;
      allow update: if request.auth != null
        && resource.data.ownerUid == request.auth.uid
        && request.resource.data.ownerUid == resource.data.ownerUid;
      allow delete: if request.auth != null && resource.data.ownerUid == request.auth.uid;

      // 景點、行程、分帳、待辦、航班、住宿：拿到連結的旅伴都能編輯
      match /{col}/{docId} {
        allow read: if request.auth != null;
        allow write: if request.auth != null
          && col in ['spots','stops','days','flights','transfers','hotels','todoGroups','todos','expenses']
          && exists(/databases/$(database)/documents/trips/$(tripId));
      }
    }
  }
}
```

## 6. 上傳到 GitHub

repo 裡只需要 **一個** `index.html`（加上這份 SETUP.md）。推上去並開啟 GitHub Pages 後，網址會像：

```
https://你的帳號.github.io/trips/
```

---

## 日常怎麼用

| 誰 | 做什麼 |
|---|---|
| 主辦人（你） | 打開網址 → 後台 →「主辦人登入（Google）」→「把目前旅程上傳成共編旅程」。上傳後網址會變成 `…/trips/?t=ha-noi-x8k2pq`，這就是分享連結 |
| 主辦人 | 下一趟旅行：後台 →「建立新的雲端旅程」→ 填城市和日期 → 得到新的 `?t=` 連結 |
| 旅伴 | 點你傳的連結就能直接編輯，不用登入。第一次打開時選「我是誰」，記帳就會自動帶入自己 |

**權限分工**

- **大家都能改**：景點、每日行程（時間／時段／候選）、分帳、待辦、航班、接送、住宿。
- **只有主辦人能改**：旅程名稱與日期、城市、幣別與匯率、配色、頭像風格、景點類別、旅伴名單、建立／刪除旅程。

**小提醒**

- 拿到連結的人都能編輯，連結只傳給這趟的旅伴就好。
- 沒網路時也能看、也能記帳，恢復連線後會自動同步。
- 兩個人同時改同一張卡片時，以最後存檔的人為準。不同卡片可以同時改，互不影響。
- 如果 `FIREBASE_CONFIG` 保持 `null`，網頁會用本機模式運作，資料只存在各自的手機裡。
