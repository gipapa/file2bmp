# File <-> BMP 雙向無損轉換器 — 完整實作規格 v2.2

> 來源：規格照片 1-15，已依照片編號順序整理。
> 目標：建立零伺服器、零依賴、單檔 HTML 的純前端工具，將任意二進位檔案封裝成可由一般圖片檢視器開啟的 24-bit BMP，並可從 BMP 完整還原原始檔案。
>
> v2.1 修訂：修正 v2.0 將 payload bytes 誤當成 pixel count 的尺寸公式；補齊解碼邊界、輸入上限語意與可驗收測試。
>
> v2.2 修訂：將原始檔案硬性上限由 150 MiB 提高至 300 MiB，並同步調整解碼端 BMP 上限與記憶體需求說明。

## 0. 概覽

純前端工具：將任意二進位檔案以**視覺無損**方式封裝進 24-bit BMP 圖片中，再從 BMP 完整還原原始檔案。編碼與解碼各為獨立流程，共享同一套 Payload 格式與尺寸計算規則。

### 關鍵特性

- 零伺服器、零依賴、單檔 HTML
- SHA-256 Checksum（完整 32-byte digest 顯示、前 8 bytes 存檔）驗證往返完整性
- 4:3 比例最佳化 + Row Padding 最小化
- Bottom-Up scanline（BMP 標準正高度）
- 原始檔案 300 MiB 硬性上限；解碼端依可承載 300 MiB 原檔的最大 BMP 尺寸驗證

## 1. 二進位 Payload 格式

Payload 由 BMP 像素資料區承載。解碼時，將像素中的 BGR 位元組視為連續裸流。

| Offset | Size | 欄位 | 說明 |
|---:|---:|---|---|
| 0 | 4 B | Magic | ASCII `"F2BI"`，bytes 為 `0x46 0x32 0x42 0x49` |
| 4 | 4 B | Version | `Uint32 LE`，目前固定為 `1` |
| 8 | 8 B | Checksum | SHA-256 雜湊的**前 8 bytes** |
| 16 | 4 B | Reserved | `Uint32 LE`，固定為 `0` |
| 20 | 4 B | Original File Size | `Uint32 LE`，原始檔案位元組數 |
| 24 | 4 B | Filename Length | `Uint32 LE`，檔名字串的 UTF-8 位元組數 |
| 28 | N B | Filename | UTF-8 編碼，不含路徑分隔符 |
| 28 + N | M B | File Content | 原始檔案二進位主體 |

限制：

- `Original File Size <= 300 * 1024 * 1024`
- `Filename Length <= 1024` bytes，避免異常 metadata 造成過量配置
- Decoder 在讀取任何切片前，必須先驗證 `28 + N + M <= pixelPayloadCapacity`

總 payload 大小：

```text
payloadBytes = 28 + filenameUtf8ByteLength + originalFileByteLength
```

## 2. 尺寸計算

### 2.1 最佳化寬高（Encoder）

目標：以 4:3 比例估算最少像素數，再將寬度對齊至 4 的倍數。每個 24-bit 像素可承載 3 個 payload bytes。

```text
requiredPixels = ceil(payloadBytes / 3)
rawWidth       = ceil(sqrt(requiredPixels * 4 / 3))
rawHeight      = ceil(requiredPixels / rawWidth)
width     = rawWidth + ((4 - rawWidth % 4) % 4)
height    = ceil(requiredPixels / width)
```

`width * height * 3` 必須大於或等於 `payloadBytes`。寬度對齊到 4 的倍數後，24-bit BMP 每列不需要額外 row padding；最後一列未使用的通道補 `0x00`。

### 2.2 BMP Row Padding

BMP 每列必須 DWORD-aligned（4 bytes）：

```text
rowSize       = ceil(width * 3 / 4) * 4
rowPadding    = rowSize - (width * 3)
pixelDataSize = rowSize * height
```

