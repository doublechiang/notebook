# TPM 2.0 架構與可信賴計算技術筆記 (Trusted Platform Module Architecture Notes)

> 本文件系統化彙整 TCG TPM 2.0 規範、硬體架構、密碼學信任根、開機度量、金鑰層級、遠端證明與工廠製造測試（MFG / Diag）實務細節。
> 建議在 VS Code 中搭配 **Markdown Preview (`Ctrl + Shift + V`)** 或左側 **Outline（大綱目錄）** 快速導航閱讀。
> 註：為確保 VS Code 大綱樹穩定解析，所有標題列一律使用純文字，不嵌入反引號（inline code）。

---

## 目錄 (Table of Contents)
1. [TPM 核心標準與架構概覽 (TCG 2.0 vs 1.2)](#1-tpm-核心標準與架構概覽-tcg-20-vs-12)
2. [實體型態與硬體介面 (dTPM vs fTPM vs vTPM)](#2-實體型態與硬體介面-dtpm-vs-ftpm-vs-vtpm)
3. [匯流排通訊協議：TIS 1.3 與 PTP CRB 深度解析](#3-匯流排通訊協議tis-13-與-ptp-crb-深度解析)
4. [晶片內部核心組件與儲存架構真相](#4-晶片內部核心組件與儲存架構真相)
5. [TPM 2.0 階層架構與金鑰管理體系 (Hierarchies & Keys)](#5-tpm-20-階層架構與金鑰管理體系-hierarchies--keys)
6. [平台配置暫存器 (PCR) 與度量數學原理](#6-平台配置暫存器-pcr-與度量數學原理)
7. [信任鏈與度量引導 (Measured Boot & TCG Event Log)](#7-信任鏈與度量引導-measured-boot--tcg-event-log)
8. [放行機制判定：本地封印 vs 遠端證明](#8-放行機制判定本地封印-vs-遠端證明)
9. [遠端證明架構 (Remote Attestation & Quote)](#9-遠端證明架構-remote-attestation--quote)
10. [Linux 驅動層與 TSS 2.0 軟體堆疊](#10-linux-驅動層與-tss-20-軟體堆疊)
11. [製造測試與量產維運實務 (Manufacturing, Diag & Provisioning)](#11-製造測試與量產維運實務-manufacturing-diag--provisioning)
12. [工程實務陷阱與除錯心法 (Brutal Truths & Pitfalls)](#12-工程實務陷阱與除錯心法-brutal-truths--pitfalls)

---

## 1. TPM 核心標準與架構概覽 (TCG 2.0 vs 1.2)

TPM（Trusted Platform Module）是由 **TCG（Trusted Computing Group）** 制定並獲 ISO/IEC 採納（ISO/IEC 11889）的國際標準硬體安全協同處理器，核心目標是為運算平台提供**硬體級別的信任根（Hardware Root of Trust, RoT）**。

### 1.1 TPM 1.2 與 TPM 2.0 的本質代差

TPM 2.0 並非 TPM 1.2 的向下相容升級，而是整個架構體系的全面重構：

| 維度 | TPM 1.2 (Main Specification) | TPM 2.0 (Library Specification) | 伺服器與架構意義 |
| :--- | :--- | :--- | :--- |
| **密碼演算法靈活性** | 演算法固化（固定 SHA-1、RSA-2048） | **Crypto Agile（密碼學敏捷性）**：支援 SHA-256/384/512, ECC (P-256, SM2), RSA, AES, Camellia | 消除 SHA-1 碰撞危機，支援現代後量子/國密過渡需求 |
| **階層體系 (Hierarchies)** | 單一 Owner 掌控全域 | **四大獨立階層**：Platform (韌體), Storage (Owner), Endorsement (隱私), Null (臨時) | 雲端多租戶與 OEM/ODM 權責切分，平台韌體與 OS 擁有者互不干擾 |
| **授權機制** | AuthData (單純密碼 / HMAC) | **Enhanced Authorization (EA / Policy)**：支援複雜布林邏輯、PCR 狀態、時間、物理授權 | 可精確限制金鑰只能在「特定 PCR + 特定實體按鈕按下」條件下解鎖 |
| **資源管理** | 晶片內 Slot 數量極少，溢出直接報錯 | 引進標準 **Resource Manager (TRM)** 模型，支援 Context 加密 Swap-out | 允許多個行程與虛擬機併發使用 TPM 資源 |
| **PCR Bank** | 僅單一 SHA-1 Bank (24 個) | **多 Bank 並行**（如 SHA-1 Bank, SHA-256 Bank, SHA-384 Bank） | 同一硬體狀態度量可同時以多種雜湊演算法記錄在不同 Bank |

---

## 2. 實體型態與硬體介面 (dTPM vs fTPM vs vTPM)

在硬體產品與伺服器架構中，TPM 有三種截然不同的落地實體：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           TPM 實體型態對比                               │
├─────────────────────┬─────────────────────┬─────────────────────────────┤
│ 1. Discrete (dTPM)  │ 2. Firmware (fTPM)  │ 3. Virtual (vTPM)           │
├─────────────────────┼─────────────────────┼─────────────────────────────┤
│ 獨立實體防竄改晶片   │ 跑在主晶片 TEE 內    │ 軟體模擬實例                │
│ (Infineon, Nuvoton) │ (Intel PTT, AMD PSP)│ (KVM / QEMU swtpm)          │
│                     │                     │                             │
│ ┌─────────────────┐ │ ┌─────────────────┐ │ ┌─────────────────────────┐ │
│ │  Dedicated IC   │ │ │ Main SoC / PCH  │ │ │ Hypervisor (Host OS)    │ │
│ │  + Crypto Core  │ │ │  ┌────────────┐ │ │ │  ┌───────────────────┐  │ │
│ │  + Shielded NV  │ │ │  │ ARM TZ /   │ │ │ │  │ swtpm daemon      │  │ │
│ └────────┬────────┘ │ │  │ Intel CSME │ │ │ │  │ (State file encrypted││
│          │          │ │  └────────────┘ │ │ │  └─────────┬─────────┘  │ │
│          ▼          │ └────────┬────────┘ │ └────────────┼────────────┘ │
│   SPI / I2C / LPC   │          ▼          │              ▼              │
│   實體匯流排引腳    │     內部 Mailbox    │       Virtual MMIO / CRB    │
└─────────────────────┴─────────────────────┴─────────────────────────────┘
```

### 2.1 各型態特性與架構考量

1. **dTPM（獨立專用晶片，如 Infineon SLB9670 / SLB9672, Nuvoton NPCT75x）**：
   * **優點**：硬體具備主動防護網（Active Shield）、金屬層保護、側通道攻擊（DPA/SPA）防護與物理防探針（Tamper-proof），獲 FIPS 140-2/3 Level 3 與 Common Criteria EAL4+/EAL5+ 認證。
   * **缺點**：BOM 成本高、佔用主機板面積、外部匯流排（SPI/I2C）存在探針竊聽風險（Bus Interception Attack，若未啟用 Parameter Encryption）。
   * **適用場景**：企業級伺服器（Dell, HPE, Quanta/QCI）、金融硬體、電信設備。

2. **fTPM（韌體型 TPM，如 Intel PTT, AMD PSP, ARM TrustZone/OP-TEE）**：
   * **優點**：零額外硬體成本，通訊全走 SoC 內部晶片總線，**完全免疫實體匯流排探針竊聽**。
   * **缺點**：安全性與主 CPU/SoC 的韌體安全漏洞（如 ME/PSP 漏洞）強綁定；若 SoC 損壞更換主機板，內嵌金鑰無法移植。

3. **vTPM（虛擬化 TPM）**：
   * **架構**：Hypervisor 為每個 VM 派生一個虛擬 TPM 實例（如 QEMU `swtpm`），其 NVRAM 資料經 Host 主機金鑰加密後存於實體儲存裝置。

---

## 3. 匯流排通訊協議：TIS 1.3 與 PTP CRB 深度解析

Host 處理器（CPU/PCH）與 TPM 晶片通訊時，遵循 TCG 定義的介面標準：

### 3.1 傳統 TIS (TPM Interface Specification)
* **起源**：TPM 1.2 時代沿用至 TPM 2.0，早期走 LPC（Low Pin Count），現代伺服器全面轉為 **SPI（Serial Peripheral Interface）** 或 **I2C**。
* **通訊模式**：暫存器映射模式（Register-based access）。
  * 透過一組精簡的暫存器：`TPM_ACCESS`, `TPM_STS`, `TPM_DATA_FIFO`, `TPM_HASH_DATA`。
  * **寫入指令**：Host 軟體一次只能透過 1-byte 或 4-byte 的 FIFO 暫存器（`TPM_DATA_FIFO`）將封包推入 TPM。
  * **延遲痛點**：FIFO 模式需頻繁觸發匯流排 Transaction，傳輸大封包（如 RSA-4096 金鑰或憑證）時效率低下。

### 3.2 PTP CRB (Platform TPM Profile: Command Response Buffer)
* **起源**：TCG 為提升效能與支援 fTPM/Memory-mapped 硬體而專門制定的現代架構。
* **通訊模式**：記憶體共用緩衝區（Shared Memory Buffer）。
  * 系統配置兩塊連續記憶體區塊：**Command Buffer** 與 **Response Buffer**。
  * **工作流程**：
    1. Host 直接將整包 TPM 指令以 DMA 或快速 Memory Copy 寫入 Command Buffer。
    2. 寫入控制暫存器 `TPM_CRB_CTRL_REQ` 發出 `START` 訊號。
    3. TPM 內部引擎非同步處理指令，完成後將結果寫回 Response Buffer，並更新狀態為 `COMPLETE`。
    4. Host 一次性讀取結果。
* **架構優勢**：大幅降低 CPU 開銷與暫存器輪詢次數，現代 x86/ARM64 伺服器之 UEFI 與 OS 均優先使用 CRB 介面。

---

## 4. 晶片內部核心組件與儲存架構真相

```
┌────────────────────────────────────────────────────────────────────────┐
│                        TPM 2.0 內部架構微觀觀                           │
│                                                                        │
│   ┌──────────────────────────┐      ┌──────────────────────────────┐   │
│   │   I/O Interface & Flow   │      │     Execution Engine         │   │
│   │   (TIS / CRB Interface)  │◀────▶│     (Secure Microcontroller) │   │
│   └─────────────┬────────────┘      └──────────────┬───────────────┘   │
│                 │                                  │                   │
│                 ▼                                  ▼                   │
│   ┌──────────────────────────┐      ┌──────────────────────────────┐   │
│   │  Cryptographic Engine    │      │  Non-Volatile Shielded Mem   │   │
│   │  • TRNG (Hardware Noise) │      │  • Endorsement Primary Seed  │   │
│   │  • RSA / ECC Core        │      │  • Platform Primary Seed     │   │
│   │  • SHA-256/384 / HMAC    │      │  • Storage Primary Seed      │   │
│   │  • AES-128/256 Engine    │      │  • Factory EK Certificate    │   │
│   └──────────────────────────┘      │  • NVRAM Indices (Defines)   │   │
│                                     └──────────────┬───────────────┘   │
│                                                    │                   │
│                                                    ▼                   │
│                                     ┌──────────────────────────────┐   │
│                                     │  Volatile Shielded Memory    │   │
│                                     │  • PCR Banks (SHA-256, etc.) │   │
│                                     │  • Active Session Contexts   │   │
│                                     │  • Loaded Key Slots (3~7)    │   │
│                                     └──────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

### 4.1 儲存架構真相：TPM 到底「存」不存金鑰？

#### 1. EK 與 SRK 平常不需要「存」在 NVRAM 裡
* TPM 內部永久保存的其實只有 **256-bit 的 Primary Seed**（例如 EPS, SPS）。
* 當開機呼叫 `TPM2_CreatePrimary` 時，TPM 是用硬體 KDF（金鑰衍生函數）**在揮發性 RAM 中動態重算出 EK 與 SRK**。
* 除非下達 `TPM2_EvictControl` 明確把它釘選（Pin）為「Persistent Handle」（如 `0x81000001` 或 `0x81010001`），它才會真正佔用 NVRAM 空間。

#### 2. 一般子金鑰（Child Keys）：存於 Host 硬碟，數量近乎無限！
這是 TPM 的**「樹狀金鑰包裹（Key Wrapping）」**核心模型：
* 呼叫 `TPM2_Create` 建立金鑰時，TPM 根本不會把它寫進晶片快閃記憶體。
* TPM 會把私鑰用父金鑰（如 SRK，使用 AES-CFB/AES-GCM）加密成 `private.blob`，將公鑰打包為 `public.blob`，**直接交給 Host OS 存放在硬碟檔案系統上**。
* **使用方式**：呼叫 `TPM2_Load` 把這兩個 blob 塞回 TPM 的揮發性 RAM，TPM 在晶片內部解密後賦予一個臨時 Handle（如 `0x80xxxxxx`），運算完畢即釋放。
* **結論**：使用者可以建立幾萬把子金鑰，因為所有金鑰都以密文狀態寄宿在主機硬碟上，不耗損 TPM NVRAM！

### 4.2 實體 NVRAM 容量數據 (By Chip Design)

專用安全晶片的 NVRAM 容量非常精簡，通常僅 **7 KB ～ 16 KB**：

| 晶片型號 | 廠商 | 總物理 Flash 空間 | 使用者可用 NV 空間 (`TPM2_PT_NV_COUNTERS_MAX`) | 典型可用容量 |
| :--- | :--- | :--- | :--- | :--- |
| **SLB 9670 / 9672** | Infineon (英飛凌) | 約 32 KB ~ 64 KB | **約 6,962 Bytes** | **~7 KB** |
| **NPCT75x 系列** | Nuvoton (新唐) | 約 32 KB ~ 64 KB | **約 8,192 ~ 12,288 Bytes** | **~8 ~ 12 KB** |
| **AT97SC / ATTPM20** | Microchip | 約 32 KB | **約 8,192 Bytes** | **~8 KB** |
| **fTPM (Intel PTT / AMD PSP)** | Intel / AMD | 劃分自 BIOS SPI Flash / ME 區塊 | 依韌體配置，通常模擬出 **16 KB ~ 64 KB** | **~16 KB** |

> **設計限制由來**：dTPM 採用主動金屬遮蔽、防 DPA 側信道攻擊的高安全性專用 EEPROM/Flash 製程，每 1 KB 矽晶圓成本高昂，且必須保證極致的耐用度與單元寫入原子性。

### 4.3 憑證與金鑰存放極限計算

1. **憑證（X.509 Certificates）存放極限**：
   * **原廠 EK Cert（出廠預先佔用）**：
     * RSA-2048 EK Cert (`0x01c00002`)：約 **1,000 ~ 1,500 Bytes**。
     * ECC P-256 EK Cert (`0x01c0000a`)：約 **500 ~ 800 Bytes**。
     * 出廠兩張憑證已吃掉約 **1.8 KB ~ 2.2 KB**。
   * **使用者自訂憑證**：
     * 扣除預載憑證後，以 7 KB NV 為例，淨空僅剩 **4 KB ~ 5 KB**。
     * 放 RSA-2048 憑證：最多再放 **2 ~ 3 張**。
     * 放 ECC P-256 憑證：最多再放 **4 ~ 6 張**。
     * 超限時 TPM 會直接回傳錯誤碼 `TPM_RC_NV_SPACE` (0x14A)。
2. **永久金鑰槽位（Persistent Keys, `0x81xxxxxx`）**：
   * 受限於晶片內部韌體常數與空間，整顆晶片通常只能釘選 **3 ～ 7 把金鑰**（預設通常留給 1 把 RSA EK, 1 把 ECC EK, 1 把 SRK，使用者通常只剩 2~4 個槽位）。
3. **執行期暫存金鑰槽位（Transient Keys, `0x80xxxxxx`）**：
   * 晶片內部 SRAM 極度狹小，同時只能載入 **3 ～ 7 把金鑰**。多行程競爭時依賴 Linux 核心 `/dev/tpmrm0` 進行自動 Context Swap-out。

---

## 5. TPM 2.0 階層架構與金鑰管理體系 (Hierarchies & Keys)

TPM 2.0 摒棄了單一金鑰根的缺陷，設計了**四大獨立的階層體系（Hierarchies）**：

```
                      TPM 2.0 四大階層架構
                               │
       ┌───────────────────────┼───────────────────────┬───────────────────────┐
       ▼                       ▼                       ▼                       ▼
【Platform Hierarchy】  【Storage Hierarchy】   【Endorsement Tier】      【Null Hierarchy】
   (平台階層 / 韌體)        (儲存階層 / Owner)       (背書階層 / 隱私)        (空階層 / 臨時)
       │                       │                       │                       │
 掌控者: BIOS/UEFI       掌控者: OS / 系統擁有者  掌控者: 隱私保護機構/CA    掌控者: 無 (臨時使用)
       │                       │                       │                       │
  Primary Seed:           Primary Seed:           Primary Seed:           隨每次開機隨機
      PPS                     SPS                     EPS                 重新生成種子
       │                       │                       │                       │
       ▼                       ▼                       ▼                       ▼
  Platform Root           Storage Root           Endorsement Key          臨時隔離金鑰
  Key (Platform Key)      Key (SRK)               (EK, RSA/ECC)          (斷電後永久失效)
                               │                       │
                               ▼                       ▼
                          衍生儲存子金鑰           Attestation Key
                        (Child Storage Keys)      (AK / AIK 證明金鑰)
```

### 5.1 三大主種子 (Primary Seeds) 的數學衍生模型

晶片在 NVRAM 儲存三組 256-bit 高熵的 **Primary Seeds（主種子）**：
* **EPS (Endorsement Primary Seed)**
* **SPS (Storage Primary Seed)**
* **PPS (Platform Primary Seed)**

呼叫 `TPM2_CreatePrimary` 時，TPM 使用 **KDF（Key Derivation Function）** 動態推導出 Primary Key：
$$\text{Primary Key} = \text{KDF}(\text{Primary Seed}, \text{Public Template Data})$$
* **特性**：只要 Template 與演算法參數相同，TPM 每次都能算出**完全相同的一對金鑰**，無需消耗 NVRAM 空間儲存私鑰。

### 5.2 核心金鑰角色定義
1. **EK (Endorsement Key，背書金鑰)**：
   * 由 EPS 衍生（或工廠直接寫入），用以代表該 TPM 晶片的硬體身分真偽。
   * **關鍵屬性**：**只能用於解密（Decrypt）與金鑰協商，絕對不支援簽章（Restricted Decrypt）**。這是為了防止惡意軟體用 EK 對任意內容簽章，導致該晶片被全網追蹤（隱私洩漏）。
2. **SRK (Storage Root Key，儲存根金鑰)**：
   * 由 SPS 衍生，是使用者/作業系統保護硬碟資料與子金鑰的根節點。如果執行 `TPM2_Clear`，SPS 會被隨機數覆寫更新，所有以舊 SRK 加密的資料將永久無法解密。
3. **AK / AIK (Attestation Key，證明金鑰)**：
   * 由 EPS 或 SPS 衍生的限制型簽章金鑰（Restricted Signing Key）。**專門用於對 PCR 狀態進行數位簽章（Quote）**，不允許使用者簽署任意自訂數據。

---

## 6. 平台配置暫存器 (PCR) 與度量數學原理

### 6.1 數學 Extend 操作公式
外部軟體**無法直接寫入覆蓋 PCR**。唯一的修改途徑是 **Extend（擴展）**：
$$\text{PCR}_{t+1} = \text{Hash}\Big(\text{PCR}_t \;\parallel\; \text{Data to Extend}\Big)$$
* 若度量標的物長度很大，系統先計算其摘要：$\text{Digest} = \text{Hash}(\text{Binary})$。
* 隨後送入 TPM 執行 Extend：
  $$\text{PCR}[i] = \text{SHA256}\Big(\text{PCR}[i]_{\text{current}} \;\parallel\; \text{Digest}\Big)$$

#### 核心密碼學保證
1. **抗順序顛倒（Order-sensitive）**：
   $\text{Hash}(A \parallel B) \neq \text{Hash}(B \parallel A)$。若開機過程中有任何模組載入順序調換，最終 PCR 必定完全改變。
2. **抗碰撞與不可逆（One-way Pre-image Resistance）**：
   攻擊者無法倒推送入特製數據「校正」回原本正常的 PCR 值。

### 6.2 硬體指令特性：Extend 支援範圍與跨 Bank
檢視 `TPM2_PCR_Extend` 底層指令介面：
```c
TPM_RC TPM2_PCR_Extend(
    TPMI_DH_PCR   pcrHandle,   // 只能傳入「單一」PCR 編號（如 0x00000004）
    TPML_DIGEST_VALUES digests // 要寫入的雜湊清單（可同時包含 SHA-256、SHA-384）
);
```
* **不能傳範圍（Range）**：無法以 `4-7` 批次寫入多個 PCR。若軟體需更新 4 到 7，必須發出 4 次獨立指令。
* **支援一次指令跨演算法 Bank 同步更新**：同一指令可同時將事件寫入 PCR 4 的 SHA-256 Bank 與 SHA-384 Bank。

### 6.3 開機時序真相：非順號累加，而是跟隨系統模組時間軸
* **晶片端視角**：TPM 晶片硬體完全不管 Extend 順序，先 Extend PCR 7 再 Extend PCR 0 完全合法。
* **韌體端視角**：順序純由 UEFI 的開機模組執行流程決定：
  1. `PCR 0`：CRTM/PEI 最早度量 BIOS 核心。
  2. `PCR 7`：DXE 早期即度量 Secure Boot 變數（**比 PCR 2、4 更早！**）。
  3. `PCR 2`：PCI 枚舉時度量 Option ROM（**單一 PCR 可被多次 Extend 滾雪球**，例如插 3 張卡即連續 Extend 3 次）。
  4. `PCR 4, 5`：BDS 階段度量開機引導檔與硬體路徑。
  5. `EV_SEPARATOR`：UEFI 交棒給 OS 前，透過迴圈對 PCR 0~7 廣播分隔常數（畫出交卷紅線）。

### 6.4 TCG PC Client 規範標準 PCR 分配表 (0 ~ 23)

| PCR 編號 | 定義階段名稱 | 負責 Extend 的軟體 | 實際被度量 (Hash) 的標的物 |
| :---: | :--- | :--- | :--- |
| **PCR 0** | **SRTM / Core Firmware** | CRTM / UEFI BIOS | **主機板核心韌體代碼**：BIOS ROM、CPU Microcode、OEM 專屬開機模組、韌體版本字串。 |
| **PCR 1** | **Host Platform Config** | UEFI BIOS | **平台硬體配置與設定**：SMBIOS 結構表、CMOS / BIOS Setup 的設定值（例如開啟/關閉虛擬化 VT-d）。 |
| **PCR 2** | **Option ROM Code** | UEFI DXE 階段 | **介面卡擴充韌體**：插在 PCIe 插槽上的 RAID 卡、網卡 PXE ROM、顯卡 GOP 韌體代碼。 |
| **PCR 3** | **Option ROM Config** | UEFI DXE 階段 | **介面卡組態**：Option ROM 執行時所依賴的設定資料與參數。 |
| **PCR 4** | **Boot Manager Code** | UEFI BDS 階段 | **引導程式本體與開機路徑**：GPT/MBR 開機引導區、`shimx64.efi` 或 `grubx64.efi` 執行檔本體、UEFI 開機變數（`BootOrder`, `Boot####`）。 |
| **PCR 5** | **Boot Manager Data** | UEFI BDS 階段 | **開機組態資料**：GPT 分割表結構、開機前讀取的所有非程式碼組態檔案。 |
| **PCR 6** | **Resume Events** | UEFI 韌體 | **電源狀態轉換**：S3 休眠喚醒代碼與狀態儲存資料。 |
| **PCR 7** | **Secure Boot Policy** | UEFI 韌體 | **安全開機策略與憑證庫**：Secure Boot 開關狀態、`PK`、`KEK`、許可資料庫 `db`、黑名單 `dbx` 的所有憑證內容。 |
| **PCR 8** | **OS Bootloader Args** | GRUB / systemd-boot | **開機載入器參數與模組**：GRUB 載入的模組、Kernel Command Line（如 `console=ttyS0 quiet`）。 |
| **PCR 9** | **OS Kernel & Initrd** | GRUB / systemd-boot | **作業系統核心**：Linux `vmlinuz` 映像檔、`initramfs` (initrd) 壓縮包。 |
| **PCR 10** | **IMA (運行期度量)** | Linux 核心 (IMA 模組) | **系統運作期應用程式**：Linux 每次執行 `execve()` 或載入動態函式庫時度量的 Hash。 |
| **PCR 11** | **Unified Kernel Images** | systemd / OS | 存放 UKI（將 Kernel+Initrd+Cmdline 打包成單一 EFI）或磁碟解密策略。 |
| **PCR 12** | **Kernel Extension** | OS / systemd | 系統擴充模組、Credentials 憑證資料。 |
| **PCR 13** | **System Extensions** | OS | OS Sysext 映像檔度量。 |
| **PCR 14** | **MOK (Machine Owner Key)** | Linux `shim.efi` | **Linux 私有簽章憑證**：`MokList`、`MokListX`（載入第三方自編譯驅動憑證）。 |
| **PCR 15** | **OS Specific** | OS / Diag | 保留給客製化軟體或工廠診斷程式自定義度量。 |
| **PCR 16** | **Debug PCR** | 任何人 | **唯一支援隨時 Reset 回 0 的暫存器**，供開發除錯或短暫授權 Session 使用。 |
| **PCR 17~22**| **DRTM (動態信任根)** | Intel TXT / AMD SINIT | **硬體隔離動態度量**（透過 CPU 特殊指令 `GETSEC[SENTER]` 或 `SKINIT` 觸發，不依賴 BIOS）。 |

---

## 7. 信任鏈與度量引導 (Measured Boot & TCG Event Log)

### 7.1 信任鏈傳遞時序與「先度量、後執行」原則

```
[ Power On ]
     │
     ▼
┌─────────────────────────┐
│ Core Root of Trust for  │ ◀── 燒死在 SPI Flash Boot Block (不可竄改代碼)
│ Measurement (CRTM)      │
└────────────┬────────────┘
             │ 1. 測量 UEFI PEI Core，寫入 Event Log 並 Extend PCR 0
             ▼
┌─────────────────────────┐
│ UEFI PEI / DXE Phase    │ ◀── 系統韌體執行硬體初始化
└────────────┬────────────┘
             │ 2. 測量 Option ROM、Secure Boot Keys，Extend PCR 2, 7
             ▼
┌─────────────────────────┐
│ UEFI Boot Manager       │ ◀── 測量開機選項路徑，Extend PCR 4
└────────────┬────────────┘
             │ 3. 測量 shim.efi / grubx64.efi，Extend PCR 4, 8
             ▼
┌─────────────────────────┐
│ OS Bootloader (GRUB)    │
└────────────┬────────────┘
             │ 4. 測量 vmlinuz 核心與 initramfs，Extend PCR 9
             ▼
┌─────────────────────────┐
│ OS Kernel (Linux)       │ ◀── 正式接管系統，啟動 systemd 與 IMA (PCR 10)
└─────────────────────────┘
```

#### 防範惡意 EFI 檔案的鐵律：Measure BEFORE Execute
若攻擊者在 ESP 分割區建立一個自訂的 `.efi` 檔案：
1. **UEFI 在執行該檔案的第一行程式碼之前**，就必須先讀取整份二進位內容，計算摘要並 **Extend PCR 4**。
2. 即使該 `.efi` 檔案嘗試自行 Extend 其他值，**PCR 4 早已被標記改變**，數學上無法倒轉。
3. 若開啟 Secure Boot，未經簽署的 `.efi` 直接被 UEFI 拒絕執行（Access Denied）。

### 7.2 TCG Event Log 的關鍵角色與回放驗證 (Replay Verification)
* **問題**：PCR 暫存器只裝得下最終的 32-byte 雜湊值。若 PCR 算出的值與預期不符，無法由 PCR 倒推何處出錯。
* **解法**：UEFI 在主機記憶體維護一份結構化的二進位歷史記錄——**TCG Event Log**（Linux 下位於 `/sys/kernel/security/tpm0/binary_bios_measurements`）。
* **回放驗證**：驗證者將 Event Log 條目在本地逐筆執行數學 Extend 模擬重算，若計算結果與晶片物理 PCR 一模一樣，即證明該 Event Log 真實可信，可逐項審計每個開機模組。

---

## 8. 放行機制判定：本地封印 vs 遠端證明

**TPM 晶片本身完全不具備「善惡判斷」能力，它是不帶感情的數學協同處理器。** 系統放行機制依場景劃分為：

### 8.1 本地放行（Local Sealing，如硬碟自動解密）
1. **封印階段（Enrollment）**：
   * 系統讀取當前健康狀態的 PCR 值（如 PCR 7 為 `0xABC...`）。
   * 建立金鑰時，TPM 計算該策略的 `PolicyDigest`，並將其**永久烙印在金鑰物件的元數據（Metadata）**中。
2. **開機解鎖階段（Unsealing）**：
   * 開機程式向 TPM 請求解開硬碟金鑰。
   * **TPM 硬體內部核對**：讀取晶片內當前真實的物理 PCR 7，即時計算 Session Digest。
   * 比對吻合 ➡️ 釋出金鑰；比對不符 ➡️ 拒絕解鎖。

```
[ Host Security Daemon ]               [ TPM 2.0 Chip ]
          │                                   │
          │ 1. TPM2_StartAuthSession(Policy)  │
          │──────────────────────────────────▶│ (建立臨時 Policy Session)
          │                                   │
          │ 2. TPM2_PolicyPCR(PCR 7)          │
          │──────────────────────────────────▶│ 讀取物理 PCR 7
          │                                   │ PolicyDigest = Hash(PolicyDigest || PCR_7)
          │                                   │
          │ 3. TPM2_Unseal (帶入 Session ID)  │
          │──────────────────────────────────▶│ 比對 Object AuthPolicy 與 Session Digest
          │                                   │
          │    4. 釋出解密後的原始金鑰數據     │
          │◀──────────────────────────────────│ (若符合則釋出，不符回傳 TPM_RC_POLICY_FAIL)
```

### 8.2 本地放行面對 BIOS / OS 升級的工程抉擇

| 封印策略綁定對象 | 安全性等級 | 面對 BIOS / OS 升級的後果 | 業界實務採納度 |
| :--- | :--- | :--- | :--- |
| **嚴苛模式：綁定 PCR 0, 1, 4, 8, 9** | 最高（任何韌體、設定或核心變動即攔截） | **維護災難**：正常更新 BIOS 或 `apt upgrade` 核心後，開機立即鎖死硬碟，需手動輸入救援金鑰 | 極少單獨使用於大規模機房 |
| **標準模式：僅綁定 PCR 7** | 極高（驗證簽章信任鏈） | **無痛升級**：只要升級的 BIOS 與 Kernel 是 OEM / 官方簽署認可的，PCR 7 紋風不動，正常放行；私造未簽章核心則會觸發攔截 | **現代 BitLocker 與 Linux systemd-cryptenroll 預設黃金標準** |

#### 正常升級的產業界解決方案：
1. **升級前暫停與重新封印（Suspend & Reseal）**：Windows Update 在刷 BIOS 前暫停 BitLocker，開機採集新版 PCR 後重新封印。
2. **動態授權簽章（`TPM2_PolicyAuthorize`）**：封印時指定「只要 PCR 符合資安單位私鑰簽名的授權票據（Ticket）即可放行」，升級新 Kernel 時由 CI/CD pipeline 簽發授權，主機開機無痛解鎖。

---

## 9. 遠端證明架構 (Remote Attestation & Quote)

遠端伺服器在無實體接觸的情況下，透過「黃金基準資料庫（Golden DB）」審核目標主機：

```
[ Attestation Verifier (遠端驗證伺服器) ]        [ Target Server (待測主機 / TPM) ]
                    │                                            │
                    │ 1. 發送 Challenge Request (帶入 Nonce 隨機數)│
                    │───────────────────────────────────────────▶│
                    │                                            │
                    │                                            │ 2. 呼叫 TPM2_Quote(AK_Handle, PCR_Select, Nonce)
                    │                                            │    • TPM 讀取指定 PCR
                    │                                            │    • 封裝結構 TPMS_ATTEST:
                    │                                            │      { ExtraData: Nonce, PCRSelect, PCRDigest }
                    │                                            │    • 使用 AK 私鑰對 TPMS_ATTEST 進行數位簽章
                    │                                            │
                    │ 3. 回傳 Quote 數據 + 簽章 + TCG Event Log   │
                    │◀───────────────────────────────────────────│
                    │                                            │
                    │ 4. 驗證 Quote 簽章有效性 (使用已知的 AK 公鑰) │
                    │ 5. 比對 Quote 中的 Nonce 是否等於步驟 1 的隨機數│
                    │ 6. 依據 Event Log 重算雜湊，核對 PCRDigest │
                    │ 7. 與資料庫內之「黃金基準值 (Golden Values)」比對│
                    ▼
          [ 判定節點可信度並准許上線 ]
```

* **防止重放攻擊（Anti-Replay）**：透過 Verifier 給予的 Nonce，攻擊者無法拿先前截獲的正常 Quote 數據矇混過關。
* **隱私保護（Privacy CA 機制）**：驗證伺服器不直接接觸具唯一識別性的 EK，而是驗證由 EK 背書授權的 AK，達到跨組織證明且無法被反向關聯硬體序號。

---

## 10. Linux 驅動層與 TSS 2.0 軟體堆疊

```
┌────────────────────────────────────────────────────────┐
│  Applications (tpm2-tools, systemd-cryptenroll, Clevis)│
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────┐
│  TSS 2.0 (TCG Software Stack)                          │
│  ├── FAPI (Feature API)    - 高階簡化操作 (自動管理 Key)│
│  ├── ESAPI (Enhanced SAPI) - 密碼學安全連線 (支援加密)  │
│  ├── SAPI (System API)     - 1:1 對映底層 TPM 指令封包 │
│  └── TCTI (TPM Command Transmission Interface)         │
│       • libtss2-tcti-device (直接通訊)                  │
│       • libtss2-tcti-tabrmd (用戶態資源管理 daemon)     │
└───────────────────────────┬────────────────────────────┘
                            │ (ioctl / read / write)
┌───────────────────────────▼────────────────────────────┐
│  Linux Kernel (/dev/tpm0 與 /dev/tpmrm0)                │
│  ├── tpm-chip core (通用 TPM 抽象層)                   │
│  ├── In-kernel Resource Manager (/dev/tpmrm0)          │
│  └── Bus Driver (tpm_tis, tpm_crb, tpm_i2c_infineon)   │
└───────────────────────────┬────────────────────────────┘
                            │ (SPI / I2C / MMIO Bus)
┌───────────────────────────▼────────────────────────────┐
│  Physical Hardware TPM 2.0 Controller                  │
└────────────────────────────────────────────────────────┘
```

### 10.1 `/dev/tpm0` 與 `/dev/tpmrm0` 的重大差異
* **`/dev/tpm0`（原始硬體節點）**：
  * 直接連通硬體。**僅限單一行程開啟（Exclusive access）**，無資源置換功能。若多個進程同時發指令會發生衝突鎖死。
* **`/dev/tpmrm0`（核心內建資源管理器節點，Linux 4.12+ 引進）**：
  * **所有使用者態應用程式的標準存取點**。
  * 核心自動處理多行程併發競爭，並在 TPM 內部 Slot 滿載時，主動將金鑰透過 Context Save 指令換出至系統記憶體，確保系統多工運作。

---

## 11. 製造測試與量產維運實務 (Manufacturing, Diag & Provisioning)

### 11.1 產線初測必測項目 (Station Checklist)
1. **晶片身分與硬體辨識**：
   * 下發 `TPM2_GetCapability` 確認硬體 Vendor ID（如 0x15D1 Infineon, 0x1050 Nuvoton）、韌體主要與次要版本。
   * 讀取晶片狀態確認未處於 Lockout Mode（防暴力破解鎖定模式）。
2. **自我測試（Self-Test）與健康狀態**：
   * 呼叫 `TPM2_SelfTest(fullTest=YES)` 強制觸發內部演算法引擎全檢。
   * 若返回 `TPM_RC_FAILURE`，晶片內部已進入安全防禦鎖定，該主機板在產線上必須直接打不良標籤。
3. **EK Certificate 讀取與驗證**：
   * 從標準 NV Index（RSA EK 憑證位於 `0x01c00002`，ECC EK 位於 `0x01c0000a`）讀取原廠 X.509 憑證。
   * 透過晶片原廠公鑰鏈（CA Chain）在測試機離線比對憑證有效性，防範產線被調包或焊上仿冒山寨晶片。
4. **隨機數與密碼測試**：
   * 呼叫 `TPM2_GetRandom` 提取 64-byte 隨機數，確認 TRNG 無故障且無連續全 0 或全 1 的熵缺失（Entropy Failure）。

### 11.2 PPI (Physical Presence Interface) 實體在場機制
* **定義**：TCG 為防範惡意軟體私自清除 TPM 而設計的規範。
* **運作模式**：執行清除 TPM（Clear）或修改平台配置等高風險指令時，系統必須向 BIOS 發出 PPI 請求。BIOS 在下次開機時會阻斷開機流程，在螢幕上跳出文字提示：「*Press F12 to confirm clearing TPM*」，必須有現場操作員實體按下按鍵，BIOS 才會執行清空指令。
* **工廠自動化旁路（MFG Bypass）**：
  在自動化量產測試線上，不可能由作業員手動按 F12。BIOS 必須支援工廠除錯模式或專屬工廠選項（Factory Mode / PPI Bypass），允許經過簽章的 Diag 程式藉由寫入特定的 UEFI NVRAM Variable 或透過 IPMI OEM 指令，在無需人工介入下直接重設 TPM。

---

## 12. 工程實務陷阱與除錯心法 (Brutal Truths & Pitfalls)

### 12.1 PCR 飄移（PCR Drift）引發硬碟鎖死
* **現象**：明明沒中木馬，系統重開機後 BitLocker 或 LUKS 卻要求輸入 48 位元緊急救援金鑰（Recovery Key）。
* **根因分析**：
  * **UEFI 更新或設定變動**：更新 BIOS 會改動 PCR 0；若把 BIOS 裡的開機順序微調、插上外接隨身碟開機，會改變 PCR 4；更新主機板驅動甚至開啟某些 PCIe 裝置的 Option ROM 會撼動 PCR 2。
  * **最佳防禦設計**：
    在量產環境中，**Sealing 策略切勿盲目綁定所有 PCR**。通常只需綁定 **PCR 7（Secure Boot 狀態與資料庫）**，必要時加綁 PCR 0 或 11。絕對不要隨意將易變動的 PCR 4 或 PCR 5 納入金鑰封印策略。

### 12.2 SPI 匯流排掛死（Bus Hang）與低壓訊號完整性
* **現象**：主機板冷開機時偶發抓不到 TPM，或 dmesg 出現 `tpm_tis: 1.2 TPM returned invalid id to vendor query`。
* **硬體排查重點**：
  * **SPI CS# 與 Clock 時序**：TPM 的 SPI 頻率通常在 33MHz ~ 48MHz 之間，檢查線路走線阻抗與信號反射。
  * **Reset 訊號時序**：TPM 晶片的硬體 Reset 腳位必須與 PCH/Platform Reset 精確同步，若 PCH 先釋放 Reset 而 TPM 還在內部初始化，開機 BIOS 偵測階段會直接判定 TPM 不存在並跳過驅動載入。

### 12.3 NVRAM 空間耗盡與 Lockout 計數器陷阱
* **Dictionary Attack Lockout（防字典暴力破解）**：
  * 當程式嘗試使用錯誤的授權密碼存取 TPM 物件時，晶片內部的 FailedTries 計數器會累加。
  * 達到臨界閾值（如 3 次）時，TPM 進入 Lockout 狀態，拒絕執行授權指令。
  * **排查指令**：
    使用 `tpm2_getcap properties-variable` 檢視 `TPM2_PT_LOCKOUT_COUNTER`。在工廠重置時，需藉由 `tpm2_dictionarylockout --clear-lockout` 解鎖。
