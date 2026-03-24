# ARIA v3.0

ARIA v3.0 是一套用於鳳凰颱風情境壓力測試的動態受災衝擊評估系統。主程式以 Jupyter Notebook 形式實作，輸出 `ARIA_v3_Fungwong.html` 互動式地圖，整合雨量站、避難所靜態風險與動態暴雨影響範圍。

## Repo Overview

本作業以 `ARIA_v3.ipynb` 為主體，完成以下流程：

- 讀取 `.env` 切換 `LIVE / SIMULATION` 模式
- 呼叫 CWA O-A0002-001 API，或 fallback 到 `fungwong_202511.json`
- 正規化 API / CoLife JSON 結構差異
- 整合避難所 `river_distance`、`max_slope`、`terrain_risk`
- 在 `EPSG:3826` 下建立 5km 雨量影響範圍並做 `gpd.sjoin()`
- 計算 `CRITICAL / URGENT / WARNING / SAFE` 動態風險
- 輸出 `ARIA_v3_Fungwong.html` Folium 儀表板

## Submission Notes

本 repo 對應第 5 週作業要求，提交內容如下：

- `ARIA_v3.ipynb`: 完整分析流程與 Captain's Log
- `ARIA_v3_Fungwong.html`: 互動式監測地圖
- `README.md`: 專案說明與 AI Diagnostic Log

如果助教直接檢查規格，請優先看：

- `.env` 控制模式、門檻與資料路徑
- `.gitignore` 已排除 `.env`
- Notebook 中有 `normalize_cwa_json()`、fallback 機制、CRS assert、動態風險邏輯與 Folium 圖層

## Deliverables

- `ARIA_v3.ipynb`
- `ARIA_v3_Fungwong.html`
- `README.md`

## Environment

所有可調設定均放在 `.env`：

- `APP_MODE`: `LIVE` 或 `SIMULATION`
- `CWA_API_KEY`: CWA API 金鑰
- `GEMINI_API_KEY`: Gemini API 金鑰，可留空
- `SIMULATION_DATA`: 本地情境 JSON 路徑
- `TARGET_COUNTY`: 預設 `花蓮縣`
- `STATION_COUNTIES`: 雨量站顯示與分析範圍，預設 `花蓮縣,宜蘭縣`
- `BUFFER_METERS`: 雨量站影響半徑，預設 `5000`
- `CRITICAL_RAIN_MM`: `CRITICAL` 門檻
- `WARNING_RAIN_MM`: `URGENT/WARNING` 門檻
- `RIVER_RISK_HIGH_METERS`: `river_risk = HIGH` 的距河門檻
- `RIVER_RISK_MEDIUM_METERS`: `river_risk = MEDIUM` 的距河門檻
- `STATION_CONTEXT_BUFFER_METERS`: 站點保留範圍，避免只取縣界內測站而漏掉鄰近強降雨

`.gitignore` 已排除 `.env`，避免 API keys 被提交到 GitHub。

## Data Notes

目前工作區的 `terrain_risk_assessment.geojson` 含有：

- `river_distance`
- `mean_elevation`
- `max_slope`
- `risk_level`

但未直接提供 `river_risk` 欄位。因此本次 notebook 會先由 `river_distance` 推導 `river_risk`：

- `<= RIVER_RISK_HIGH_METERS` → `HIGH`
- `RIVER_RISK_HIGH_METERS < distance <= RIVER_RISK_MEDIUM_METERS` → `MEDIUM`
- `> RIVER_RISK_MEDIUM_METERS` → `LOW`

如果你要完全對齊 Week 3 課堂版本，只要把 `.env` 內的 `RIVER_RISK_HIGH_METERS` 與 `RIVER_RISK_MEDIUM_METERS` 改成課堂門檻即可，不需要修改 notebook 程式。

## How To Run

1. 建立並啟用具備 `geopandas`, `folium`, `python-dotenv`, `requests`, `shapely` 的 Python 環境
2. 視需要更新 `.env`
3. 開啟 `ARIA_v3.ipynb`
4. 依序執行所有 cells
5. 產出 `ARIA_v3_Fungwong.html`

## AI Diagnostic Log

### 1. `-998` 雨量值造成符號顏色異常

問題：CWA 以 `-998` 表示缺值，若直接進圖層渲染，會被當成極端負值，導致 CircleMarker 顏色分級失真，HeatMap 也會被污染。

修正：在 `normalize_cwa_json()` 內先統一轉數值，再於建立 GeoDataFrame 前過濾 `-998` 與 `NaN`。這樣後續地圖符號、緩衝區與動態風險運算都只基於有效測站。

### 2. `sjoin` 結果為空

問題：避難所資料分析時使用 `EPSG:3826`，而雨量站初始為 `EPSG:4326`。如果直接做 `sjoin`，空間連接會回傳空集合。

修正：在建立 buffer 前先把測站轉為 `EPSG:3826`，避難所也強制轉成 `EPSG:3826`，並在 `sjoin` 前加入：

```python
assert str(shelters_3826.crs) == str(rain_buffers.crs), "CRS MISMATCH!"
```

這可以在空間分析前即時攔下 CRS 對不齊的情況。

在輸出 Top 10 表格時，遇到 KeyError，原因是先行篩選了欄位導致 sort_values 找不到排序依據。後來透過 AI 輔助，將邏輯改為『先對完整 DataFrame 排序，再篩選顯示欄位』，成功解決問題。

### 3. Folium 經緯度順序

問題：Folium 一律使用 `[latitude, longitude]`。若誤用 `[longitude, latitude]`，點位會偏移到錯誤位置。

修正：所有 `CircleMarker` 與 `Marker` 都統一以 `location=[row.geometry.y, row.geometry.x]` 寫入，避免經緯度反轉。

## Final Checklist

交作業前請確認：

- `ARIA_v3.ipynb` 可以從第一格順序執行
- `ARIA_v3_Fungwong.html` 可正常打開，且有 HeatMap、雨量站、避難所風險圖層
- `.env` 存在於本機，但不會被 Git 追蹤
- GitHub repo 中沒有 API key 明碼
- README 已包含 AI Diagnostic Log
- Notebook 內有 `Captain's Log`
- `TARGET_COUNTY` 設為此次提交要展示的縣市
- `river_risk` 門檻已改成你課堂版本

## GitHub Homepage Paragraph

ARIA v3.0 is a dynamic disaster impact auditing notebook for Typhoon Fung-wong stress testing. The project combines CWA rainfall stations, Week 3 river-distance shelter context, Week 4 terrain-risk indicators, CRS-safe spatial joins, and a Folium dashboard to identify which shelters are currently at greatest risk. The notebook supports both LIVE API mode and SIMULATION mode with automatic fallback, and exports the final operational map as `ARIA_v3_Fungwong.html`.