因 Encoder 已將 `width` 對齊至 4 的倍數，24-bit BMP 的 `width * 3` 也會是 4 的倍數，因此本工具產生的檔案 `rowPadding = 0`。Decoder 仍保留通用公式，並要求本格式的 width 可被 4 整除。

### 2.3 完整 BMP 檔案大小

```text
bmpFileSize = 54 + pixelDataSize
// 14 bytes File Header + 40 bytes DIB Header + pixelDataSize
```

## 3. 編碼流程（Encoder: File -> BMP）

### Step 1 — 檔案驗證與讀取

1. 檢查 `file.size <= 300 * 1024 * 1024`，超限即拒絕。
2. Sanitize 後的 UTF-8 檔名不可超過 1024 bytes；超過時拒絕並提示縮短檔名。
3. 使用 `FileReader.readAsArrayBuffer(file)` 讀入 Buffer。
4. 使用 `FileReader.onprogress` 更新進度條，讀檔階段顯示百分比。

### Step 2 — Sanitize 檔名

Sanitize 僅在顯示與下載時使用；payload 中存的是 sanitize 後的檔名。

```javascript
function sanitize(name) {
  return name
    .split(/[\\/]/).pop()             // 取最後一段路徑
    .replace(/[?*:"><|]/g, "")        // 移除 OS 非法字元
    .replace(/^[.\s]+|[.\s]+$/g, "")  // 去首尾點與空白
    || "unknown";                     // 避免空字串
}
```

### Step 3 — 計算 SHA-256

```javascript
const shaBuffer = await crypto.subtle.digest("SHA-256", fileBuffer);
const shaBytes = new Uint8Array(shaBuffer); // 取前 8 bytes 存入 payload
```

### Step 4 — 建構 Payload（Uint8Array）

```text
payload = new Uint8Array(payloadBytes)
idx = 0
```

Magic 與 header：

```text
payload[0..3] = [0x46, 0x32, 0x42, 0x49] // "F2BI"
dataView.setUint32(4, 1, true)           // version = 1 (LE)
payload.set(shaBytes.slice(0, 8), 8)     // SHA-256 前 8 bytes
payload[16..19] = 0x00 0x00 0x00 0x00   // Reserved
```

Metadata：

```text
dataView.setUint32(20, originalFileSize, true)
dataView.setUint32(24, filenameUtf8ByteLength, true)
```

Filename 與 File Content：

```text
payload.set(utf8EncodedFilename, 28)
payload.set(originalFileBytes, 28 + filenameUtf8ByteLength)
```

### Step 5 — 寫入 BMP 像素資料（Bottom-Up）

BMP 標準：當 `height > 0` 時，掃描線由底至頂排列（row 0 = 最底行）。

```text
bmp = new Uint8Array(54 + pixelDataSize) // header 與像素區初始化為 0
idx = 0

for row from 0 to height - 1:
  dstOffset = 54 + (height - 1 - row) * rowSize

  for col from 0 to width - 1:
    if idx < payloadBytes:
      bmp[dstOffset + col * 3]     = payload[idx++] // B
      bmp[dstOffset + col * 3 + 1] = payload[idx++] // G
      bmp[dstOffset + col * 3 + 2] = payload[idx++] // R

// idx >= payloadBytes 後，bmp 像素區保持 0x00，作為尾端 padding
```

關鍵：payload 的位元組直接映射到 BGR 通道，不需要 BGR <-> RGB 轉換，因為 payload 是裸位元組流。

### Step 6 — 寫入 BMP Header

```text
bmp 已在 Step 5 建立；以 `new DataView(bmp.buffer)` 寫入 header。
```

File Header（14 bytes）：

```text
bmp[0..1] = [0x42, 0x4D]                  // "BM"
dataView.setUint32(2, bmpFileSize, true)  // 總檔案大小
dataView.setUint32(10, 54, true)          // 像素資料偏移
```

DIB / BITMAPINFOHEADER（40 bytes）：

