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
  * EID 空間耗盡的四大產業界實務解法 (Isolated Domains, Dynamic Reset, Bridge, 16-bit EID)
  * 附錄：PCIe Enumeration 枚舉階段資源分配深度剖析 (BDF、BAR 探測原理與 VDM 路由依賴)

### 平台安全與信任根技術 (Platform Security & Root of Trust)
* [**TPM 2.0 架構與可信賴計算技術筆記**](tpm-architecture-notes.md)
  * TCG TPM 2.0 規範與密碼學敏捷性 (Crypto Agile vs. TPM 1.2)
  * 實體型態 (dTPM, fTPM, vTPM) 與匯流排介面 (TIS 1.3 FIFO vs. PTP CRB Shared Memory)
  * 晶片儲存真相與極限：Key Wrapping 模型 (無限子金鑰)、NVRAM 容量 (7~16 KB)、兩大 NV 存儲型態 (Raw NV Index 0x01 vs. Persistent Key 0x81)、憑證與金鑰 Slot 限制
  * 四大獨立階層架構 (Platform, Storage, Endorsement, Null) 與 Primary Seeds 衍生模型
  * 平台配置暫存器 (PCR) 數學 Extend 原理、單一指令約束 (非 Range)、開機時序交錯真相與 0~23 分配表
  * 信任鏈 (Chain of Trust) 傳遞時序：CRTM 燒錄器攻擊與矽晶片信任根 (Silicon RoT)、eFuse 驗證 IBB 邊界 (NEM 快取限制)、雙軌接力 (Verified vs. Measured Boot)、NVRAM 盲區防禦與 TCG Event Log 回放驗證
  * 放行機制判定：本地封印 (Local Sealing) vs 遠端證明 (Remote Attestation) 與 BIOS/OS 升級雙面刃解法
  * Linux 核心節點 (`/dev/tpm0` vs. `/dev/tpmrm0`) 與 TSS 2.0 軟體堆疊 (FAPI/ESAPI/SAPI/TCTI)
  * 製造測試 (MFG/Diag) 檢測清單：PPI 實體在場、EK 憑證檢驗、Self-Test 故障鎖定
  * 工程陷阱與除錯心法：PCR 飄移 (PCR Drift)、SPI Bus Hang 與 Lockout 計數器

### 核心架構與系統工程 (Kernel Architecture & System Engineering)
* [**Linux 核心電源管理與高效能 I/O 多路復用架構筆記**](linux-system-architecture-notes.md)
  * CPU 電源管理核心概念：C-state (Idle 睡眠) 與 P-state (Active DVFS) 層級架構
  * x86 `HLT` 指令本質與 C-state 關閉狀態下的硬體行為 (C1 Clock Gating vs. `idle=poll` C0 鎖定)
  * 為什麼需要 P-state：Race-to-Sleep (Race-to-Idle) 假說與四大物理限制 ($P \propto C \cdot V^2 \cdot f$ 電壓平方懲罰、Memory/IO 延遲等公車效應、TjMax 散熱牆與機房功率封頂)
  * 伺服器製造測試 (MFG/Diag) 調優：停用 C-state (消除 Jitter 與 di/dt 突波) 與鎖定 P-state (基準可重現性)
  * I/O 多路復用演進與瓶頸：`select` (Bitmap 1024 限制、全量拷貝、雙重 O(N) 遍歷) 與 `poll` (`struct pollfd` 動態陣列)
  * 現代高效能基石：`epoll` 核心雙結構 (紅黑樹 O(log N) 管理 vs. 就緒雙向鏈結串列 O(1)~O(K) 提取) 與 LT/ET 觸發模式
  * 信號競爭 (Race Condition) 陷阱：`select` 盲區與 `pselect` 原子性信號遮罩 (Atomic Sigmask) 原理
  * 核心機制對比總結表 (C-state vs. P-state、select vs. poll vs. epoll)

### 網路與資料傳輸 (Networking & Data Transport)
* [**RDMA 三項核心特性筆記**](rdma-core-properties-notes.md)
  * RDMA 的定義 (Remote Direct Memory Access，對端 CPU 不介入)
  * Zero-copy 與 Memory Registration (lkey / rkey、GPUDirect RDMA)
  * Kernel Bypass：控制路徑 vs. 資料路徑 (kernel 內 RDMA 反例：NVMe-oF、iSER、SMB Direct)
  * Hardware Transport：InfiniBand / RoCE v2 / iWARP，及軟體實作 Soft-RoCE (rxe)、siw
  * 總整理：三項特性與定義的關係
  * GPUDirect RDMA 控制流程 (CPU 發起 vs. GPU-initiated IBGDA) 與 PCIe 拓撲 / ACS 測試陷阱
  * RDMA 網卡 vs. DPU：卸載傳輸 vs. 卸載基礎設施，BF-4 使用場景與測試差異

---

## 閱讀建議 (Viewing in VS Code)
1. **大綱導航**：在 VS Code 左側檔案總管展開 **Outline（大綱）**，即可依章節層級快速點擊跳轉。
2. **Markdown 預覽**：使用快捷鍵 `Ctrl + Shift + V` 開啟渲染視窗閱讀排版與圖表。
