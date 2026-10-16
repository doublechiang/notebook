# Technical Notebook

個人技術架構與工程知識庫，專為 VS Code 閱讀與版本控制設計。

## 目錄索引 (Index)

### 硬體管理與韌體通訊 (Hardware Management & Firmware Protocols)
* [**PLDM & MCTP 伺服器硬體管理架構筆記**](pldm-mctp-architecture-notes.md)
  * DMTF PLDM 規範 (DSP0245, DSP0246, DSP0248, DSP0257)
  * SMBIOS 資料傳輸機制 (BIOS to BMC over MCTP)
  * Sensor (TMP75) 與 FRU (24C02) 的架構邊界 (Dumb Devices vs. Smart Endpoints)
  * 卡片開機、在位偵測 (PRSNT#)、電源時序 (3.3V_AUX vs. Main Power)
  * PCIe VDM、`Discovery Notify` (`000b` Route to Root Complex) 與 BMC Bus Owner 轉發拓撲
  * 工程實務陷阱與防呆機制
  * 轉發機制與算力解密 (PCIe Forward vs. SoC/PCH vs. Host OS Zero CPU Overhead)
  * PCH 獨立架構與 I2C 匯流排隔離 (Host-Owned SMBus vs. OOB-Owned SMLink / Bus Hang 防呆)
  * MCTP EID 分配機制與多 BMC 拓撲 (Topmost Bus Owner vs. 網段隔離)
  * MCTP 實體匯流排與 PCIe VDM TLP 深度解析 (無需 BAR 對映 / PCI-SIG VID 0x1AB4)
  * MCTP SPDM 與安全機制 (DSP0274 Type 0x05 身分憑證量測與 Type 0x06 加密通訊)
  * 高密度與 AI 伺服器的 EID 規模爆炸 (Scale & NVMe-MI 獨立 Message Type 0x04)
  * CXL (Compute Express Link) 架構與 Flit 模式解密 (cxl.cache/mem 原生 Flit 與 cxl.io 打包 TLP)

---

## 閱讀建議 (Viewing in VS Code)
1. **大綱導航**：在 VS Code 左側檔案總管展開 **Outline（大綱）**，即可依章節層級快速點擊跳轉。
2. **Markdown 預覽**：使用快捷鍵 `Ctrl + Shift + V` 開啟渲染視窗閱讀排版與圖表。