```text
dataView.setUint32(14, 40, true)              // DIB Header 大小
dataView.setInt32(18, width, true)            // 寬度，有號整數
dataView.setInt32(22, height, true)           // 高度，正數 = bottom-up
dataView.setUint16(26, 1, true)               // 平面數 = 1
dataView.setUint16(28, 24, true)              // 位元深度 = 24
dataView.setUint32(30, 0, true)               // 壓縮 = BI_RGB (0)
dataView.setUint32(34, pixelDataSize, true)   // 圖片大小
```

`bytes 38-51`（水平/垂直解析度、色彩數、重要色彩數）維持 `0` 預設值。

### Step 7 — 匯出與清理

```javascript
const blob = new Blob([bmp], { type: "image/bmp" });
const url = URL.createObjectURL(blob);
// 觸發下載：a.href = url; a.download = sanitizedName + ".bmp"
// 完成後：
URL.revokeObjectURL(url);
```

## 4. 解碼流程（Decoder: BMP -> File）

### Step 1 — 檔案驗證

BMP 本身可能比內含原檔多出 payload/header/padding，因此不能直接套用原檔 300 MiB 上限。Decoder 先用「300 MiB 原檔 + 1024-byte 檔名」計算可接受的最大 BMP 大小；解析 payload 後再確認 `Original File Size <= 300 MiB`。

### Step 2 — 讀取 BMP

```javascript
const reader = new FileReader();
reader.readAsArrayBuffer(file);
```

### Step 3 — BMP 格式驗證（全部必填）

| 檢查 | Offset | 預期值 | 失敗時 |
|---|---:|---|---|
| File Header | `[0..1]` | `0x42 0x4D` (`"BM"`) | 拋出錯誤 |
| 實際檔案長度 | `bytes.byteLength` | `>= 54` | 拋出錯誤 |
| Header File Size | `dataView.getUint32(2, true)` | 等於實際檔案長度 | 拋出錯誤 |
| Pixel Offset | `dataView.getUint32(10, true)` | `54` | 拋出錯誤 |
| DIB Header Size | `dataView.getUint32(14, true)` | `40` (`BITMAPINFOHEADER`) | 拋出錯誤 |
| Width | `dataView.getInt32(18, true)` | `> 0` 且 `% 4 === 0` | 拋出錯誤 |
| Height | `dataView.getInt32(22, true)` | `> 0` | 拋出錯誤 |
| Planes | `dataView.getUint16(26, true)` | `1` | 拋出錯誤 |
| Bits Per Pixel | `dataView.getUint16(28, true)` | `24` | 拋出錯誤 |
| Compression | `dataView.getUint32(30, true)` | `0` (`BI_RGB`) | 拋出錯誤 |
| Pixel Bounds | `54 + rowSize * height` | 等於實際檔案長度 | 拋出錯誤 |

### Step 4 — 萃取像素資料（Bottom-Up -> 連續 Byte Array）

```text
rowSize   = ceil(width * 3 / 4) * 4
extracted = new Uint8Array(width * height * 3)

for row from 0 to height - 1:
  srcOffset = (height - 1 - row) * rowSize   // BMP bottom-up
  dstOffset = row * width * 3                // 連續排列

  for col from 0 to width - 1:
    extracted[dstOffset + col * 3]     = bytes[54 + srcOffset + col * 3]
    extracted[dstOffset + col * 3 + 1] = bytes[54 + srcOffset + col * 3 + 1]
    extracted[dstOffset + col * 3 + 2] = bytes[54 + srcOffset + col * 3 + 2]
```

不需要 BGR <-> RGB 轉換，因為 payload 是裸位元組流，解碼時僅按順序讀取。

### Step 5 — 解析與驗證 Payload

