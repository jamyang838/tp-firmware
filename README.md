# tp-firmware：DD TCB Update FW 的雲端 config 與韌體

DD TCB Update FW（V0.2 起）讀取這裡的 `manifest.json` 決定三件事：**每天幾點檢查**、**雲端放在哪裡**、**哪塊板子要更新成哪一版韌體**。
DD 日後維護的只有兩種檔案：**hex（韌體）** 和 **config（`manifest.json`）**。客戶端不需要、也不能改這些設定。

> 目前的位置（GitHub jamyang838/tp-firmware）只是測試與 demo 用的暫時空間，正式雲端確定後依下面「搬家」一節移過去。

## 檔案

| 檔案 | 用途 |
|---|---|
| `manifest.json` | **雲端 config**：程式實際讀取的檔案 |
| `manifest_rules.json` | 依板上版本決定：Enlarge→Final（單向），已是 Final 不動，未知版本不動 |
| `manifest_final.json` | 預備：一律更新為 20250716_Final |
| `manifest_enlarge.json` | 預備：一律更新為 20260828_Enlarge |
| `TP1210UWDA-BCG_V2_DD_Final_20250716.hex` | 韌體 20250716_Final |
| `TP1210UWDA-BCG_V2_DD_Enlarge_20260828.hex` | 韌體 20260828_Enlarge |

## 1. 改每日檢查時間

改 `settings.dailyCheckTime`：

```json
"settings": { "dailyCheckTime": "03:30", "configUrl": "https://.../manifest.json" }
```

- 格式 `HH:mm`，24 小時制，以各電腦的本地時間為準。填 `"off"` 表示關閉每日檢查。
- **生效時間**：各電腦的程式每 60 分鐘讀一次 config，插上板子檢查時也會立即讀。改完約 1 小時內全部生效。
- 程式讀到後會記住，之後即使暫時連不上雲端，也照最後一次讀到的時間檢查。
- 寫錯（例如 `25:00`）時，程式改用預設的 02:00，並在記錄檔留下警告。
- 刪掉這個欄位，程式回到安裝時的預設值（02:00）。
- 到了檢查時間，程式有在執行、而且板子插著，才會執行一次檢查；板子沒插就略過。

## 2. 搬家（更換雲端位置）

程式會記住「目前的雲端位置」。config 裡的 `settings.configUrl` 寫的是 **這份 config 應該在哪裡**，寫成別的網址時，程式就會搬過去。

1. 把整個資料夾（config + 所有 hex）複製到新位置，用瀏覽器確認新網址打得開。
2. **新位置**的 config：`settings.configUrl` 填新位置自己的網址。
3. **舊位置**的 config：`settings.configUrl` 改成新位置的網址。
4. **舊位置保留一段時間**（建議至少一個月），讓所有電腦都至少讀過一次。
   程式讀到新網址、而且新網址讀得到，才會改記新位置；新網址讀不到時，繼續用舊位置，下次再試。
5. 新發的安裝檔，把 `config.json` 的 `manifestUrl`（起始網址）改成新位置。

補充：

- 搬家後如果新位置暫時壞掉，程式會退回安裝時的起始網址讀取。所以起始網址最好是長期不變的網址，例如 DD 自己的網域。
- `firmwares[].file` 可以只寫檔名（相對於 config 的位置），整個資料夾搬家時就不用逐一改網址。
- **已安裝 V0.1 的電腦不支援搬家**，搬家前要先升級到 V0.2。所以目前 `file` 還寫完整網址：V0.1 只認完整網址。

## 3. 新增一版韌體

1. 把 hex 放進資料夾。
2. 在裝有程式的電腦執行（路徑依實際安裝位置）：

   ```
   "%LOCALAPPDATA%\DDTCBUpdateFW\DDTCBUpdateFW.exe" --fw-entry 新韌體.hex 版本名稱 機型名稱
   ```

   會印出一段 JSON（含 SHA-256 與各 block 的 CRC），貼進 `firmwares` 清單。
3. 補上 `xch` / `ych`（觸控通道數，例如 40 / 24；0 表示不檢查）。同一顆 IC 有多個機型時，再填 `moduleName` 或 `fwid`（見第 5 節）。
4. 決定要不要讓板子更新到這一版（見第 4 節）。

`sha256` 建議保留：程式下載後會驗證，檔案不符就不燒錄。沒有 `sha256` 也能用，但每次都會重新下載。

## 4. 決定板子要更新成哪一版

程式先用原廠工具讀出板上韌體，辨識出是清單中的哪一版（都不符就是 `unknown`），再依下面順序決定目標：

1. `rules`：由上往下找第一條 `from` 等於板上版本（或 `*`）的規則，更新為它的 `to`。`to` 是 `none` 表示不更新。
2. `modelTargets`：依板子的機型找目標版本。
3. `target`：預設目標。`none` 或空白表示不更新。

板上已經是目標版本時不會燒錄（**版本不同才安裝**）。

