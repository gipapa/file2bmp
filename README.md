# file2bmp

將任意二進位檔案無損封裝成標準 24-bit BMP，並從 BMP 還原原始檔案。

`file2bmp` 是零伺服器、零依賴、單檔 HTML 的純前端工具。所有讀取、編碼、驗證與下載都在瀏覽器本機完成，不會上傳檔案，也不載入 CDN、分析服務或其他外部資源。

## 功能

- 任意檔案轉為可由一般圖片檢視器開啟的 24-bit BMP
- 從本工具產生的 BMP 完整還原原始檔案與檔名
- SHA-256 完整雜湊顯示，並在 payload 儲存前 8 bytes 供往返驗證
- 4:3 尺寸估算、4-byte 對齊與標準 Bottom-Up scanline
- 拖放與點擊選檔、進度顯示、錯誤提示及一鍵下載
- 桌面與行動裝置響應式介面
- 原始檔案上限 300 MiB

## 直接使用

use github pages: https://gipapa.github.io/file2bmp/

or deploy to your PC:
1. 下載 `index.html`。
2. 直接用瀏覽器開啟

### File to BMP

選取原始檔案後，工具會清理檔名、計算 SHA-256、建立 F2BI payload，並將 payload 依 BMP Bottom-Up 順序寫入像素資料區。輸出檔名為 `<原檔名>.bmp`。

### BMP to File

選取由本工具產生的 BMP 後，工具會驗證 BMP header、F2BI magic、版本、邊界與 checksum，再提供原始檔案下載。若 checksum 不符，會明確警告，但仍允許取回可讀出的內容。

## 隱私與安全

- 檔案只在目前瀏覽器分頁的記憶體中處理，程式本身不發出網路請求。
- BMP 封裝**不是加密，也不是存取控制**。任何知道格式的人都能還原內容，請勿將它視為保護機密資料的方法。
- 儲存在 BMP 中的是 SHA-256 的前 8 bytes，適合偵測傳輸或儲存損毀，但不是用來抵抗蓄意偽造的數位簽章。
- 下載檔名會移除路徑與常見作業系統非法字元，避免路徑型檔名被直接沿用。
- 300 MiB 是原始檔案硬性上限；大型檔案處理時，瀏覽器可能需要約原檔 3 倍、約 900 MiB 的可用記憶體。

## F2BI payload 格式

Payload 直接承載於 BMP 像素資料區；B、G、R 通道位元組依序視為連續 byte stream，不做色彩轉換。

| Offset | Size | 欄位 | 格式 |
| ---: | ---: | --- | --- |
| 0 | 4 B | Magic | ASCII `F2BI` |
| 4 | 4 B | Version | Uint32 LE，目前為 `1` |
| 8 | 8 B | Checksum | SHA-256 前 8 bytes |
| 16 | 4 B | Reserved | Uint32 LE，固定為 `0` |
| 20 | 4 B | Original File Size | Uint32 LE |
| 24 | 4 B | Filename Length | UTF-8 byte length，最大 1024 |
| 28 | N B | Filename | Sanitized UTF-8 filename |
| 28 + N | M B | File Content | 原始檔案 bytes |

完整二進位格式、尺寸公式、驗證條件與錯誤處理請見 [plan.md](plan.md)。

## 相容性

需要支援下列瀏覽器 API：

- `FileReader`
- `Blob` 與 `URL.createObjectURL`
- `TextEncoder` / `TextDecoder`
- Web Crypto API 的 `crypto.subtle.digest`

現代版 Chrome、Edge、Firefox 與 Safari 均提供這些 API。頁面可由 `file://` 直接開啟；若瀏覽器政策限制本機頁面的 Web Crypto，請改用任意靜態 HTTP server 開啟。

## 驗證紀錄

目前實作已完成下列真實測試：

- 257 KiB 隨機二進位檔（UTF-8 檔名）由 UI 完成 `File -> BMP -> File`
- 原檔與還原檔通過 byte-for-byte `cmp`，SHA-256 完全一致
- 8 MiB + 317 bytes 樣本往返成功
- 300 MiB 上限的容量與 BMP 尺寸計算通過；300 MiB + 1 byte 會在讀檔前拒絕
- macOS `file`、`sips` 與瀏覽器圖片解碼器均辨識輸出為合法 24-bit BMP
- 空檔案、截斷 BMP、錯誤版本、錯誤 reserved 欄位、非 F2BI BMP 與 checksum mismatch 路徑均已測試
- 1440 px、390 px 與 320 px viewport 無控制項重疊或水平溢出

詳細測試數值與驗收條件記錄於 [原始實測紀錄](plan.md#10-2026-07-16-實測紀錄)與 [300 MiB 上限驗證](plan.md#11-2026-07-17-300-mib-上限驗證)。

## 專案結構

```text
file2bmp/
├── index.html   # 完整 UI、樣式與編解碼實作
├── plan.md      # 格式規格、流程、邊界條件與測試紀錄
└── README.md    # 專案說明
```

核心函式也以唯讀物件公開於 `window.file2bmp`，方便從瀏覽器開發者工具進行測試：

```javascript
window.file2bmp.calculateDimensions(payloadBytes);
const encoded = await window.file2bmp.encodeFileToBmp(sourceFile);
const decoded = await window.file2bmp.decodeBmpToFile(bmpFile);
```

此專案不需要 build step；修改 `index.html` 後重新整理頁面即可。