```text
// 1. 驗證 Magic
if extracted[0..3] != "F2BI" -> 拋出錯誤

payloadView = new DataView(extracted.buffer)

// 2. 驗證 Version
if payloadView.getUint32(4, true) != 1 -> 拋出錯誤（目前僅支援 v1）
if payloadView.getUint32(16, true) != 0 -> 拋出錯誤（Reserved 必須為 0）

// 3. 讀取 Metadata
originalFileSize = payloadView.getUint32(20, true)
filenameUtf8Len  = payloadView.getUint32(24, true)

if originalFileSize > 300 MiB -> 拋出錯誤
if filenameUtf8Len > 1024 -> 拋出錯誤
if 28 + filenameUtf8Len + originalFileSize > extracted.byteLength -> 拋出錯誤

// 4. 讀取檔名並 Sanitize
rawFilename = new TextDecoder().decode(extracted[28 .. 28 + filenameUtf8Len - 1])
filename = sanitize(rawFilename)

// 5. 讀取檔案內容
fileBuffer = extracted[28 + filenameUtf8Len .. 28 + filenameUtf8Len + originalFileSize - 1]

// 6. 驗證 Checksum
storedChecksum = extracted[8..15] // 8 bytes
computedSHA = await crypto.subtle.digest("SHA-256", fileBuffer)
match = storedChecksum == computedSHA[0..7]
```

Checksum 結果：

- match：顯示 `✓`（綠色，完整性已驗證）
- 不 match：顯示 `⚠`（黃色，檔案可能已損毀），但仍提供下載

### Step 6 — 還原下載

```javascript
const blob = new Blob([fileBuffer], { type: "application/octet-stream" });
const url = URL.createObjectURL(blob);
// a.href = url; a.download = filename
// 完成後：
URL.revokeObjectURL(url);
```

## 5. 錯誤處理

| 情境 | 行為 |
|---|---|
| 原始檔案超過 300 MiB | 阻斷，顯示友善錯誤訊息 |
| BMP 超過可承載 300 MiB 原檔的理論最大值 | 阻斷，顯示友善錯誤訊息 |
| Header 宣告長度、像素長度或 payload metadata 超出實際檔案 | 阻斷，顯示檔案遭截斷或格式不符 |
| BMP 非有效格式（BM magic / offset / DIB / bpp / compression） | 拋出錯誤，顯示解析失敗 |
| Magic != `"F2BI"` | 拋出錯誤，提示非本工具產生 |
| Version != 1 | 拋出錯誤，目前無向後相容 |
| Checksum 比對失敗 | 顯示警告與「檔案可能已損毀或遭修改」，但仍提供下載 |
| Checksum 比對成功 | 顯示 `✓` 與「完整性已驗證，可安全下載」 |
| 記憶體不足 | 捕獲異常並提示 |

## 6. 記憶體模型

### 編碼時

```text
峰值記憶體 ≈ fileBuffer + payload + bmpBuffer
           ≈ 1 * fileSize + 1 * payloadBytes + 1 * bmpFileSize
```

### 解碼時

```text
峰值記憶體 ≈ bmpBuffer + extracted + fileBuffer + blob
           ≈ 1 * bmpFileSize + 1 * (width * height * 3) + 1 * fileSize
```

### 總覽

峰值約為**原始檔案的 3 倍**（Blob 是否複製 backing store 依瀏覽器實作而定）。對 300 MiB 檔案，實務上應預留至少約 900 MiB 可用記憶體。

## 7. UI 規格

| 元件 | 說明 |
|---|---|
| 分頁切換 | 兩個 tab：`File to BMP` / `BMP to File` |
| 拖放區域 | 支援點擊選檔 + 全域拖放，hover/over 時視覺變化 |
| 檔案資訊 | 顯示檔名、大小、SHA-256（編碼前）、預估 BMP 尺寸 |
| 進度條 | 讀檔時顯示進度百分比，編碼/解碼時顯示階段 |
| 結果區 | 成功（綠邊）/ 警告（黃邊）/ 錯誤（紅邊） |
| 複製按鈕 | 一鍵複製檔名或 Checksum |
| 下載按鈕 | 一鍵下載編碼結果 BMP 或還原檔案 |
| 響應式 | 桌面與行動裝置均支援 |

