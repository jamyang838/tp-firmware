# tp-firmware

TP 觸控控制板韌體雲端更新空間（IlitekFwTool 讀取）。

| 檔案 | 用途 |
|---|---|
| `manifest.json` | 更新程式讀取的清單，決定目前要更新到哪一版 |
| `manifest_final.json` | 預備清單：指向 20250716_Final |
| `manifest_enlarge.json` | 預備清單：指向 20260828_Enlarge |
| `TP1210UWDA-BCG_V2_DD_Final_20250716.hex` | 韌體 20250716_Final |
| `TP1210UWDA-BCG_V2_DD_Enlarge_20260828.hex` | 韌體 20260828_Enlarge |

切換版本：把 `manifest_final.json` 或 `manifest_enlarge.json` 的內容複製到 `manifest.json` 後 commit。
