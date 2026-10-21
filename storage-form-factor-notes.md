# 伺服器儲存裝置外形對照筆記 (EDSFF / U.2 / U.3 / M.2)

> 本文件對照伺服器 SSD 與 EDSFF 裝置外形：E1.S、E1.L、E2、E3.S、E3.L，以及它們要取代的 U.2 / M.2。
> 建議在 VS Code 中搭配 **Markdown Preview (`Ctrl + Shift + V`)** 或左側 **Outline（大綱目錄）** 快速導航閱讀。
> 註：為確保 VS Code 大綱樹穩定解析，所有標題列一律使用純文字，不嵌入反引號（inline code）。
> 註：每個數字後標示查證狀態 —— **[已核]** 有 SNIA 或廠商出處（見文末來源）；**[未核]** 為記憶或常見引用，未對過規格書原文。2026-10-21 整理。

---

## 目錄 (Table of Contents)
1. [一句話總結：三個維度](#1-一句話總結三個維度)
2. [EDSFF 家族與規格書編號](#2-edsff-家族與規格書編號)
3. [尺寸與功耗對照表](#3-尺寸與功耗對照表)
4. [E1.S 厚度與功耗等級](#4-e1s-厚度與功耗等級)
5. [E2 與 E3.L：長不等於大](#5-e2-與-e3l長不等於大)
6. [時間順序：編號不是時間](#6-時間順序編號不是時間)
7. [與 U.2 / U.3 / M.2 的關係](#7-與-u2--u3--m2-的關係)
8. [對製造測試 (MFG/Diag) 的影響](#8-對製造測試-mfgdiag-的影響)
9. [待查證事項](#9-待查證事項)
10. [來源](#10-來源)

---

## 1. 一句話總結：三個維度

EDSFF 各型號的差異沿三個維度展開：

* **高度** → 插 1U 還是 2U：E1 矮（約 31.5 mm），E2 / E3 高（76 mm）。
* **長度** → 能鋪多少 NAND：.S 短，.L 與 E2 長。
* **厚度** → 散熱與功耗等級：E3 的 1T / 2T，E1.S 的 5.9 ~ 25 mm 各級。

所有 EDSFF 共用 **SFF-TA-1002 卡緣式 (card-edge) 連接器**，腳位與訊號定義在 **SFF-TA-1009**。

---

## 2. EDSFF 家族與規格書編號

| 型號 | 規格書 | 定位 | 狀態 |
| :--- | :--- | :--- | :--- |
| **E1.S** | SFF-TA-1006 | 1U 高密度 NVMe、取代伺服器內 M.2（開機碟、hyperscale） | [已核] |
| **E1.L** | SFF-TA-1007 | 1U 長條「Ruler」，衝容量 | [未核] |
| **E2** | **SFF-TA-1042**（Rev 1.0，2025-06-16） | 超高容量 QLC，≥ 64 顆 NAND package，直立插 2U | [已核] |
| **E3.S / E3.L** | SFF-TA-1008 | 取代 U.2；SSD、CXL 記憶體、加速卡、NIC | [已核] |

* **沒有「E2.S」**，E2 只有一種長度。看到 E2.S 多半是筆誤或廠商的內部叫法。
* SFF-TA-1008 在 Rev 2.1.2（2025-12）更新了 power/thermal 需求，並註明 Rev 3.0 起，與 Rev 2.1 以前的裝置可能不相容 [已核]。**舊資料裡的 E3 功耗數字可能已經過時。**

---

## 3. 尺寸與功耗對照表

| 型號 | 高 × 長 (mm) | 厚度 (mm) | 介面 | 功耗上限 |
| :--- | :--- | :--- | :--- | :--- |
| E1.S | 31.5 × 111.49（PCB / 5.9 mm 版）[已核] | 5.9 / 8.01 / 9.5 / 15 / 25 | PCIe x4 (1C) | 12 ~ 25 W，依厚度（見第 4 節） |
| E1.L | 約 38.4 × 318.75 [未核] | 9.5 / 18 [未核] | PCIe x4 | [未核] |
| E2 | 76 × 200 [已核] | 9.5 [已核] | PCIe x4 [未核] | **查無數字**，規格書 Table 7-1 有定義 |
| E3.S 1T | 76 × 112.75 [已核] | 7.5 | x4 / x8 / x16 (1C/2C/4C) | 25 W [已核，舊版] |
| E3.S 2T | 76 × 112.75 [已核] | 16.8 | 同上 | 40 W [已核，舊版] |
| E3.L 1T | 76 × 142.2 [已核] | 7.5 | 同上 | 約 40 W [未核] |
| E3.L 2T | 76 × 142.2 [已核] | 16.8 | 同上 | 70 W [已核，舊版] |
| U.2 | 2.5"：69.85 × 100.45 [未核] | 7 / 15 | PCIe x4（或雙埠 x2+x2），SFF-8639 | 約 25 W [未核] |

* 供電：EDSFF 為 12 V 主電源加 3.3 V Aux [未核]。
* E3 的 2T 插槽可插兩片 1T 或一片 2T [未核]。
* 「舊版」表示數字出自 SFF-TA-1008 Rev 2.1.2 以前的資料（Kioxia 頁面、SNIA 2022 簡報）。

---

## 4. E1.S 厚度與功耗等級

| 厚度 | 外觀 | 建議功耗 | 狀態 |
| :---: | :--- | :---: | :--- |
| 5.9 mm | 裸板 | 12 W | [已核] |
| 8.01 mm | Heat spreader | 16 W | [已核，僅 SNIA 簡報] |
| 9.5 mm | 對稱外殼（外殼 33.75 × 118.75 mm） | 20 W | [已核] |
| 15 mm | 散熱片 | 20 W | [已核，僅 SNIA 簡報] |
| 25 mm | 非對稱散熱片 | 25 W | [已核，僅 SNIA 簡報] |

* 上表是 SNIA 簡報與 Kioxia 資料給的**建議 (recommended) 功耗**，不是規格書的硬性上限。
* 厚度是在挑散熱能力：hyperscaler 1U 常用 9.5 / 15 mm，25 mm 給高功耗或空間鬆的設計。

---

## 5. E2 與 E3.L：長不等於大

| 型號 | 高 × 長 × 厚 (mm) | 體積 |
| :--- | :--- | :---: |
| E3.L 1T | 76 × 142.2 × 7.5 | 約 81 cm³ |
| **E2** | 76 × 200 × 9.5 | 約 144 cm³ |
| E3.L 2T | 76 × 142.2 × 16.8 | 約 182 cm³ |

* **E2 比 E3.L 長、高度一樣，但比 E3.L 2T 薄**，體積介於 1T 和 2T 之間。
* 設計取向不同：E2 **又長又薄**，在單片 PCB 上平鋪最多 NAND，並在 2U 前面板並排最多片；E3 2T 加厚是為了散熱與高功耗，給加速卡、CXL 用。
* 最長的是 **E1.L**（約 318.75 mm [未核]），但它很矮，體積不大。

---

## 6. 時間順序：編號不是時間

| 時間（約） | 型號 | 背景 |
| :--- | :--- | :--- |
| 2017 | E1.L | Intel「Ruler」SSD，EDSFF 起點 [未核] |
| 2018 前後 | E1.S、E3 | 與 E1.L 同期定案；E3 由 Dell / HPE 等 OEM 推動，用來取代 U.2 [未核] |
| **2025-06** | **E2** | SFF-TA-1042 Rev 1.0 發布 [已核]；QLC 成熟後才出現的「吃 HDD 市場」容量層 |

* 順序是 **E1 / E3 → E2**，不是 E1 → E2 → E3。
* 編號的由來（是否對應機箱高度）沒有可靠出處，**不要拿來推理**。

---

## 7. 與 U.2 / U.3 / M.2 的關係

### 7.1 對應關係
* **U.2 → E3.S**：尺寸等級相近，是直接的接班人。**兩者機構與電氣都不相容**：連接器（SFF-8639 vs. SFF-TA-1002）、背板、托架全部不同。
* **M.2 →（伺服器內）E1.S**：E1.S 支援 hot-plug、散熱較好。
* **U.3 (SFF-TA-1001)**：與 EDSFF 平行的另一條路。保留 2.5" 外形，背板做 tri-mode（同一個槽吃 SAS / SATA / NVMe），目的是延長 2.5" 生態，不是走向 EDSFF。

### 7.2 為什麼要換掉 U.2
U.2 的外形是為了讓 NVMe 沿用 HDD 托架和背板而保留的，不是為 flash 從頭設計：

| 問題 | U.2 | EDSFF |
| :--- | :--- | :--- |
| 散熱 | 封閉 2.5" 外殼，風道差 | 從一開始就按風道設計 |
| 功耗 | 約 25 W | E3.S 2T 40 W、E3.L 2T 70 W（舊版數字） |
| 頻寬 | 最多 x4 | 最多 x16 |
| PCIe Gen5 / Gen6 訊號 | SFF-8639 在高速下吃力 | 卡緣式連接器為高速設計 |
| 裝置種類 | 只有 SSD | SSD、CXL 記憶體、加速卡、NIC |
| 管理 | 背板 SES / VPP 控 LED | LED、present、sideband 寫進 SFF-TA-1009 |

---

## 8. 對製造測試 (MFG/Diag) 的影響

> 本節為推論，尚未對照任何實際機種的測項驗證。

* **治具與 carrier 全部重做**：背板、托架、連接器都和 U.2 不同。
* **NVMe 協定層不變**：nvme-cli、FIO、SMART 這類測項基本上可以沿用。
* **會變的是 sideband**：LED、present、PERST#、CLKREQ#、SMBus / MCTP 管理路徑。原本靠 U.2 背板 VPP / SES 做的 locate / fault LED 測項要重寫。NVMe-MI over MCTP 見 [PLDM & MCTP 筆記第 12 節](pldm-mctp-architecture-notes.md#12-高密度與-ai-伺服器的-eid-規模爆炸-scale--nvme-mi)。
* **同一個 E3.S 槽可能插的是 CXL 記憶體**（例如 CXL Type 3 記憶體擴充模組），測試流程完全不同，要先判斷裝置類型再分流。CXL 協定見 [PLDM & MCTP 筆記第 13 節](pldm-mctp-architecture-notes.md#13-cxl-compute-express-link-架構與-flit-模式解密)。
* **功耗與散熱驗證要看規格版本**：SFF-TA-1008 Rev 3.0 的 power/thermal 改動可能讓新舊 E3 裝置不相容，量測功耗時先確認平台按哪一版設計。

---

## 9. 待查證事項

- [ ] E2 功耗上限（SFF-TA-1042 Rev 1.0 Table 7-1）
- [ ] E3 在 SFF-TA-1008 Rev 2.1.2 / 3.0 的新功耗表
- [ ] E1.L 尺寸、厚度、功耗（SFF-TA-1007）
- [ ] E1.S 8.01 / 15 / 25 mm 功耗在 SFF-TA-1006 原文的數字（目前只有 SNIA 簡報）
- [ ] U.2 尺寸與功耗（SFF-8201 / SFF-8639）
- [ ] E1 / E3 確切推出年份

---

## 10. 來源

* SNIA CMSI Form Factors：<https://snia.org/forums/cmsi/knowledge/formfactors>
* SNIA SSDS-201（2022，E3 尺寸）：<https://www.snia.org/sites/default/files/SSSI/2022-0803_SSDS-201-1_Brett.pdf>
* SNIA EDSFF（SDC EMEA 2021，E1.S 功耗等級）：<https://www.snia.org/sites/default/files/SDCEMEA/2021/snia-edsff.pdf>
* SNIA CMSS24 The Latest on Form Factors：<https://www.snia.org/sites/default/files/SSSI/CMSS24/CMSS24-Hands-The-Latest-on-Form-Factors.pdf>
* Kioxia EDSFF E1：<https://americas.kioxia.com/en-us/business/ssd/solution/edsff/e1.html>
* Kioxia EDSFF E3：<https://europe.kioxia.com/en-mea/business/ssd/solution/edsff/e3.html>
* SNIA 規格書（SFF-TA-1042 等）：SNIA members 文件庫，公開審閱版見 SNIA specifications 頁面
