# tp-firmware

TP 觸控控制板韌體雲端更新空間（IlitekFwTool 讀取 `manifest.json`）。

| 檔案 | 用途 |
|---|---|
| `manifest.json` | 更新程式實際讀取的清單 |
| `manifest_final.json` | 預備清單：一律更新為 20250716_Final |
| `manifest_enlarge.json` | 預備清單：一律更新為 20260828_Enlarge |
| `manifest_rules.json` | 預備清單：依板上目前版本決定（Enlarge→Final、Final→Enlarge、未知版本不動） |
| `TP1210UWDA-BCG_V2_DD_Final_20250716.hex` | 韌體 20250716_Final |
| `TP1210UWDA-BCG_V2_DD_Enlarge_20260828.hex` | 韌體 20260828_Enlarge |

## manifest.json 格式（schema 2）

```json
{
  "schema": 2,
  "target": "20250716_Final",
  "rules": [
    { "from": "20260828_Enlarge", "to": "20250716_Final", "note": "說明" },
    { "from": "unknown", "to": "none" }
  ],
  "firmwares": [
    { "version": "20250716_Final", "ic": "ILI2520", "xch": 40, "ych": 24,
      "file": "https://.../xxx.hex", "sha256": "...",
      "blocks": [ { "start": "0x3000", "end": "0x1CFA1", "crc": "0x32BC" } ] }
  ]
}
```

判定方式：
1. 程式用原廠工具讀出板上每個 flash block 的 CRC，和 `firmwares[].blocks` 比對，認出板上目前是哪個版本（都不符則為 `unknown`）。
2. 由上往下找第一條 `from` 等於目前版本（或 `*`）的規則，更新為它的 `to`；`to` 為 `none` 表示不更新。
3. 沒有規則符合時，更新為 `target`；`target` 為 `none` 或空白表示不更新。
4. 目前版本已等於目標版本時不燒錄。