| 用途 | 怎麼設 |
|---|---|
| 正式：所有板子更新到某一版 | `target` 設為該版本，`rules` 留空 |
| 正式：未知版本不要動 | `rules` 放一條 `{ "from": "unknown", "to": "none" }`，`target` 設正式版本 |
| 多個機型各自的正式版本 | `modelTargets` 每個機型一行 |
| 只讓 Enlarge 更新到 Final（目前使用中） | 用 `manifest_rules.json`（`切換到依版本規則.bat`） |

## 5. 機型辨識：IC 型號 + 軟體名稱

程式從板子讀出 IC 型號（例如 `ILI2520`）、觸控通道數、Module Name、FWID，顯示在進度視窗的 Board 區塊。
`firmwares` 的 `ic`、`xch`、`ych`、`moduleName`、`fwid` **全部符合**的韌體，才會被用在這塊板子上；欄位空白表示不檢查。
板子的機型取自符合條件的韌體的 `model` 欄位。

目前 TP1210UWDA-BCG V2 的 Module Name 是 `ILI2520000000000`、FWID 是 `0xffff`，兩者都是 ILITEK 的預設值。
同一顆 IC、同樣通道數的不同機型要分開時，需請韌體提供方在各機型的 hex 填入不同的 Module Name 或 FWID，再把它填進對應的 `moduleName` / `fwid`。

## 6. 欄位說明

```json
{
  "schema": 2,
  "updated": "2026-10-21 15:00:00",
  "settings": {
    "dailyCheckTime": "02:00",
    "configUrl": "https://example.com/tcb/manifest.json"
  },
  "target": "20260828_Enlarge",
  "modelTargets": [
    { "model": "TP1210UWDA-BCG_V2_DD", "target": "20260828_Enlarge" }
  ],
  "rules": [
    { "from": "unknown", "to": "none", "note": "unknown version, leave as is" }
  ],
  "firmwares": [
    {
      "version": "20260828_Enlarge",
      "model": "TP1210UWDA-BCG_V2_DD",
      "ic": "ILI2520", "xch": 40, "ych": 24,
      "moduleName": "", "fwid": "",
      "file": "TP1210UWDA-BCG_V2_DD_Enlarge_20260828.hex",
      "sha256": "2536102f...",
      "blocks": [ { "start": "0x3000", "end": "0x1CFA1", "crc": "0x32BC" } ],
      "notes": "Enlarge release, 2026-08-28"
    }
  ]
}
```

| 欄位 | 說明 |
|---|---|
| `updated` | 更新時間，只用來顯示 |
| `settings.dailyCheckTime` | 每日檢查時間 `HH:mm`，`off` = 關閉，不寫 = 程式預設 02:00 |
| `settings.configUrl` | 這份 config 應該在的網址；和目前位置不同時，程式搬過去 |
| `target` | 預設目標版本，`none` = 不更新 |
| `modelTargets` | 各機型的目標版本（V0.2 起；V0.1 會忽略） |
| `rules` | 依板上目前版本決定目標，`from` 可用 `unknown`、`*` |
| `firmwares[].version` | 版本名稱，唯一，給 target / rules 引用 |
| `firmwares[].model` | 機型名稱 |
| `firmwares[].ic` / `xch` / `ych` | IC 型號與觸控通道數，必須和板子相符才會使用；0 或空白 = 不檢查 |
| `firmwares[].moduleName` / `fwid` | 軟體名稱，用來區分同 IC 的不同機型；空白 = 不檢查 |
| `firmwares[].file` | hex 的網址，或相對於 config 的路徑 |
| `firmwares[].sha256` | 檔案的 SHA-256，下載後驗證（建議填） |
| `firmwares[].blocks` | 各 flash block 的位址與 CRC，用來辨識板上版本；不填時程式改用原廠工具逐版比對（較慢） |
| `notes` / `note` | 說明文字，會顯示在畫面上，請用英文 |

## 7. 改完怎麼確認

- 用瀏覽器打開 config 網址，確認讀得到最新內容。
- 用任何 JSON 檢查工具（例如 VS Code）確認格式正確。**格式錯誤時**，各電腦沿用最後一次讀到的設定，進度視窗顯示 `Cannot read the cloud config ... is not valid JSON`，不會燒錄任何東西。
- 在裝有程式的電腦執行 `DDTCBUpdateFW.exe --cloud`：印出目前雲端位置、每日檢查時間與來源、每一版韌體與規則。
- 系統匣圖示按右鍵：可看到每日檢查時間、雲端位置、最後一次讀取 config 的時間；「Reload cloud config」立即重讀。

## 8. demo 期間的切換工具（維護者電腦）

`03_工作檔/雲端上傳` 是這個倉庫的編輯來源，用下面的工具推送（需要 Python 與已存的 GitHub 憑證）：

| 工具 | 作用 |
|---|---|
| `切換到依版本規則.bat` | manifest.json = Enlarge→Final 單向（Final 不動） |
| `切換到Final.bat` / `切換到Enlarge.bat` | manifest.json = 固定目標 |
| `設定檢查時間.bat` | 輸入 `HH:mm` 或 `off`，只改每日檢查時間 |
| `產生清單.py` | hex 有變動時重新產生三份預備 config（保留目前的 settings） |

GitHub 未登入的 API 每小時限 60 次（每台電腦每小時約讀 1~2 次，demo 足夠）。
