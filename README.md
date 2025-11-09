# 服務通 API (Standing Backend)

台北市建築風險分析與道路風險評估 API 服務

---

## 目錄

- [功能特色](#功能特色)
- [資料來源](#資料來源)
- [系統架構](#系統架構)
- [API 端點](#api-端點)
- [安裝指南](#安裝指南)
- [使用說明](#使用說明)
- [資料格式](#資料格式)
- [API 使用範例](#api-使用範例)

---

## 功能特色

### 1. **六角格風險分析**
- 使用 H3 地理空間索引系統
- 整合建築高度與即時風速資料
- 30,301 個六角格覆蓋整個台北市
- 5 級風險分類系統

### 2. **道路風險評估**
- 61,453 條道路的風險等級分析
- 即時從 OpenStreetMap 抓取最新道路資料
- 自動快取機制,首次計算後秒回
- 支援按風險等級篩選

---

## 📊 資料來源

### Overpass API
- **端點**: `http://overpass-api.de/api/interpreter`
- **查詢範圍**: 台北市 (24.95°N-25.20°N, 121.40°E-121.70°E)
- **道路類型**: motorway, trunk, primary, secondary, tertiary, residential, service, unclassified

### 建築資料
- **來源**: 台北市開放資料平台
- **資料欄位**: 建築高度、座標

### 氣象資料
- **來源**: 中央氣象署 API
- **資料欄位**: 風速、風向

---

## 系統架構

```
Standing_backend/
├── app.py                   # FastAPI 主程式
├── router/                  # API 路由
│   ├── map.py              # 地圖與風險分析 API
│   ├── issue.py            # 障礙回報 API
│   ├── news.py             # 警廣新聞 API
│   └── wind.py             # 風速資料 API
├── functions/               # 業務邏輯函數
│   ├── mapData_proccess.py # 地圖資料處理
│   ├── report.py           # 障礙回報資料庫操作
│   ├── police.py           # 警廣新聞爬蟲
│   ├── windspeed.py        # 即時風速資料
│   └── weather_future.py   # 未來風速預測
├── util/                    # 工具函數
│   ├── config.py           # 環境變數配置
│   └── nowtime.py          # 時間工具函數
├── dataStore/              # 資料儲存目錄
│   ├── hexgrid_data.json   # 六角格資料
│   └── taipei_roads.json   # 道路資料
├── .env                    # 環境變數設定 (需自行建立)
├── requirements.txt        # Python 依賴套件
└── Dockerfile             # Docker 容器設定
```

---

## 📁 模組說明

### Router (API 端點)

#### `router/map.py` - 地圖與風險分析
- `GET /map/update_hexgrid_data` - 更新六角格資料
- `GET /map/analyze_road_risk` - 分析道路風險等級
- `POST /map/clear_road_risk_cache` - 清除快取

#### `router/issue.py` - 障礙回報
- `POST /issue/create` - 建立障礙回報
- `GET /issue/getByTime` - 取得最近 N 小時的回報
- `GET /issue/getByStatus` - 依狀態查詢回報

#### `router/news.py` - 警廣新聞
- `GET /news/police` - 取得警廣即時路況新聞
- `GET /news/police_local` - 取得本地快取新聞
- `GET /news/police_opendata` - 取得開放資料新聞

#### `router/wind.py` - 風速資料
- `GET /wind/` - 取得即時風速資料
- `GET /wind/future` - 取得未來風速預測

### Functions (業務邏輯)

#### `functions/mapData_proccess.py`
- `TaipeiDataManager` - 台北市資料管理器
- `update_hexgrid_data()` - 更新六角格風險資料
- 整合建築高度與風速計算組合值

#### `functions/report.py`
- `insert_issue()` - 寫入障礙回報到 Supabase
- `read_issues_by_time()` - 讀取指定時間範圍的回報
- `read_issues_by_status()` - 讀取指定狀態的回報

#### `functions/police.py`
- `police_news_data()` - 爬取警廣即時路況
- `opendata_news_data()` - 取得開放資料新聞

#### `functions/windspeed.py`
- `windspeed_taipei()` - 取得台北市即時風速

#### `functions/weather_future.py`
- `windspeed_taipei_future()` - 取得未來風速預測

### Util (工具函數)

#### `util/config.py`
- `Env` 類別 - 環境變數管理
- 支援變數：
  - `CWA_API_KEY` - 中央氣象署 API 金鑰
  - `SUPABASE_URL` - Supabase 資料庫 URL
  - `SUPABASE_KEY` - Supabase API 金鑰
  - `DOCS_USERNAME` / `DOCS_PASSWORD` - API 文件認證
  - `PORT` - 服務埠號（預設 7860）

#### `util/nowtime.py`
- `getTaiwanTime()` - 取得台灣當前時間 (datetime)
- `getTaiwanTimeString()` - 取得台灣當前時間字串
- 自動處理時區轉換 (Asia/Taipei)

---

## API 端點

### 🗺️ 地圖與風險分析 (`/map`)

#### 1. 更新六角格資料
```http
GET /map/update_hexgrid_data?background=false
```

**參數：**
- `background` (bool): 是否在背景執行（預設: false）

**回應範例：**
```json
{
  "success": true,
  "message": "六角格資料更新成功",
  "data": {
    "total_buildings": 217414,
    "total_weather_stations": 31,
    "update_time": "2025-11-09T02:27:33",
    "resolutions": {
      "res_10": 30301
    }
  }
}
```

---

### 2. 分析道路風險
```http
GET /map/analyze_road_risk?risk_level=5&use_cache=true
```

**參數：**
- `risk_level` (int, 可選): 指定風險等級 1-5
  - `1`: 極低風險 (< 10.8)
  - `2`: 低風險 (10.8-12.5)
  - `3`: 中風險 (12.5-14.4)
  - `4`: 高風險 (14.4-16.2)
  - `5`: 極高風險 (≥ 16.2)
- `use_cache` (bool): 是否使用快取（預設: true）

**回應範例（指定 risk_level=5）：**
```json
{
  "success": true,
  "message": "道路風險分析完成 (使用快取) - 極高風險",
  "cached": true,
  "data": {
    "risk_level": 5,
    "risk_level_name": "極高風險",
    "count": 449,
    "roads": [
      {
        "name": "重慶北路一段",
        "start": {"lat": 25.0123, "lng": 121.4567},
        "end": {"lat": 25.0234, "lng": 121.4678}
      }
    ]
  },
  "statistics": {
    "total_roads_analyzed": 61453,
    "level_1_count": 17972,
    "level_2_count": 15234,
    "level_3_count": 12876,
    "level_4_count": 9541,
    "level_5_count": 449,
    "unknown_count": 5381
  }
}
```

**回應範例（不指定 risk_level）：**
```json
{
  "success": true,
  "message": "道路風險分析完成",
  "cached": false,
  "data": {
    "level_1": {
      "risk_level": 1,
      "risk_level_name": "極低風險",
      "count": 17972,
      "roads": [...]
    },
    "level_2": {...},
    "level_3": {...},
    "level_4": {...},
    "level_5": {...}
  },
  "statistics": {...}
}
```

---

### 📋 障礙回報 (`/issue`)

#### 1. 建立障礙回報
```http
POST /issue/create
```

**參數：**
- `address` (string): 障礙地點
- `obstacle_type` (string): 障礙類型
- `description` (string): 詳細描述
- `modtified_userid` (string, 可選): 使用者 ID（預設: "visitor"）

**回應：**
```json
"回報成功"
```

---

#### 2. 取得最近回報
```http
GET /issue/getByTime?hours=24
```

**參數：**
- `hours` (int): 查詢最近幾小時的回報（預設: 24）

**回應範例：**
```json
[
  {
    "id": 1,
    "address": "台北市中正區重慶南路一段",
    "type": "路面坑洞",
    "description": "路面有大坑洞，影響行車安全",
    "time": "2025-11-09T14:30:00",
    "modtified_userid": "user123",
    "status": "Unsolved"
  }
]
```

---

#### 3. 依狀態查詢回報
```http
GET /issue/getByStatus?status=Unsolved
```

**參數：**
- `status` (string): 回報狀態（預設: "Unsolved"）
  - `Unsolved` - 未解決
  - `Solved` - 已解決
  - `Processing` - 處理中

---

### 📰 警廣新聞 (`/news`)

#### 1. 即時路況新聞
```http
GET /news/police
```

**回應範例：**
```json
{
  "news": [
    {
      "title": "台北市忠孝東路四段車流量大",
      "time": "14:30",
      "location": "忠孝東路四段"
    }
  ]
}
```

---

#### 2. 本地快取新聞
```http
GET /news/police_local
```

---

#### 3. 開放資料新聞
```http
GET /news/police_opendata
```

---

### 🌬️ 風速資料 (`/wind`)

#### 1. 即時風速
```http
GET /wind/
```

**回應範例：**
```json
{
  "stations": [
    {
      "station_name": "臺北",
      "wind_speed": 2.5,
      "wind_direction": "東北風",
      "observation_time": "2025-11-09T14:00:00"
    }
  ]
}
```

---

#### 2. 未來風速預測
```http
GET /wind/future
```

**回應範例：**
```json
{
  "forecast": [
    {
      "time": "2025-11-09T15:00:00",
      "wind_speed": 3.0,
      "wind_direction": "東北風"
    }
  ]
}
```

---

### 🗺️ 地圖與風險分析 (續)

#### 3. 清除快取
```http
POST /map/clear_road_risk_cache
```

**使用時機：**
- 六角格資料更新後
- 道路資料更新後
- 需要重新計算風險值時

**回應範例：**
```json
{
  "success": true,
  "message": "快取已清除，下次調用將重新載入資料並計算"
}
```

---

## 📦 安裝指南

### 1. 環境需求
- Python 3.11+
- pip

### 2. 克隆專案
```bash
git clone https://github.com/Ynn622/Standing_backend.git
cd Standing_backend
```

### 3. 安裝依賴
```bash
pip install -r requirements.txt
```

### 4. 環境變數設定

建立 `.env` 檔案（參考 `.env.example`）：

```bash
# 中央氣象署 API
CWA_API_KEY=your_cwa_api_key_here

# Supabase 資料庫（用於障礙回報）
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your_supabase_anon_key

# API 文件認證（可選）
DOCS_USERNAME=admin
DOCS_PASSWORD=your_password

# 服務設定
PORT=7860
RELOAD=true
```

**取得 API 金鑰：**
- **中央氣象署**: https://opendata.cwa.gov.tw/
- **Supabase**: https://supabase.com/ （免費方案可用）

### 5. 必要套件
```
# API 框架
fastapi
uvicorn[standard]

# 地理空間
h3

# HTTP 請求
requests

# 資料庫
supabase

# 環境變數
python-dotenv

# 網頁爬蟲
beautifulsoup4
lxml
cloudscraper

# 其他
python-multipart
```

---

## 💻 使用說明

### 啟動服務

#### 方式 1: 直接啟動
```bash
cd Standing_backend
uvicorn app:app --reload --host 0.0.0.0 --port 8000
```

#### 方式 2: 使用環境變數
```bash
# .env 檔案中設定 PORT=7860
python app.py
```

#### 方式 3: Docker 啟動
```bash
docker build -t standing-backend .
docker run -p 8000:7860 --env-file .env standing-backend
```

---

## 💻 使用說明

### 啟動服務
```bash
cd Standing_backend
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### 訪問 API 文件
開啟瀏覽器訪問：
- **Swagger UI**: http://localhost:8000/docs
- **ReDoc**: http://localhost:8000/redoc

### 首次使用流程

1. **設定環境變數**
```bash
cp .env.example .env
# 編輯 .env，填入必要的 API 金鑰
```

2. **更新六角格資料**（如果沒有）
```bash
curl http://localhost:8000/map/update_hexgrid_data
```

3. **測試障礙回報**
```bash
curl -X POST "http://localhost:8000/issue/create" \
  -H "Content-Type: application/json" \
  -d '{
    "address": "台北市中正區重慶南路一段",
    "obstacle_type": "路面坑洞",
    "description": "路面有大坑洞",
    "modtified_userid": "test_user"
  }'
```

4. **分析道路風險**（首次會自動下載道路資料）
```bash
# 首次調用：約 90-120 秒（下載 + 計算）
curl http://localhost:8000/map/analyze_road_risk

# 第二次調用：< 100ms（使用快取）
curl http://localhost:8000/map/analyze_road_risk
```

5. **查詢即時風速**
```bash
curl http://localhost:8000/wind/
```

6. **查詢警廣新聞**
```bash
curl http://localhost:8000/news/police
```

7. **查詢特定風險等級**
```bash
# 只查詢極高風險道路
curl http://localhost:8000/map/analyze_road_risk?risk_level=5
```

---

## 📊 資料格式

### 六角格資料結構
```json
{
  "metadata": {
    "update_time": "2025-11-09T02:27:33",
    "total_buildings": 217414,
    "total_weather_stations": 31
  },
  "resolutions": {
    "res_10": {
      "resolution": 10,
      "total_hexagons": 30301,
      "hexagons": [
        {
          "h3_index": "8a4ba1d16587fff",
          "max_height": 33.15,
          "combined_value": 12.13
        }
      ]
    }
  }
}
```

### 道路資料結構
```json
{
  "timestamp": "2025-11-09 03:19:45",
  "total_roads": 61453,
  "roads": [
    {
      "id": 123456789,
      "type": "primary",
      "name": "重慶北路一段",
      "geometry": [
        {"lat": 25.0123, "lng": 121.4567},
        {"lat": 25.0124, "lng": 121.4568}
      ]
    }
  ]
}
```

---

## 📝 API 使用範例

### Python
```python
import requests

# 查詢極高風險道路
response = requests.get(
    'http://localhost:8000/map/analyze_road_risk',
    params={'risk_level': 5}
)
data = response.json()

for road in data['data']['roads']:
    print(f"{road['name']}: {road['start']} -> {road['end']}")
```

### JavaScript
```javascript
// 查詢所有風險等級
fetch('http://localhost:8000/map/analyze_road_risk')
  .then(res => res.json())
  .then(data => {
    console.log('總道路數:', data.statistics.total_roads_analyzed);
    console.log('極高風險道路:', data.data.level_5.count);
  });
```

### cURL
```bash
# 更新六角格資料
curl -X GET "http://localhost:8000/map/update_hexgrid_data"

# 查詢中風險道路
curl -X GET "http://localhost:8000/map/analyze_road_risk?risk_level=3"

# 清除快取
curl -X POST "http://localhost:8000/map/clear_road_risk_cache"
```
