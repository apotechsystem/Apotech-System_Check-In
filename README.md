# 🏢 APOTECH 員工打卡系統

**悦通系統有限公司** 員工打卡系統 — 基於 GitHub Pages 部署，打卡紀錄自動同步至 Notion。

## 🌐 線上網址

部署後網址：`https://apotechsystem.github.io/attendance/`

---

## ✅ 功能說明

| 功能 | 說明 |
|------|------|
| 📍 GPS 定位打卡 | 距公司 500 公尺內才可打卡（楊梅區） |
| 📶 WiFi 驗證打卡 | 需連接公司授權 WiFi |
| 📷 動態 QR Code | 每 60 秒更新，防止截圖代打 |
| 🌐 IP 驗證打卡 | 限公司授權 IP 位址 |
| 📊 Notion 同步 | 打卡紀錄自動寫入 Notion 資料庫 |

---

## 🚀 部署步驟

### 1. Fork 或建立 Repository

在 GitHub `apotechsystem` 組織下建立名為 `attendance` 的 repository。

### 2. 上傳檔案

將 `index.html` 上傳至 repository 根目錄。

### 3. 開啟 GitHub Pages

- 進入 Repository → **Settings** → **Pages**
- Source 選擇 **Deploy from a branch**
- Branch 選擇 `main`，資料夾選 `/ (root)`
- 點擊 **Save**

### 4. 設定 Notion Integration

在 `index.html` 中找到 `CONFIG` 區塊，填入：

```javascript
const CONFIG = {
  // 公司座標（已預設為楊梅公司地址）
  LAT: 24.9165,
  LNG: 121.1445,
  MAX_DIST: 500,

  // 填入你的 Notion Integration Token
  NOTION_TOKEN: 'secret_xxxxxxxxxx',

  // 打卡紀錄資料庫 ID（已建立）
  NOTION_DB_ID: 'dfac652b-ef76-49ff-b926-ece9fbda4d9f',

  // 填入 Cloudflare Workers Proxy URL（見下方說明）
  NOTION_PROXY: 'https://apotech-checkin.YOUR_WORKER.workers.dev',
};
```

### 5. 建立 Notion Integration Token

1. 前往 https://www.notion.so/my-integrations
2. 點擊「+ New integration」
3. 填入名稱「APOTECH打卡系統」
4. 複製 **Internal Integration Token**（`secret_`開頭）
5. 回到 Notion，在打卡紀錄資料庫右上角「⋯」→「Connections」→ 加入此 Integration

### 6. 建立 Cloudflare Workers（處理 CORS）

GitHub Pages 靜態頁面無法直接呼叫 Notion API（CORS 限制），
需要一個 Proxy。使用免費 Cloudflare Workers：

1. 前往 https://workers.cloudflare.com 建立免費帳號
2. 建立新 Worker，貼入以下程式碼：

```javascript
export default {
  async fetch(request, env) {
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': 'https://apotechsystem.github.io',
          'Access-Control-Allow-Methods': 'POST',
          'Access-Control-Allow-Headers': 'Content-Type',
        }
      });
    }

    const data = await request.json();
    const props = {
      '打卡紀錄': { title: [{ text: { content: data['打卡紀錄'] } }] },
      '員工姓名': { rich_text: [{ text: { content: data['員工姓名'] } }] },
      '員工編號': { rich_text: [{ text: { content: data['員工編號'] } }] },
      '部門': { select: { name: data['部門'] } },
      '打卡類型': { select: { name: data['打卡類型'] } },
      '驗證方式': { select: { name: data['驗證方式'] } },
      '驗證結果': { select: { name: data['驗證結果'] } },
      '打卡時間': { date: { start: data['打卡時間'] } },
      '裝置資訊': { rich_text: [{ text: { content: data['裝置資訊'] || '' } }] },
    };
    if (data['距離公司(公尺)'] !== null && data['距離公司(公尺)'] !== undefined) {
      props['距離公司(公尺)'] = { number: data['距離公司(公尺)'] };
    }
    if (data['GPS緯度']) props['GPS緯度'] = { rich_text: [{ text: { content: data['GPS緯度'] } }] };
    if (data['GPS經度']) props['GPS經度'] = { rich_text: [{ text: { content: data['GPS經度'] } }] };

    const res = await fetch('https://api.notion.com/v1/pages', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${env.NOTION_TOKEN}`,
        'Notion-Version': '2022-06-28',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        parent: { database_id: env.NOTION_DB_ID },
        properties: props,
      }),
    });

    const result = await res.json();
    return new Response(JSON.stringify(result), {
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': 'https://apotechsystem.github.io',
      }
    });
  }
};
```

3. 在 Worker 的 **Settings → Variables** 新增：
   - `NOTION_TOKEN` = 你的 Integration Token
   - `NOTION_DB_ID` = `dfac652b-ef76-49ff-b926-ece9fbda4d9f`

4. 複製 Worker URL，填入 `index.html` 的 `NOTION_PROXY`

---

## 📱 員工使用方式

1. 手機瀏覽器開啟：`https://apotechsystem.github.io/attendance/`
2. 建議加入**手機書籤**（主畫面捷徑）
3. 輸入員工編號、姓名、部門
4. 選擇驗證方式 → 打卡

---

## 📍 公司打卡設定

| 項目 | 數值 |
|------|------|
| 公司地址 | 桃園市楊梅區中山北路1段199巷83號 |
| GPS 緯度 | 24.9165 |
| GPS 經度 | 121.1445 |
| 允許範圍 | 500 公尺 |
| 上班時間 | 09:00 |
| 下班時間 | 18:00 |

---

## 📊 Notion 資料庫

| 資料庫 | 說明 |
|--------|------|
| 👥 員工資料庫 | 員工基本資料（10人） |
| 📋 打卡紀錄 | 每次打卡自動寫入 |
| 📅 請假申請 | 請假申請與審核 |

---

*© 2026 悦通系統有限公司 APOTECH*
