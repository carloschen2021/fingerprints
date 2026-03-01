# 部署指南：本地 → GitHub → Render

## 前置條件

- 已安裝 Git 並設定 GitHub 帳號
- 已在 [Render.com](https://render.com) 建立 Web Service 並連結此 GitHub repo
- Render 設定監聽 `main` 分支（預設自動部署）

---

## 日常更新流程

### 1. 修改程式碼（本地開發）

啟動本地伺服器測試：
```powershell
$env:PORT=3002; npm run dev
```
瀏覽器開啟 `http://localhost:3002` 驗證功能正常。

### 2. 提交更改到 Git

```powershell
# 查看哪些檔案有修改
git status

# 加入所有修改的檔案
git add .

# 建立 commit（描述這次改了什麼）
git commit -m "fix: 修復登入驗證碼問題"
```

### 3. 推送到 GitHub

```powershell
git push origin main
```

### 4. Render 自動部署

推送到 `main` 後，Render 會**自動觸發重新部署**，約需 1-3 分鐘。

確認部署狀態：
1. 登入 [Render Dashboard](https://dashboard.render.com)
2. 選擇 `fingerprints` Service
3. 點擊 **Deploys** 頁籤，確認最新部署狀態為 `Live` ✅

---

## 如果你在 feature 分支開發

```powershell
# 切換到 main 分支
git checkout main

# 合併 feature 分支
git merge feature/你的分支名稱

# 推送到 GitHub
git push origin main
```

---

## 本專案資訊

| 項目 | 值 |
|------|-----|
| GitHub Repo | https://github.com/carloschen2021/fingerprints |
| Render URL | https://fingerprints-xb9k.onrender.com |
| Render Service ID | srv-d6hev67kijhs73ffed10 |
| 部署分支 | `main` |
| 本地預設埠 | 3002（3000 被系統佔用） |

---

## 注意事項

> ⚠️ **資料庫持久性**：Render 免費方案使用 SQLite 存在伺服器暫時檔案系統，**每次重新部署後資料庫會重置**（用戶帳號清空）。若需要持久儲存，請改用 Render 提供的 PostgreSQL 資料庫服務。

> ⚠️ **port 3000 被系統佔用**：本機的 AMAgent 服務占用了 3000 port，本地開發請一律使用 3002：`$env:PORT=3002; npm run dev`
