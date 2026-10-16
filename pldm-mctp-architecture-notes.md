# PLDM & MCTP 伺服器硬體管理架構筆記 (Server Hardware Management Notes)

> 本文件彙整 BMC、BIOS、PCH 與 PCIe 擴充卡之間的 MCTP/PLDM 通訊架構、硬體訊號、標準規範與實務設計細節。
> 建議在 VS Code 中搭配 **Markdown Preview (`Ctrl + Shift + V`)** 或左側 **Outline（大綱目錄）** 快速導航閱讀。
> 註：為確保 VS Code 大綱樹穩定解析，所有標題列一律使用純文字，不嵌入反引號（inline code）。

---

## 目錄 (Table of Contents)
1. [PLDM 核心規範概覽 (DSP0245 / 0246 / 0248 / 0257)](#1-pldm-核心規範概覽-dsp0245--0246--0248--0257)
2. [DSP0246：SMBIOS 資料傳輸機制](#2-dsp0246smbios-資料傳輸機制)
3. [DSP0248 與 DSP0257：Sensor 與 FRU 的架構劃分](#3-dsp0248-與-dsp0257sensor-與-fru-的架構劃分)
4. [卡片開機與 BMC 偵測機制 (Power & Presence)](#4-卡片開機與-bmc-偵測機制-power--presence)
5. [深度剖析：PCIe VDM、Discovery Notify 與 Bus Owner](#5-深度剖析pcie-vdmdiscovery-notify-與-bus-owner)
6. [工程實務真相與常見陷阱 (Brutal Truths & Pitfalls)](#6-工程實務真相與常見陷阱-brutal-truths--pitfalls)
7. [轉發機制與算力解密：PCIe Forward vs. SoC/PCH vs. Host OS](#7-轉發機制與算力解密pcie-forward-vs-socpch-vs-host-os)
8. [PCH 獨立架構與 I2C 匯流排隔離 (Host-Owned vs. OOB-Owned)](#8-pch-獨立架構與-i2c-匯流排隔離-host-owned-vs-oob-owned)
9. [MCTP EID 分配機制與多 BMC 拓撲 (DSP0236)](#9-mctp-eid-分配機制與多-bmc-拓撲-dsp0236)
10. [MCTP 實體匯流排與 PCIe VDM TLP 深度解析](#10-mctp-實體匯流排與-pcie-vdm-tlp-深度解析)
11. [MCTP SPDM 與安全機制 (DSP0274 / DSP0276)](#11-mctp-spdm-與安全機制-dsp0274--dsp0276)
12. [高密度與 AI 伺服器的 EID 規模爆炸 (Scale & NVMe-MI)](#12-高密度與-ai-伺服器的-eid-規模爆炸-scale--nvme-mi)
13. [CXL (Compute Express Link) 架構與 Flit 模式解密](#13-cxl-compute-express-link-架構與-flit-模式解密)

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

### 5.2 Route to Root Complex (000b) 定址機制
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

---

## 7. 轉發機制與算力解密：PCIe Forward vs. SoC/PCH vs. Host OS

在理解訊息轉發（Forwarding）時，工程師最常探討：**轉發過程是否會消耗 Host CPU 的算力（CPU Cycles / Compute Power）？**

### 7.1 PCIe Forwarding：純硬體線速交換（零算力消耗）
* PCIe VDM 封包在 PCIe Switch 或 Root Complex 晶片內部的轉發，完全屬於 **PCIe Transaction Layer（交易層）** 的硬體行為。
* 晶片內部只根據 TLP Header 的 Routing 欄位進行硬體重定向（改寫為 `Route by ID` 並指向 BMC BDF）。
* **特性**：
  * **Zero CPU Cycles**：不消耗 Host CPU 任何運算時脈。
  * **No Interrupt**：不觸發 Host CPU 的 MSI/MSI-X 中斷。
  * **Host OS 完全無感知**。

### 7.2 SoC / PCH 硬體 FIFO 與 eSPI 橋接（不需 Host CPU Decode）
在現代伺服器 SoC（如 AMD EPYC 或 Intel Xeon + PCH）中，晶片內部明確區分：
1. **Host 運算核心（Compute Cores）**：跑 OS、BIOS、業務應用程式的 x86/ARM 核心。
2. **I/O Die / Uncore 硬體電路**：包含 PCIe 控制器、內部 Crossbar / NoC（Network-on-Chip）。
3. **輔助管理核心（Embedded Co-processor）**：如 AMD **PSP（Platform Security Processor / MP0）** 或 Intel **ME/SPS（Management Engine）**。

* **運作機制**：
  * PCIe 控制器內建硬體濾波器（比對 DMTF Vendor ID `0x1AB4`）。
  * 命中後直接透過硬體 DMA 串流送入專屬 FIFO，或直接轉換推給 eSPI 控制器往外送至 BMC。
  * 若需微碼協助，也是由晶片內建的獨立輔助小核心處理，**Host CPU 核心完全不參與 Decode**。

### 7.3 何時才真的會吃 Host CPU 算力？（In-Band 軟體代理）
只有在沒有硬體轉發電路、必須在 Host 作業系統內掛載驅動（如 Linux Kernel 的 `mctp-pci` 驅動）時，才會消耗 CPU 算力：
* 封包到達 ➔ 觸發 Host CPU **MSI-X 中斷** ➔ CPU 暫停業務執行中斷處理常式 ➔ **OS 核心驅動 Decode 封包** ➔ 轉拋給 BMC。
* **致命缺點**：消耗 CPU 算力、產生延遲抖動（Jitter）、且若 Host OS 當機（Kernel Panic），整套硬體監控立即中斷。

### 7.4 三種轉發方式架構比較表

| 比較維度 | 1. 純 PCIe Forward (硬體交換) | 2. SoC 硬體 FIFO / eSPI 橋接 | 3. Host OS 軟體轉發 (In-Band) |
| :--- | :--- | :--- | :--- |
| **處理單元** | PCIe Switch / RC 交易層硬體邏輯 | I/O Die 專屬硬體濾波器 / eSPI 電路 | **Host x86/ARM CPU Cores** |
| **Host CPU 算力消耗** | **0%（Zero Cycles）** | **0%（Zero Cycles）** | **被中斷打斷，需消耗 CPU 算力** |
| **Decode 發生點**| **BMC 端** 才做 Decode | **BMC 端** 才做 Decode | **Host OS 驅動** 必須先 Decode |
| **Host 當機時** | 依然正常運作 | 只要 SoC 供電正常就可運作 | **直接癱瘓** |
| **硬體代價** | 需硬體支援 PCIe 路由至 BMC | SoC 需內建硬體 VDM 濾波器邏輯 | 零硬體成本（純靠軟體苦工） |

---

## 8. PCH 獨立架構與 I2C 匯流排隔離 (Host-Owned vs. OOB-Owned)

### 8.1 PCH 電源域與獨立運作能力
* **主運算電（Main Power: VCORE / VCCIN）**：伺服器關機（ACPI S5/G3 狀態）時完全斷電。
* **待機電（Standby Power: 3.3V_AUX / Primary Well）**：只要交流電源線插上即通電。
* **PCH 內部獨立分區**：PCH 內部吃 `3.3V_AUX` 的獨立電源域（Primary Well）中，跑著獨立的微控制器（Intel ME / SPS）。
* **結論**：在主機完全關機、CPU 核心連半點電都沒有時，PCH 內部的管理引擎與專屬 I2C Master 早就已經通電全速運作。

### 8.2 PCH 內的 I2C Master：SMLink 職責
PCH 內部專為帶外管理配置的 I2C/SMBus 控制器通常稱為 **SMLink（System Management Link）**：
* **SMLink0**：直連 BMC，作為 PCH/ME 與 BMC 之間的側頻管理幹道。
* **SMLink1 / SMLink2**：連接主機板 VRM 供電晶片（PMBus）、記憶體溫度感測、以及 PCIe 插槽側頻線路。
* **自主轉發（Forwarding）**：PCH 的 ME 自主擔任 Master，向 VRM / 電源定期收集功耗與溫度（Node Manager 功能），再打包轉發給 BMC。**全程由 PCH 自帶的微小核心運算，Host CPU 負載為 0%**。

### 8.3 Host Linux 下的 i2cdetect 究竟接到什麼？
在 Host Linux 敲 `i2cdetect` 看到的是 **Host-Owned SMBus**（透過 `i2c-i801` 驅動，吃 Host CPU 算力），其連接對象受到嚴格硬體侷限：

| I2C 位址範圍 (Address) | 對應的真實硬體零件 | 用途說明 |
| :--- | :--- | :--- |
| **`0x50` ~ `0x57`** | **DDR 記憶體 SPD EEPROM** | 各 DIMM 插槽記憶體規格（容量、時序、序號）。OS 用 `decode-dimms` 讀取。 |
| **`0x18` ~ `0x1F`** | **DDR TSOD（記憶體板載測溫晶片）** | 記憶體條上的 Thermal Sensor on DIMM。 |
| **`0x70` ~ `0x77`** | **I2C Mux 切換晶片（如 PCA9548）** | 記憶體插槽眾多時，用來分 Channel 切換位址。 |
| **`0x69`** | **PCIe Clock Generator（時脈產生器）** | 產生 PCIe 100MHz 參考時脈（Refclk）的振盪晶片。 |
| **`0x20` 左右** | **PCIe Retimer / Redriver 晶片** | 少數設計將 PCIe Gen4/Gen5 訊號放大晶片掛於此處供主機端調測眼圖。 |

### 8.4 物理隔離（Physical Segregation）的工程鐵律
* **嚴禁共用同一條 I2C Bus**：
  伺服器硬體設計極力避免讓 Host OS 與 PCH/ME/BMC 共享同一條 I2C 匯流排。
  I2C 的 Multi-Master 碰撞處理能力極差，一旦兩邊同時發起傳輸，極易引發 **SDA Held Low（Bus Hang 鎖死）**，導致管理遙測中斷甚至誤觸發硬體保護斷電。
* **SMBIOS (DSP0246) 的閉環價值**：
  BMC 不需要也不應該直接去探測 Host SMBus 的記憶體 SPD。
  正確架構是由 BIOS 在開機 POST 階段，利用 Host 算力讀完所有 SPD，組成標準 **SMBIOS Table (Type 17)**，再透過 DSP0246 (PLDM over MCTP) 直接轉交給 BMC。

---

## 9. MCTP EID 分配機制與多 BMC 拓撲 (DSP0236)

### 9.1 EID 數值定義與保留區間
EID（Endpoint ID）是 **8-bit（0～255）** 邏輯位址，在一個連通的 MCTP 網路中必須全域唯一：
* **`0x00` (Null EID)**：未指派狀態。端點剛通電、尚未獲得 EID 時發送封包必須填入此值。
* **`0x01` ~ `0x07`**：DMTF 規範保留。
* **`0x08`**：通常作為 Topmost Bus Owner（主 BMC）的固定 EID。
* **`0x09` ~ `0xFE`**：一般端點與子號碼池（Pool）動態/靜態分配區間。
* **`0xFF`**：Broadcast EID（廣播位址）。

### 9.2 三大 EID 指定與分配機制
1. **動態指派（Set Endpoint ID, 0x01）**：
   * Bus Owner 向目標硬體位址（PCIe BDF 或 I2C Slave）發送。
   * 包含 Operation 參數：`00b` (Set EID)、`01b` (Force EID 強制覆蓋)、`10b` (Reset EID 清除重置)、`11b` (Set Discovered Flag 標記探索)。
2. **靜態端點（Static EID 上報）**：
   * 某些出廠寫死或硬體 Strap 固定的端點，在收到 `Set Endpoint ID` 時回覆拒絕，並主動通報自身固定 EID，由 Bus Owner 將其登記至路由表。
3. **號碼池批發（Allocate Endpoint IDs, 0x02）**：
   * 當端點為 **MCTP Bridge（例如 PCIe Switch 或擴充卡）**，可向 Bus Owner 申請連續 EID Pool（如 Size = 8），由 Bridge 轉發分配給下游子裝置。

### 9.3 路由表綁定 (Routing Table Binding)
EID 指派完成後，BMC 核心驅動（如 Linux 5.15+ 原生 MCTP 協定堆疊）會將邏輯 EID 與實體介面/位址建立映射：
```text
EID 0x08  <=====>  local (BMC 自己)
EID 0x14  <=====>  dev mctppcie0,  Target BDF: 01:00.0 (PCIe VDM)
EID 0x15  <=====>  dev mctpi2c3,   Target I2C Addr: 0x32 (SMBus)
```

### 9.4 多 BMC 串接架構：Topmost Bus Owner vs. 網段隔離
在同一個平坦的 MCTP 空間中，**絕對只能有一個 Master（Topmost Bus Owner）**，避免位址衝突：
* **架構一：Topmost Bus Owner + Bridge 模式（單一 EID 全域空間）**
  * 主 BMC 作為 Topmost Bus Owner 掌管 `0x09 ~ 0xFE` 號碼池。
  * 副 BMC 作為 MCTP Bridge，向主 BMC 批發一組 EID Pool（如 `0x20 ~ 0x2F`），分給擴充匣零件。
  * 優點：全機台平坦互通，主 BMC 可直達所有 EID。
* **架構二：網段隔離模式（Isolated MCTP Domains - 實務常見）**
  * 每個節點的 BMC 各自作為該節點的 Topmost Bus Owner。
  * 兩邊內部 EID 獨立（哪怕重複使用 `0x10` 也互不干擾）。
  * 跨節點/跨 BMC 通訊改走上層 **Redfish RESTful API、內部 Ethernet 或 Mailbox**。
  * 優點：避免 256 個 EID 上限耗盡，並達到物理故障隔離（Fault Isolation）。

---

## 10. MCTP 實體匯流排與 PCIe VDM TLP 深度解析

### 10.1 MCTP 支援的實體匯流排全貌
MCTP 為實體層無關協定，由不同的 DMTF Transport Binding 規範定義：
* **DSP0238**：**PCIe VDM**（現代高頻遙測與韌體更新主力）。
* **DSP0237**：**SMBus / I2C**（傳統側頻線與待機狀態監控）。
* **DSP0241**：**I3C**（次世代高速感測器與帶內中斷管理）。
* **DSP0239**：**KCS (LPC/eSPI)**（Host CPU/BIOS 與 BMC 通訊）。
* **DSP0283**：**USB**（機箱外接維護與內部虛擬通道）。
* **DSP0234**：**Serial (UART)**（除錯與極簡微控制器）。

### 10.2 為什麼在 PCIe 上「只能使用 VDM TLP」？
根據 **DSP0238**，MCTP 嚴格採用 **PCIe Type 1 Vendor Defined Message with Data（MsgD, Type 1）**：
1. **無需記憶體對映（No BAR / No Memory Mapping）**：
   * 一般 Memory Read/Write TLP 需要 Host BIOS/OS 分配實體記憶體空間（BAR）。
   * VDM 是 Message TLP，純封包交換，不佔用 Host 記憶體位址空間，維持帶外（Out-of-band）獨立性。
2. **內建豐富的路由標頭（Routing Capabilities）**：
   * 原生支援 `Route to Root Complex (000b)`、`Route by ID (010b)` 與 `Broadcast (011b)`。
3. **PCI-SIG 合規與防撞車**：
   * DMTF 註冊了專屬 Vendor ID：**`0x1AB4`**。硬體濾波器藉此快速辨識，不與其他廠商私有 Message 衝突。

---

## 11. MCTP SPDM 與安全機制 (DSP0274 / DSP0276)

### 11.1 MCTP Message Type 分類
MCTP 表頭使用 1 個 Byte 的 Message Type 標示負載類型：
* **`0x00`**：MCTP Control（Discovery, Set EID）。
* **`0x01`**：PLDM（DSP0240~0267，感測器、FRU、SMBIOS、韌體更新）。
* **`0x02`**：NC-SI（網卡側頻管理）。
* **`0x04`**：NVMe-MI（NVMe 儲存管理）。
* **`0x05`**：**SPDM（DSP0274，安全協定與資料模型）** ◀── 核心安全握手通道。
* **`0x06`**：**Secured MCTP（DSP0276，加密保護封包）**。

### 11.2 SPDM (DSP0274) 解決的三大安全痛點
防止偽造硬體、惡意替換卡片或韌體篡改（Supply Chain Attacks）：
1. **身分認證（Authentication）**：
   * 卡片回傳原廠私鑰簽署的 X.509 憑證鏈，BMC 透過內建原廠 Root CA 檢驗真實身分。
2. **防偽質詢（Challenge）**：
   * BMC 送出隨機亂數（Nonce），卡片以晶片硬體安全私鑰完成數位簽章並回傳，證明為合法實體。
3. **韌體度量（Attestation / Measurement）**：
   * BMC 索取卡片開機韌體 Hash 值（Measurement Record），驗證韌體未遭未授權竄改。

### 11.3 Type 0x05 (SPDM) 與 Type 0x06 (Secured MCTP) 的協同流程
```text
[ 階段一：安全交握 (Message Type 0x05 - SPDM) ]
  BMC 與 卡片 透過 Type 0x05 進行憑證驗證、質詢與安全量測
       │
       ▼ KEY_EXCHANGE
  雙方協商出對稱加密臨時會話金鑰 (Session Key: AES-GCM / ChaCha20)
       │
       ▼
[ 階段二：全程加密傳輸 (Message Type 0x06 - Secured MCTP) ]
  後續敏感資料傳輸與 PLDM 韌體更新，全部以 Type 0x06 加密並附帶 MAC 防篡改驗證
```

---

## 12. 高密度與 AI 伺服器的 EID 規模爆炸 (Scale & NVMe-MI)

### 12.1 傳統伺服器 vs. 現代 AI 伺服器的 EID 差距
* **傳統通用型伺服器**：全機只有約 10~20 個 EID（BMC 自身、BIOS、2~3 張 PCIe 卡、電源、硬碟背板 MCU）。8-bit EID（256 個空間）綽綽有餘。
* **現代 AI 伺服器 (如 8-GPU 系統)**：EID 出現指數級增長，單機可達到 80~120 個以上：
  1. **高階加速卡/SmartNIC 內建多 Endpoint**：單張卡包含主 DPU + 安全 RoT 晶片 + 內部 PCIe Switch + 光模組控制器，單卡就佔用 4~8 個 EID。
  2. **NVMe-MI 儲存端點**：24 顆 NVMe SSD 直接吃掉 24 個 EID。
  3. **800G 光模組 (CMIS over MCTP)**：每個光模組控制器獨立消耗 EID。
  4. **Rack-Scale (整機櫃架構)**：多節點串接時瞬間突破 256 個上限，這正是推動實務走向「Isolated MCTP Domains」的物理根因。

### 12.2 NVMe-MI 的協定本質
* **獨立的 Message Type (0x04)**：NVMe-MI 不是 PLDM 的子集，而是由 NVM Express 組織制定的獨立標準，與 PLDM 平起平坐。
* **Smart Device 證明**：每顆支援 NVMe-MI over MCTP 的 SSD 都自帶完整的微控制器與協定堆疊，必須分配獨立 EID。BMC 透過 `0x04` 指令直讀 Flash 磨損率、SMART 健康度與壞軌率，映射至 Redfish。

---

## 13. CXL (Compute Express Link) 架構與 Flit 模式解密

### 13.1 CXL 解決的核心痛點
* **記憶體牆 (Memory Wall)**：打破 CPU 封裝與主機板 PCB 走線限制，允許記憶體做成標準 PCIe 擴充卡。
* **快取一致性 (Cache Coherency)**：免去傳統 PCIe DMA 資料搬移與軟體快取同步的巨大開銷。
* **三合一協定**：
  * **`cxl.io`**：傳統 PCIe 協定（枚舉、配置空間、中斷、MCTP VDM）。
  * **`cxl.cache`**：裝置直接存取 CPU 快取（以極短硬體延遲向 CPU Home Agent 發起快取行協商）。
  * **`cxl.mem`**：CPU 以標準 `Load/Store` 存取外部裝置記憶體（延遲僅約 150~200 ns，逼近主記憶體）。

### 13.2 實體型態與協商機制
* **插槽與外觀完全未變**：依然插在標準 **PCIe CEM Slot (x8/x16)** 或 **EDSFF (E1.S/E3.S)** 插槽，走既有的高速差分線。
* **Alternate Protocol 協商**：開機 LTSSM 階段，CPU 與裝置透過標準機制對上暗號，實體層立刻由純 PCIe 模式切換為 CXL 模式。

### 13.3 實體傳輸層巨變：告別裸奔 TLP，全面改跑 Flit (Flow Control Unit)
CXL 協商成功後，實體鏈路不再跑傳統變長、裸奔的 TLP，而是切換為**固定大小的 Flit（如 68-Byte 或 256-Byte）**：
1. **`cxl.cache` 與 `cxl.mem`**：
   * 原生 Flit 結構，完全不使用 TLP。
   * 操作精確對齊 **64-Byte Cache Line**，無任何 TLP Header 解包延遲。
   * 定址使用標準 **Host Physical Address (HPA)**，由 CPU Home Agent 負責硬體 Snoop 窺探。
2. **`cxl.io`（保留 TLP 靈魂）**：
   * 傳統的 PCIe TLP（包含 BIOS 枚舉、設定暫存器、以及 **MCTP PCIe VDM 封包**）被硬體**切碎、打包塞進 Flit 的 Payload 貨櫃**裡運送，到達後再還原。
3. **推動標準演進**：
   * CXL 的 Flit 設計奠定了次世代匯流排標準，**PCIe 6.0 規範已正式全面廢除傳統裸奔 TLP，強制採用 256-Byte Flit 模式**。
