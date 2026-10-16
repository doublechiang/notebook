# PLDM & MCTP 伺服器硬體管理架構筆記 (Server Hardware Management Notes)

> 本文件彙整 BMC、BIOS 與 PCIe 擴充卡之間的 MCTP/PLDM 通訊架構、硬體訊號、標準規範與實務設計細節。
> 建議在 VS Code 中搭配 **Markdown Preview (`Ctrl + Shift + V`)** 或左側 **Outline（大綱目錄）** 快速導航閱讀。

---

## 目錄 (Table of Contents)
1. [PLDM 核心規範概覽 (DSP0245 / 0246 / 0248 / 0257)](#1-pldm-核心規範概覽-dsp0245--0246--0248--0257)
2. [DSP0246：SMBIOS 資料傳輸機制](#2-dsp0246smbios-資料傳輸機制)
3. [DSP0248 與 DSP0257：Sensor 與 FRU 的架構劃分](#3-dsp0248-與-dsp0257sensor-與-fru-的架構劃分)
4. [卡片開機與 BMC 偵測機制 (Power & Presence)](#4-卡片開機與-bmc-偵測機制-power--presence)
5. [深度剖析：PCIe VDM、Discovery Notify 與 Bus Owner](#5-深度剖析pcie-vdmdiscovery-notify-與-bus-owner)
6. [工程實務真相與常見陷阱 (Brutal Truths & Pitfalls)](#6-工程實務真相與常見陷阱-brutal-truths--pitfalls)

---

## 1. PLDM 核心規範概覽 (DSP0245 / 0246 / 0248 / 0257)

DMTF 在 **DSP0245** 定義了 PLDM Type Codes，將不同功能模組標準化：

| PLDM Type | 規範編號 | 全名 / 功能 | 典型應用場景 |
| :---: | :---: | :--- | :--- |
| `0x00` | **DSP0240** | **Base Specification** | 通訊控制、協定握手、版本協商 (`GetPLDMTypes`, `GetPLDMVersion`) |
| `0x01` | **DSP0246** | **SMBIOS Data Transfer** | 主機 BIOS 在開機過程將 SMBIOS Table 傳送至 BMC |
| `0x02` | **DSP0248** | **Platform Monitoring and Control** | 數值型/狀態型感測器 (Sensors)、效應器 (Effecters)、PDR 平台描述記錄 |
| `0x03` | **DSP0247** | **BIOS Control and Configuration** | BMC 與 BIOS 之間傳遞 BIOS 設定值與屬性表 |
| `0x04` | **DSP0257** | **FRU Data** | 存取 Field Replaceable Unit 資產資料 (序號、料號、製造商) |
| `0x05` | **DSP0267** | **Firmware Update** | 透過 BMC 對各端點裝置 (PCIe 卡、MCU) 執行標準化韌體更新 |
| `0x06` | **DSP0218** | **Redfish Device Enablement (RDE)** | 讓端點裝置直接提供 Redfish Schema 資源 (降低 BMC 負擔) |

---

## 2. DSP0246：SMBIOS 資料傳輸機制

### 2.1 解決的痛點
* **傳統做法**：依賴 OS In-band Agent (如 `dmidecode` 搭配 `ipmitool`) 或私有 Mailbox/IPMI OEM 命令。OS 未開機或當機時，BMC 無法掌握 CPU、DIMM 等硬體配置。
* **標準解法**：BIOS 在 POST 階段透過標準 PLDM over MCTP (經由 eSPI 或 SMBus) 將整份 SMBIOS 傳送給 BMC。

### 2.2 運作流程
```
[ Host CPU / BIOS ]
       │  1. POST 階段硬體探索（CPU、DIMM、PCIe 裝置）
       │  2. 組成 SMBIOS 結構表 (Type 0 BIOS, Type 1 System, Type 4 CPU, Type 17 Memory...)
       ▼
 [ PLDM over MCTP ] (DSP0246, PLDM Type 0x01)
       │  3. 傳送 GetSMBIOSStructureTableMetadata (取得長度、數量、CRC)
       │  4. 分段傳輸 (SetSMBIOSStructureTable / GetSMBIOSStructureTable)
       ▼
[ BMC (OpenBMC) ]
       │  5. pldmd 接收後由 smbios-mdr 模組解析
       ▼
[ Redfish API / WebUI ]
       │  6. 轉換為 Redfish 標準資料 (/redfish/v1/Systems/system/Processors...)
```

---

## 3. DSP0248 與 DSP0257：Sensor 與 FRU 的架構劃分

### 3.1 核心疑問：一般的 Sensor 與 FRU 支援 PLDM 嗎？
**不支援。**
* 傳統單體 IC（如 TMP75 溫度感測器、24C02 EEPROM）屬於**「純硬體 Slave IC（Dumb Devices）」**，沒有微控制器（MCU）與韌體，無法處理 MCTP 網路層與 PLDM 應用層。
* **BMC 對待它們的方式**：透過 Linux Kernel Driver（如 `hwmon`、`at24`）以傳統 I2C/SMBus 暫存器讀寫直接控制。

### 3.2 誰在跑 DSP0248 / DSP0257？
只有具備**「運算與韌體處理能力」的智慧子系統**才會實作 PLDM：
1. **PCIe 擴充卡 (SmartNIC, GPU, DPU, CXL, RAID 卡)**：卡上的管理 MCU 負責讀取板載 TMP75 與 EEPROM，並對外扮演 **PLDM Endpoint** 代表該卡向 BMC 報告。
2. **衛星管理控制器 (Satellite Management Controller, SMC)**：如硬碟背板、風扇板或 PDB 上的專屬 MCU。

### 3.3 架構對比
```
                    ┌─────────────────────────┐
                    │      BMC (OpenBMC)      │
                    └───────────┬─────────────┘
                                │
        ┌───────────────────────┴───────────────────────┐
        │ [傳統 Direct I2C/SMBus]                        │ [PLDM over MCTP: PCIe VDM / SMBus]
        ▼                                               ▼
┌───────────────────────┐                   ┌───────────────────────┐
│ 主機板本地元件 (Dumb) │                   │ 智慧擴充模組 / PCIe 卡 │
│                       │                   │                       │
│ ┌───────────────────┐ │                   │   ┌───────────────┐   │
│ │ TMP75 (溫度 IC)   │ │ (BMC Linux hwmon  │   │ 卡載 MCU/ASIC │◀──┼── PLDM Endpoint
│ └───────────────────┘ │  driver 直接讀寫) │   │ (跑 PLDM 堆疊)│   │   (DSP0248/0257)
│ ┌───────────────────┐ │                   │   └───────┬───────┘   │
│ │ 24C02 (FRU EEPROM)│ │                   │           │ (卡內私有 I2C)
│ └───────────────────┘ │                   │   ┌───────┴───────┐   │
│                       │                   │   │ 板載 Sensor / │   │
│                       │                   │   │ 板載 FRU IC   │   │
└───────────────────────┘                   │   └───────────────┘   │
                                            └───────────────────────┘
```

### 3.4 為什麼擴充模組需要 PDR (Platform Descriptor Record)？
* **自描述能力 (Self-Describing)**：BMC 不需預先在系統設定檔寫死該卡有哪顆 Sensor、門檻值是多少。卡片開機時透過 `GetPDR` 將感測器清單吐給 BMC，達到隨插即用（Auto-discovery）。
* **實體線路隔離**：避免幾十張卡掛在同一個 I2C bus 上造成位址衝突或線路毀損。

---

## 4. 卡片開機與 BMC 偵測機制 (Power & Presence)

BMC 如何知道卡片通電並就緒？拆解為實體層與通訊層：

### 4.1 硬體層感知
1. **在位偵測 (PRSNT#)**：卡片插入插槽時，`PRSNT2#` 被卡片內部接地拉低，主機板 CPLD / BMC GPIO 收到邊緣觸發中斷。
2. **電源狀態**：
   * **3.3V_AUX (待機電)**：伺服器接上交流電後即供電，許多高階網卡/SmartNIC 此時 MCU 就已甦醒。
   * **Main Power (+12V / +3.3V) & PERST#**：主機按開機鈕後啟動，主板 de-assert `PERST#`，卡片主晶片進入運作。

### 4.2 MCTP 傳輸層握手
* **方法 A（卡片主動通知 - Discovery Notify）**：卡片 MCU 完成自我初始化後，往匯流排發送 `Discovery Notify`，BMC 收到後回覆 `Set Endpoint ID (Set EID)` 分配通訊位址。
* **方法 B（BMC 主動探測 - Bus Owner Probing）**：BMC 偵測到硬體在位或主機進入 S0 後，主動廣播發送 `Prepare for Endpoint Discovery` / `Get Endpoint UUID` 並指派 EID。

### 4.3 SMBus vs. PCIe VDM 通道特性比較

| 特性 | SMBus / I2C Sideband (DSP0237) | PCIe VDM (DSP0238) |
| :--- | :--- | :--- |
| **傳輸介質** | PCIe 插槽的專屬 SMBus 針腳 | PCIe 高速差分數據訊號線 (In-band) |
| **開機就緒時機** | **極早**。只要有 3.3V_AUX，主機關機狀態下亦可通訊 | **較晚**。主機必須開機且 PCIe Link 訓練完成 (L0) |
| **先決條件** | 不需要主機 CPU / BIOS 介入 | **必須等待 BIOS 完成 PCIe BDF 枚舉** (Routing by ID 需求) |
| **傳輸頻寬** | 極低 (100 kHz ~ 1 MHz) | 極高 (PCIe 封包速率) |

---

## 5. 深度剖析：PCIe VDM、Discovery Notify 與 Bus Owner

### 5.1 疑問澄清：Discovery Notify 走 PCIe VDM 嗎？BMC 是 Bus Owner 嗎？
1. **是的，Discovery Notify 在規範中確實走 PCIe VDM**。
2. **是的，BMC 是「MCTP 邏輯上的 Bus Owner」，但它「不是 PCIe 實體上的 Root Complex」**。

### 5.2 `Route to Root Complex` (000b) 定址機制
根據 **DSP0238**：
* 卡片開機後，不知道 BMC 的 BDF，因此發送 PCIe VDM 時，Header 中的 Routing 欄位設為 **`000b` (Route to Root Complex)**。
* 卡片將封包往 Upstream（朝樹頂的 CPU）拋送。
* 規範 Section 6.5 明訂：*MCTP PCIe VDM Bus Owner 的功能必須能夠透過 "Route to Root Complex" 定址存取*。

### 5.3 封包如何從 CPU (Root Complex) 送達 BMC？
在伺服器實體電路中，主要有三種硬體轉發拓撲：

```
     [ PCIe 卡 (Endpoint) ]
               │
               │ 發送 PCIe VDM (Route to Root Complex, 000b)
               ▼
      [ PCIe Switch ] ──(向上轉發)──┐
                                   ▼
                       [ Host CPU / Root Complex ]
                                   │
               ┌───────────────────┴───────────────────┐
               │ 內部硬體 VDM 濾波器 (DMTF VID: 0x1AB4) │
               └───────────────────┬───────────────────┘
                                   │
             ┌─────────────────────┼─────────────────────┐
             ▼                     ▼                     ▼
        [ 拓撲 A ]            [ 拓撲 B ]            [ 拓撲 C ]
    專屬 PCIe 轉發         專屬 eSPI 通道        PCIe Switch 側頻
   (Route by ID 給 BMC)   (eSPI MCTP 轉給 BMC)   (Switch 截獲轉給 BMC)
             │                     │                     │
             └─────────────────────┼─────────────────────┘
                                   ▼
                         [ BMC (MCTP Bus Owner) ]
                                   │
                          回覆 Set Endpoint ID
```

* **拓撲 A（Host CPU 內部 VDM Bridge）**：現代伺服器 CPU（AMD EPYC、Intel Xeon）硬體辨識 DMTF Vendor ID (`0x1AB4`)，直接透過內部路由轉送給掛在 PCIe 上的 BMC。
* **拓撲 B（eSPI 轉發）**：PCH 接收 PCIe VDM 後，轉為 eSPI MCTP 封包發給 BMC。
* **拓撲 C（智慧 PCIe Switch）**：在 GPU / NVMe 高密度架構中，PCIe Switch 自身連有 BMC 專屬管理埠，在中途即截獲並轉發給 BMC。

---

## 6. 工程實務真相與常見陷阱 (Brutal Truths & Pitfalls)

在真實伺服器產線與韌體整合時，不能只看標準規格書，必須注意以下工程陷阱：

1. **不能單壓「Discovery Notify」**：
   * 許多第三方 PCIe 介面卡的韌體品質不一，開機後**根本不會主動送出 Discovery Notify**。
   * **正解防呆**：OpenBMC / BMC 韌體必須實作後備機制，在 Host 進入 S0 且收到 BIOS 回報「PCIe Enumeration 完成」後，主動去掃描探測（Probing）。
2. **PCIe VDM 對 Host CPU 的死鎖依賴**：
   * PCIe VDM 仰賴 CPU Root Complex 的硬體暫存器被 BIOS 正確設定。若 CPU 處於 Reset、Hang 在開機階段，PCIe VDM 完全失效。
   * 重要管理訊號（如緊急電源監控）若走 PCIe VDM，在 Crash Dump 時可能無法提取，需審慎評估是否輔以 SMBus Sideband。
3. **PDR 解析開銷**：
   * 某些高階加速卡（如多 GPU 系統）的 PDR 數量可達上千筆，BMC 在拉取時若單次封包傳輸逾時，可能導致整台伺服器 Redfish 初始化停滯。需要實作快取（Cache）與非同步更新機制。
