# 伺服器液冷架構筆記 (Cold Plate / UQD / CDU / Leak Detection)

> 本文件整理伺服器冷板式 (cold plate / direct-to-chip) 液冷的元件、接頭標準、迴路架構與 BMC 管理介面，以及對製造測試的影響。
> 建議在 VS Code 中搭配 **Markdown Preview (`Ctrl + Shift + V`)** 或左側 **Outline（大綱目錄）** 快速導航閱讀。
> 註：為確保 VS Code 大綱樹穩定解析，所有標題列一律使用純文字，不嵌入反引號（inline code）。
> 註：每個數字後標示查證狀態 —— **[已核]** 有 OCP / DMTF 或廠商出處（見文末來源）；**[未核]** 為記憶或推論。廠商數字只代表該廠商，不等於標準。2026-10-21 整理。

---

## 目錄 (Table of Contents)
1. [整體架構：兩個迴路與一個 CDU](#1-整體架構兩個迴路與一個-cdu)
2. [伺服器端元件：冷板到接頭](#2-伺服器端元件冷板到接頭)
3. [UQD 與 UQDB 快速接頭 (OCP)](#3-uqd-與-uqdb-快速接頭-ocp)
4. [CDU：In-rack vs. In-row](#4-cduin-rack-vs-in-row)
5. [冷卻液 (Coolant)](#5-冷卻液-coolant)
6. [漏液偵測與 BMC 管理 (Redfish)](#6-漏液偵測與-bmc-管理-redfish)
7. [對製造測試 (MFG/Diag) 的影響](#7-對製造測試-mfgdiag-的影響)
8. [待查證事項](#8-待查證事項)
9. [來源](#9-來源)

---

## 1. 整體架構：兩個迴路與一個 CDU

```
[ 機房側 FWS (Facility Water System) ]   ← 冰水主機 / 冷卻水塔，通常是純水
             │
             ▼
     [ CDU (Coolant Distribution Unit) ]   ← 熱交換器 + 泵 + 過濾 + 控制；兩側液體不混合
             │
             ▼
[ IT 側 TCS (Technology Cooling System) ]  ← 常見 PG25；壓力、潔淨度要求比 FWS 嚴
             │
   [ Rack Manifold（機櫃分歧管）]
             │  UQD（手插）或 UQDB（盲插）
             ▼
   [ 伺服器內管路 → Cold Plate（CPU / GPU / 交換晶片）]
```

* **CDU 的工作是隔離**：把機房水和 IT 側冷卻液分開，只透過熱交換器傳熱 [已核]。
* 所以伺服器只會接觸到 **TCS**。製造端治具模擬的也是 TCS（流量、溫度、壓力、液體種類）。

---

## 2. 伺服器端元件：冷板到接頭

| 元件 | 作用 | 測試關注點 [未核] |
| :--- | :--- | :--- |
| **Cold plate（冷板）** | 貼在 CPU / GPU 上，內部微流道帶走熱 | 與晶片的接觸（TIM、鎖附扭力）、流道堵塞 |
| **管路 (tubing / hose)** | 連接冷板與機箱外的接頭 | 彎折、接頭壓接、異物 |
| **UQD / UQDB** | 機箱和 manifold 間的快速接頭 | 漏液、插拔壽命、端面污染 |
| **Leak sensor（漏液感測）** | 感測線或感測點，接到 BMC | 誤報、斷線、BMC 事件是否正確 |
| **Flow / pressure / temp sensor** | 部分設計有，回報 BMC | 讀值合理性、門檻 |

* 冷板式液冷通常只冷 CPU / GPU 這些大熱源，DIMM、VR、NIC、SSD 仍要風扇，是「液氣混合」(hybrid) [未核]。

---

## 3. UQD 與 UQDB 快速接頭 (OCP)

### 3.1 名稱
* **UQD = Universal Quick Disconnect**：由 OCP 標準化，讓不同廠商的公母頭可以互插 [已核]。
* **UQD**：手插 (hand-mate)，常用在 manifold 與伺服器之間。
* **UQDB**：B = **Blind-mate**，伺服器推進機櫃時自動對接，整櫃液冷（例如 GB200 NVL72 這類）常用 [未核：舉例未逐項確認]。
* **規格版本**：OCP UQD v1.0 與 UQDB v1.0 原本是兩份文件；**v2.0.0（生效日 2026-09-22）合併成一份**，讓各尺寸、各型態的關鍵性能參數一致 [已核]。新設計要先確認平台引用的是哪一版。

### 3.2 尺寸
v1.0 定義四種可互換的介面尺寸：**UQD02 / UQD04 / UQD06 / UQD08** [已核]。v2.0 是否沿用同樣四種尺寸，尚未確認。

| 尺寸 | Cv（流量係數，Amphenol） | 每次插拔殘液（Amphenol） |
| :---: | :---: | :---: |
| UQD02 | 0.4 | 0.004 ~ 0.01 ml 範圍內 |
| UQD04 | 1.29 | 同上 |
| UQD06 | 2.32 | 同上 |
| **UQD08** | **3.7** | 約 0.01 ml |

* **UQD08** = 1/2" 流量等級（CPC）[已核]；FITOK 標示等效孔徑 0.39 in (10 mm) [已核]。
* 編號是否以 1/16 吋為單位（08/16 = 1/2"）**是推論** [未核]。

### 3.3 共通特性（廠商 datasheet，非 OCP 原文）
| 項目 | 數值 | 出處 |
| :--- | :--- | :--- |
| 設計 | Dry-break，平面式斷流閥，拔開時兩端自動封閉 | 多家 |
| 殘液量 | 約 0.01 ml / 最多 0.05 ml | Amphenol / FITOK |
| 插接力 (UQD08) | 100 ± 10 N | FITOK |
| 插拔壽命 | > 5000 次 | FITOK |
| 工作壓力 | 0 ~ 0.6 MPa（Amphenol）；典型 6.9 bar（Danfoss） | 差異大 |
| 溫度 | -40 ~ 105 °C（Amphenol）；-40 ~ 150 °C（Danfoss） | 差異大 |
| 材質 | 303 / 304 不鏽鋼本體 + EPDM 密封 | 多家 |

> **廠商之間的數字差很多**，寫進測試規格前要以 OCP 規格書原文和實際採購料號的 datasheet 為準。

---

## 4. CDU：In-rack vs. In-row

| 型態 | 位置 | 容量量級（廠商例） |
| :--- | :--- | :--- |
| **In-rack CDU** | 裝在機櫃內，常見 4U | 約 150 ~ 250 kW（Rittal、Nidec）[已核] |
| **In-row CDU** | 獨立一櫃，放在機櫃列中，供多櫃 | 約 1,000 ~ 1,500 kW（Rittal）[已核] |

* 兩者都要透過熱交換器接 FWS [已核]。
* 另有 Liquid-to-Air CDU（不接機房水，靠風扇排熱），用在沒有機房水的場地 [未核]。
* 廠商例（Rittal in-rack）：FWS 10 ~ 50 °C / 最高 10 bar；TCS 20 ~ 55 °C / 最高 5 bar；二次側過濾 25 µm [已核，單一廠商]。

---

## 5. 冷卻液 (Coolant)

* **PG25**：約 25% 丙二醇 (propylene glycol) 水溶液，用於 TCS，提供防凍與防腐蝕 [未核：濃度為一般知識]。
* FWS 通常是處理過的純水；TCS 常用 PG25 [已核，單一廠商例]。
* **冷卻液種類會影響**：比熱與黏度（流量、壓損、散熱表現）、導電度（漏液感測器的行為）、與密封材料的相容性 [未核]。
* 製造端治具使用的液體若和客戶現場不同（例如治具用純水、現場用 PG25），量測到的溫度與壓損不能直接對照 [未核：推論]。

---

## 6. 漏液偵測與 BMC 管理 (Redfish)

* DMTF Redfish 有 **LeakDetection**（目前 v1_2_0）與 **LeakDetector** schema（v1_0_0 起於 2023.1 release）[已核]。
* 現行路徑：`/redfish/v1/Chassis/{ChassisId}/LeakDetectors/{LeakDetectorId}`；舊的 ThermalEquipment / ThermalSubsystem 路徑已標示 deprecated [已核]。
* **Redfish 2024.4** 把 LeakDetectors 加到 Chassis 底下統一位置，並新增壓力、流量、液位的 Environmental message [已核]。
* 廠商可能停在舊版 schema：例如 Lenovo SMM3 用 `/redfish/v1/Chassis/{ChassisName}/CoolingUnit/LeakDetection`、LeakDetection.v1_0_1 [已核]。**測試腳本不要寫死路徑**，要從 Chassis 往下探索。
* CDU 本身的管理模型見 DMTF DSP2064（Redfish for Cooling Units）[已核]。
* BMC 與 sensor 之間的通訊（I2C / MCTP / PLDM）見 [PLDM & MCTP 筆記](pldm-mctp-architecture-notes.md)。

---

## 7. 對製造測試 (MFG/Diag) 的影響

> 本節為推論，尚未對照任何實際機種的測項或工站流程驗證。

* **漏液測試是新的必測項，也是新的工站**：通常用氣壓或氦氣保壓，量壓降。它和電氣測試的設備、節拍、判定方式完全不同。
* **燒機溫度異常的懷疑範圍要擴大**：GPU 溫度高，除了風扇與 TIM，還要查流量、入水溫度、冷板鎖附、流道堵塞。測試 log 要能記到治具側的流量與水溫，否則無法區分是 UUT 的問題還是治具的問題。
* **治具 UQD 是耗材**：反覆插拔有壽命（廠商標 > 5000 次），要計次管理；端面異物會造成密封不良，被誤判為 UUT 漏液。
* **BMC 漏液事件要驗證兩個方向**：真的漏時要報（感測線接通測試），沒漏時不能誤報；路徑依 schema 版本不同，腳本要探索而非寫死。
* **液體與客戶現場一致**：治具液體種類、溫度、壓力若和客戶 TCS 規格不同，散熱與壓損結果不能直接當成出貨依據。
* **機台搬運與殘液**：出貨前是否要排空、吹乾，殘液量要求依客戶規範（dry-break 接頭仍有微量殘液）。

---

## 8. 待查證事項

- [ ] OCP UQD v2.0.0 的尺寸表與性能要求（是否仍為 02 / 04 / 06 / 08、壓力、殘液上限）
- [ ] UQD 編號與口徑的對應規則（1/16 吋假設）
- [ ] UQDB 在公開整櫃液冷參考設計中的使用情形
- [ ] PG25 的標準濃度與規範來源
- [ ] OpenBMC 對 LeakDetector 的實作與 schema 版本
- [ ] 業界常見的液冷漏液測試方法（氣壓 vs. 氦氣）與判定標準

---

## 9. 來源

* OCP Design Spec UQD v2.0.0：<https://www.opencompute.org/wp-content/uploads/2026/10/OCP-Design-Spec_UQD_v2.0.0.pdf>
* OCP Liquid-to-Liquid CDU Test Methodology：<https://www.opencompute.org/documents/ocp-wp-l-lcdu-test-methodology-performance-rating-r1-pdf/>
* CPC UQD08 / UQDB08：<https://www.cpcworldwide.com/global/cn/Liquid-Cooling/Everis/UQD08-UQDB08>
* Amphenol UQD / UQDB catalog：<https://www.amphenol.co.jp/military/catalog/UQD_UQDB.pdf>
* FITOK UQD Series：<https://www.fitok.com/tmp/file/Liquid_Cooling_Universal_Quick_Disconnect_UQD_Series_EN.pdf>
* Danfoss UQD datasheet：<https://assets.danfoss.com/documents/514077/AI423953312825en-000401.pdf>
* Rittal in-rack CDU SK 3343.230：<https://www.rittal.com/pdf-creator/variant/ca-en/3343230>
* Rittal in-row CDU SK 3343.200：<https://www.rittal.com/pdf-creator/variant/ca-en/3343200>
* Nidec In-rack CDU：<https://www.nidec.com/en/product/liquid-cooling-line-up/products/in-rack/>
* DMTF Redfish LeakDetection v1_2_0：<https://redfish.dmtf.org/schemas/v1/LeakDetection.v1_2_0.json>
* DMTF Redfish LeakDetector：<https://redfish.dmtf.org/schemas/v1/LeakDetector_v1.xml>
* DMTF Redfish Release 2024.4 Overview：<https://dmtf.org/sites/default/files/Redfish%20Release%202024.4%20Overview.pdf>
* DMTF DSP2064 Redfish for Cooling Units：<https://dmtf.org/sites/default/files/standards/documents/DSP2064_1.1.0.pdf>
* Lenovo SMM3 LeakDetection：<https://pubs.lenovo.com/smm3-restapi/resource_leakdetection>