## 8. 實作結構

- 交付檔案：`index.html` 與本 `plan.md`。
- `index.html` 內含 HTML、CSS、JavaScript，不載入 CDN、字型、分析服務或任何網路資源。
- 核心純函式：`sanitizeFilename`、`calculateDimensions`、`encodeFileToBmp`、`decodeBmpToFile`、`sha256`。
- UI 僅負責檔案讀取、進度/狀態顯示、複製與下載；二進位格式邏輯集中於核心函式。
- 頁面可直接以 `file://` 開啟；SHA-256 使用瀏覽器原生 Web Crypto API。

## 9. 驗收與真實測試

完成條件必須同時滿足：

1. 文字檔、含 UTF-8 檔名的二進位檔、空檔案皆能完成 `File -> BMP -> File` 往返。
2. 還原檔案的 byte-for-byte checksum 與原始檔案相同。
3. 產生的檔案具有合法 24-bit BMP header，且可由真實瀏覽器載入為圖片。
4. 修改像素 payload 後，Decoder 顯示 checksum 警告但仍允許下載。
5. 非 BMP、非本工具產生的 BMP、截斷 BMP、錯誤 version、超限檔案均顯示明確錯誤且不崩潰。
6. 桌面與行動寬度下，頁籤、拖放區、資訊區、進度與下載控制不重疊且文字不溢出。
7. 真實瀏覽器中實際選檔、下載 BMP、切換解碼、選取該 BMP 並下載還原檔；最後以系統工具比對原檔與還原檔完全一致。

## 10. 2026-07-16 實測紀錄

- 真實 UI 往返：257 KiB 隨機二進位檔（UTF-8 檔名）成功完成 `File -> BMP -> File`。
- 原檔與還原檔通過 `cmp` byte-for-byte 比對，SHA-256 均為 `be15b7535bd65e3160ecfac9b7d7e6ffc8ab3e21b60793ea8ed53bbf0f7527e4`。
- macOS `file` 與 `sips` 均辨識輸出為合法 `344 x 256 x 24-bit BMP`；真實瀏覽器的圖片解碼器亦成功載入。
- 8 MiB + 317 bytes 樣本往返成功；輸出 BMP 為 8,392,662 bytes，確認沒有 v2.0 公式造成的約 3 倍膨脹。
- 核心錯誤測試通過：無效 BMP、合法但無 F2BI magic、截斷 BMP、不支援 version、checksum mismatch、超限輸入、過長 filename metadata、reserved 欄位錯誤。
- 空檔案與 UTF-8 檔名往返通過。
- 1440 px、390 px 與 320 px viewport 已檢查；320 px 下 `scrollWidth === clientWidth`，無水平溢出，瀏覽器 console/page errors 為空。

## 11. 2026-07-17 300 MiB 上限驗證

- `MAX_ORIGINAL_BYTES` 已確認為 `314,572,800` bytes（300 MiB）。
- 以 1024-byte 最大檔名計算時，payload 為 `314,573,852` bytes，輸出尺寸為 `11828 x 8866`，理論最大 BMP 為 `314,601,198` bytes，像素容量足以承載完整 payload。
- Encoder 對 `300 MiB + 1 byte` 輸入會在讀檔前拒絕，Decoder 亦會拒絕超過理論最大 BMP 大小的輸入。
- 1 MiB + 17 bytes 樣本（UTF-8 檔名）在真實瀏覽器完成往返，還原 bytes 完全一致且 checksum 通過。
- UI 的頂部限制與拖放提示均顯示 300 MiB；瀏覽器 console/page errors 為空。
- 完整 300 MiB 往返未列入自動回歸測試，因現行全記憶體模型預期需要約 900 MiB 可用記憶體。
