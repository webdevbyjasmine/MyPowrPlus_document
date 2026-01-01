# mypowrplus ADAPTER、TCP、WiFi、Wireshark 技術文檔

---

## 目錄
1. [WiFi 轉板基本信息](#1-wifi-轉板基本信息)
2. [ADAPTER API 路徑與配置](#2-adapter-api-路徑與配置)
3. [ADAPTER 傳輸流程](#3-adapter-傳輸流程)
4. [WiFi 連線狀態與配置](#4-wifi-連線狀態與配置)
5. [MAC 位址定義](#5-mac-位址定義)
6. [WiFi 連線配置方案](#6-wifi-連線配置方案)
7. [Wireshark 抓包分析](#7-wireshark-抓包分析)
8. [ADAPTER 傳輸頻率與超時設定](#8-adapter-傳輸頻率與超時設定)

---

## 1. WiFi 轉板基本信息

### 1.1 轉板標識符

| 欄位 | 值 | 說明 |
|------|-----|------|
| MAC 位址 | `88:DA:1A:54:99:2C` | WIFI 轉板實體位址（無冒號格式） |
| 轉板版本 | `v0.0.1` | WIFI 轉板韌體版本 |
| 轉板 UUID | `***` | 轉板唯一識別號 |

### 1.2 轉板相關欄位

```json
{
  "macAddress": "88:DA:1A:54:99:2C",
  "adapterVersion": "v0.0.1",
  "adapterUuid": "***",
  "groupName": "ADAPTER_GROUP_NAME",
  "processStreamName": "ADAPTER1-PROCESS-STREAM",
  "streamName": "ADAPTER1-WSS-STREAM",
  "connectionDate": "2025-03-19T06:04:44.281Z"
}
```

### 1.3 轉板連線時戳欄位

- **connectionDate**: 第一次轉板 Adapter 連接上的時間，也就是此筆創建的時間

---

## 2. ADAPTER API 路徑與配置

### 2.1 WebSocket 連線位址

#### 生產環境
```
ws://powr.plus/adpt
```

#### 開發環境
```
ws://powr.plus/dev
```

#### 本地測試
```
ws://57.180.206.69:5100
```

### 2.2 API 流向概念

**基本流向**:
```
phone → SG (Server Gateway) → adapter
```

**完整流向範例 (SetTime)**:
```
phone → SG-user Server → adapter
```

---

## 3. ADAPTER 傳輸流程

### 3.1 E070 UserProfile 傳遞流程（已實作）

**流向**: 手機用戶資料傳遞到機台

```
E070PS (user) 
    ↓
E070SC (adapter) 
    ↓
E070CS (adapter) 
    ↓
E070SP (user)
```

### 3.2 E024 用戶啟動流程（已實作）

**說明**: 手機掃 QRCode 後傳給伺服器

**動作**:
1. 把 userID 寫入 WORKOUT-HASH 內
2. 建立 user 和 adapter 的連接橋接

**輸入格式**:
```json
{ 
  "E024_UserStart_PS": {
    "macAddress": "88:DA:1A:6B:7E:08"
  }
}
```

**輸出**:
```json
{ 
  "E024_UserStart_SP": {
    "status": "success"
  }
}
```

### 3.3 E410 設備類型與單位傳遞

```json
{ 
  "E410_DeviceTypeAndUnit_CS": {
    "devType": "0C-GreenSystemBike",
    "unitType": "0-Metric",
    "timeUnit": "0-Second",
    "speedUnit": "0-Metric",
    "speedScale": "1-x1/10Km/h",
    "distanceUnit": "1-x1/10",
    "distanceScale": "0-x1/100",
    "caloriesScale": "2-1Kcal",
    "humanWattsEnergyUnit": "0-Watt",
    "heightScale": "2-x1cm",
    "weightScale": "2-x1Kg"
  }
}
```

---

## 4. WiFi 連線狀態與配置

### 4.1 E603_ClientStatus_SC API（需實作）

**用途**: 由公司取得 MAC → 去找出 WiFi 狀態

**流向**:
```
company → SG → Adapter → SG → company
```

**輸入格式**:
```json
{
  "E603_ClientStatus_SC": {
    "status": "?"
  }
}
```

### 4.2 E601 數據傳輸間隔設置（需實作）

**用途**: 設定 Adapter 向服務器上傳數據的時間間隔

```json
{
  "E601_IntervalTimeSetting_SC": {
    "workoutDataIntervalTimeSecond": 1,
    "workoutDataIntervalTimeMillisecond": 0
  }
}
```

---

## 5. MAC 位址定義

### 5.1 MAC 位址格式規定

| 來源 | 格式 | 說明 |
|------|------|------|
| QRCode | **無冒號** | `88DA1A54992C` | 確定格式，不用再轉碼 |
| 轉板傳輸 | **無冒號** | `88DA1A54992C` | 確定格式，需改轉板程式確保統一 |
| 資料庫記錄 | **無冒號** | `88DA1A54992C` | 排序方便 |
| 工程人員用 | *有冒號* | `88:DA:1A:54:99:2C` | 易讀性，不涉及程式邏輯 |

### 5.2 重要提示

> 🔴 **請改轉板程式** - 轉板來的 MAC 位址應傳送無冒號格式，可減少 5 個 byte 的傳輸

---

## 6. WiFi 連線配置方案

### 6.1 無顯示器設備的參數配置方法

在設備上**沒有顯示器**並且需要將配置參數（如 companyID）傳輸到 WiFi 板上時，除了使用 USB 外，可以考慮以下方法：

#### 6.1.1 網絡配置（Network Configuration）

**配置服務器法**
- 設置一個配置服務器，當板子連接到網絡時，自動從服務器下載配置參數
- 通過配置文件或 API 提供參數

**DHCP 配置法**
- 通過 DHCP 服務器傳遞配置信息
- 在 DHCP 服務器中設置適當的選項來分發參數

#### 6.1.2 藍牙配置（Bluetooth）

- 如果板子支持藍牙，使用藍牙設備將配置文件或參數直接傳輸到板子上

#### 6.1.3 無線網絡傳輸

**Wi-Fi Direct**
- 如果板子支持 Wi-Fi Direct，通過 Wi-Fi Direct 連接並傳輸文件

**局域網共享**
- 在與板子相同的網絡上，使用局域網共享文件
- 通過網絡共享的文件夾或 FTP 服務實現

#### 6.1.4 串行通信（Serial Communication）

- 使用串行端口（UART 或 RS-232）將配置數據傳輸到板子上
- 如果板子有串行接口，可以直接發送數據

#### 6.1.5 HTTP/FTP 下載

**HTTP 下載**
- 如果板子能夠連接到互聯網，運行小型 HTTP 客戶端
- 從預先設置的 HTTP 伺服器下載配置文件

**FTP 下載**
- 類似地，通過 FTP 客戶端從 FTP 伺服器下載配置文件

#### 6.1.6 智能卡或 NFC

- 如果板子支持 NFC 或智能卡技術，可使用這些設備傳輸配置數據

### 6.2 companyName 存儲位置

> ℹ️ **重要** - companyName 的內容是被寫在 WIFI 轉板內，格式為 JSON

**companyName 定義**:
- 由力伽授權，第三方須先提供申請後才能加入白名單
- 只有授權的 companyName 才能使用
- 例：`SportsArt` 或 `奇美醫院中華店`

---

## 7. Wireshark 抓包分析

### 7.1 Wireshark 用途

Wireshark 是網絡封包分析工具，用於：
- 監控 WiFi 轉板與服務器之間的 TCP/IP 通信
- 分析數據包結構與傳輸流量
- 調試連線問題與性能優化

### 7.2 Wireshark 分析重點

#### 7.2.1 監控內容
- **WebSocket 連線**: 監控 `ws://` 或 `wss://` 連線建立與維護
- **數據傳輸**: 分析 JSON 數據包的傳輸情況
- **心跳信號**: 確認 adapter 定期傳送狀態信號
- **故障排查**: 識別連線中斷或延遲的原因

#### 7.2.2 常見 Wireshark 過濾條件

```
# 監控 WebSocket 流量
tcp.port == 5100

# 監控特定 IP 的轉板
ip.src == 57.180.206.69

# 監控 HTTPS/WSS 加密連線
tcp.port == 443 || tcp.port == 8443

# 篩選 JSON 數據
tcp.payload contains "E024_UserStart" || tcp.payload contains "macAddress"
```

#### 7.2.3 Wireshark 操作步驟

1. **啟動 Wireshark** 並選擇正確的網卡
2. **設置過濾條件** 聚焦於 adapter 相關流量
3. **開始抓包** - 點選 `Capture Start`
4. **觸發事件** - 執行測試場景（例：手機掃 QRCode）
5. **停止抓包** - 點選 `Capture Stop`
6. **分析數據**:
   - 檢查 TCP 三向交握是否成功
   - 驗證 WebSocket 握手信息
   - 檢查 JSON 數據包內容
   - 確認每 10 秒傳送一次的心跳信號

#### 7.2.4 Wireshark 圖表分析

- **image 13.png** - 轉板連線初始化過程
- **image 14.png** - 持續數據傳輸過程

### 7.3 TCP 三向交握（Three-Way Handshake）

WebSocket 連線前的 TCP 建立過程：

```
Adapter                          Server
    |                                |
    |-------- SYN (seq=x) --------→ |
    |                                |
    | ← SYN-ACK (seq=y, ack=x+1) ←  |
    |                                |
    |------- ACK (seq=x+1) -------→ |
    |                                |
    ✓ TCP 連線已建立
    |
    |---- WebSocket Upgrade ----→  |
    | (Upgrade: websocket)          |
    |                                |
    | ← 101 Switching Protocols ←   |
    |                                |
    ✓ WebSocket 連線已建立
```

### 7.4 WebSocket 握手示例

**客戶端請求**:
```http
GET /adpt HTTP/1.1
Host: powr.plus
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: ***
Sec-WebSocket-Version: 13
```

**服務器響應**:
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: ***
```

---

## 8. ADAPTER 傳輸頻率與超時設定

### 8.1 傳輸頻率規定

| 項目 | 設定值 | 說明 |
|------|--------|------|
| Adapter 傳送間隔 | **10 秒** | adapter 每 10 秒傳送一次數據/心跳 |
| 程式偵測超時 | **30 秒** | 若 30 秒內未收到 adapter 信號，判定連線中斷 |
| 可調間隔 | 可自訂 | 通過 E601 API 調整 `workoutDataIntervalTimeSecond` |

### 8.2 實時資料流

```
Adapter 發送時間線:

時間 0s:   ✓ 首次連接建立
時間 10s:  ✓ 第 1 次數據/心跳傳送
時間 20s:  ✓ 第 2 次數據/心跳傳送
時間 30s:  ✓ 第 3 次數據/心跳傳送 (若超過此時間無信號 → 連線中斷)
時間 40s:  ✓ 第 4 次數據/心跳傳送
...
```

### 8.3 監控檢查點

1. **連線建立**: 確認 adapter 成功連接到服務器
2. **心跳信號**: 每 10 秒檢查一次信號接收
3. **數據完整性**: 驗證接收的 JSON 數據格式正確
4. **超時判定**: 超過 30 秒未收到信號時觸發重新連線
5. **流量分析**: 使用 Wireshark 監控實時流量

---

## 附錄：相關 API 端點

### A.1 E306_SetTime - 設置時間

```json
{
  "E306_SetTime": {
    "dateTime": "2024-02-23 13:45:30"
  }
}
```

### A.2 E53A_ErrorCode_CS - 錯誤碼

```json
{
  "E53A_ErrorCode_CS": {
    "errorCode": 1234
  }
}
```

### A.3 E522_CompanyProfileRequest_CS - 公司資料請求

```json
{
  "E522_CompanyProfileRequest_CS": {
    "request": "unknow"
  }
}
```

---




