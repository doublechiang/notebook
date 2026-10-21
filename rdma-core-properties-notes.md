# RDMA 三項核心特性筆記 (Zero-copy / Kernel Bypass / Hardware Transport)

> 本文件整理 RDMA 常被一起提到的三項特性，以及它們和 RDMA「定義」之間的關係。
> 建議在 VS Code 中搭配 **Markdown Preview (`Ctrl + Shift + V`)** 或左側 **Outline（大綱目錄）** 快速導航閱讀。
> 註：為確保 VS Code 大綱樹穩定解析，所有標題列一律使用純文字，不嵌入反引號（inline code）。
> 註：文中延遲、頻寬數字皆為數量級經驗值，非實測；需要數字時請在實機以 ib_write_lat / ib_write_bw / iperf3 量測。

---

## 目錄 (Table of Contents)
1. [先釐清：RDMA 的定義只有一條](#1-先釐清rdma-的定義只有一條)
2. [特性一：Zero-copy（零複製）](#2-特性一zero-copy零複製)
3. [特性二：Kernel Bypass（繞過核心）](#3-特性二kernel-bypass繞過核心)
4. [特性三：Hardware Transport（協定在網卡硬體內完成）](#4-特性三hardware-transport協定在網卡硬體內完成)
5. [總整理：三項特性與定義的關係](#5-總整理三項特性與定義的關係)
6. [GPUDirect RDMA：資料不經 CPU，控制仍經 CPU](#6-gpudirect-rdma資料不經-cpu控制仍經-cpu)
7. [RDMA 網卡 vs. DPU：卸載的是傳輸，還是基礎設施](#7-rdma-網卡-vs-dpu卸載的是傳輸還是基礎設施)

---

## 1. 先釐清：RDMA 的定義只有一條

**RDMA (Remote Direct Memory Access)**：遠端主機的網卡直接讀寫本機記憶體，**本機 CPU 與 OS 不參與該次資料傳輸**。

* 最典型的形式是 **one-sided** 操作（`RDMA READ` / `RDMA WRITE`）：對端 CPU 完全不知道資料被讀寫了。
* 另有 two-sided 的 `SEND` / `RECV`，接收端需預先 post 一個 receive buffer，但資料搬移本身仍由網卡完成。

下面三項是「RDMA 為什麼快」的來源，**不是**「什麼才算 RDMA」的判準。三項中只有 Zero-copy 是定義本身就帶著的。

---

## 2. 特性一：Zero-copy（零複製）

### 2.1 是什麼
網卡直接 DMA 進出應用程式的 buffer，資料不在 kernel buffer 與 user buffer 之間複製。

### 2.2 對照傳統 TCP 路徑
```
[ 傳統 TCP 接收 ]
 NIC ──DMA──▶ kernel socket buffer ──CPU copy──▶ user buffer

[ RDMA 接收 ]
 NIC ──DMA──▶ user buffer（事先註冊的 Memory Region）
```

### 2.3 前提：Memory Registration
* 應用程式須先 `ibv_reg_mr()` 註冊記憶體：pin 住實體頁面，並把虛擬→實體位址對映交給網卡。
* 註冊會產生 `lkey` / `rkey`；對端憑 `rkey` + 虛擬位址才能存取這塊記憶體。
* 註冊成本高（pin page、更新網卡轉譯表），因此實務上會預先註冊、重複使用 buffer。

### 2.4 與定義的關係
**定義本身就帶著。** 「網卡直接存取目標記憶體」意味著中間沒有暫存複製，否則就不是 Direct Memory Access。

### 2.5 延伸：GPUDirect RDMA
Zero-copy 推到極致：註冊的記憶體是 **GPU HBM**，網卡經 PCIe peer-to-peer 直接 DMA 進 GPU，host DRAM 與 CPU 完全不在資料路徑上。這是 AI 叢集非用 RDMA 不可的主因。

---

## 3. 特性二：Kernel Bypass（繞過核心）

### 3.1 是什麼
應用程式在 user space 直接操作網卡的 Queue Pair (QP) 與 Completion Queue (CQ)，**資料路徑 (data path)** 上沒有系統呼叫、沒有 TCP/IP 協定棧、沒有中斷（可用 polling）。

### 3.2 控制路徑仍經過 kernel
| 路徑 | 操作 | 是否經過 kernel |
| :--- | :--- | :---: |
| 控制路徑 (control path) | 開裝置、建立 PD / QP / CQ、註冊 MR | **是** |
| 資料路徑 (data path) | post send / post recv、poll CQ、敲 doorbell | **否** |

Kernel 只負責把網卡的 doorbell 頁面與 queue 記憶體 mmap 給行程，之後每筆 I/O 都由 user space 直接寫入網卡。

### 3.3 與定義的關係
**不是必要條件。** 以下都在 kernel 內使用 RDMA，沒有 bypass kernel，但對端 CPU 一樣不參與資料傳輸，仍然是 RDMA：
* NVMe-oF RDMA（kernel initiator / target）
* iSER（iSCSI Extensions for RDMA）
* SMB Direct
* NFS over RDMA

這些場景中 RDMA 省掉的是複製與協定處理，而不是系統呼叫（因為呼叫端本來就在 kernel 內）。

---

## 4. 特性三：Hardware Transport（協定在網卡硬體內完成）

### 4.1 是什麼
切封包、重組、ACK、重傳、順序保證等 transport 工作，全部由網卡硬體完成，不佔 host CPU。

### 4.2 三種主流實作
| 實作 | 底層網路 | Transport 位置 | 備註 |
| :--- | :--- | :--- | :--- |
| **InfiniBand** | IB 專用網路 | 網卡硬體 | 原生 lossless，credit-based flow control |
| **RoCE v2** | Ethernet (UDP/IP) | 網卡硬體 | 傳統上依賴 PFC 造 lossless 網路；Spectrum-X 等平台另加 adaptive routing 與壅塞控制 |
| **iWARP** | Ethernet (TCP/IP) | 網卡硬體 (TOE) | 在網卡上跑 TCP，不需 lossless 網路 |

### 4.3 與定義的關係
**不是必要條件。** 軟體實作依然是 RDMA：
* **Soft-RoCE（`rxe`）**：在普通乙太網卡上用 kernel 軟體模擬 RoCE v2。
* **soft-iWARP（`siw`）**：用 kernel 軟體模擬 iWARP。

這兩者對應用程式呈現完全相同的 verbs 語意，程式不用改就能跑；但 transport 由 CPU 處理，**省 CPU 與低延遲的好處大多消失**。

### 4.4 實務用途
* 適合：在沒有 RDMA 網卡的機器上開發、CI 測試、驗證 verbs 程式邏輯。
* 不適合：效能量測。用 Soft-RoCE 量到的數字不能代表硬體 RDMA。

---

## 5. 總整理：三項特性與定義的關係

| 特性 | 與 RDMA 定義的關係 | 主要效益 | 反例（仍算 RDMA） |
| :--- | :--- | :--- | :--- |
| **Zero-copy** | 定義本身就帶著 | 省記憶體頻寬、省 CPU 複製、可直達 GPU HBM | 幾乎沒有 |
| **Kernel Bypass** | 非必要，是 user-space verbs 的特性 | 去掉系統呼叫與中斷的固定成本，降低延遲與尾延遲 | NVMe-oF kernel target、iSER、SMB Direct、NFS over RDMA |
| **Hardware Transport** | 非必要，是效能來源 | 協定處理不佔 CPU，高線速下 CPU 才跟得上 | Soft-RoCE (`rxe`)、soft-iWARP (`siw`) |

**一句話**：RDMA 的**定義**是「直接存取遠端記憶體、對端 CPU 不介入」；它的**效能優勢**來自 Hardware Transport + Kernel Bypass，拿掉其中之一，語意還在，優勢打折。

---

## 6. GPUDirect RDMA：資料不經 CPU，控制仍經 CPU

> 不需要 SmartNIC，一般 RDMA 網卡（ConnectX）即可。以下 API 名稱憑記憶整理，未對照官方文件。

### 6.1 Setup（一次性，CPU + kernel）
1. `cudaMalloc` 在 HBM 配 buffer。
2. `ibv_reg_mr` 註冊該 GPU 位址 → `nvidia-peermem`（或 dma-buf）向 NVIDIA driver 取頁，經 **GPU BAR1** 映射成 PCIe 位址並 pin 住，寫入網卡轉譯表，得到 `lkey` / `rkey`。
3. 建 QP / CQ，out-of-band 與對端交換 QP 資訊與遠端 buffer 的位址、`rkey`。

GPU 本身不知道被註冊；它只是落在 BAR1 範圍內的被動記憶體。

### 6.2 傳統 GDR 資料流程（A 機 GPU → B 機 GPU，RDMA WRITE）
```
① CPU 等 GPU kernel 算完（stream sync / event）   ← CPU
② CPU 在 host memory 的 SQ 寫 WQE                  ← CPU
③ CPU 敲 doorbell（MMIO 寫網卡 UAR）               ← CPU：「控制通知網卡」就是這一下
④ 網卡 DMA 讀回 WQE
⑤ 網卡經 PCIe P2P 從 GPU BAR1 讀資料               ← 不經 host DRAM / CPU
⑥ 上網路 → ⑦ B 機網卡直接 DMA 寫進 B 的 HBM（B 的 CPU 不知情）
⑧ 網卡寫 CQE 到 host memory → ⑨ CPU poll CQ        ← CPU
```
* **B 機 GPU 如何得知資料到了**：`WRITE_WITH_IMM` 讓 B 的 CPU 收到 CQE 再 launch kernel；或 GPU poll 一個 flag（需 flush / fence，NIC 經 BAR1 的寫入對 GPU 的可見順序不保證）。

### 6.3 GPU-initiated（IBGDA）
| 項目 | 傳統 GDR | IBGDA |
| :--- | :--- | :--- |
| WQE / CQ 位置 | host memory | GPU memory |
| 寫 WQE、敲 doorbell、poll CQ | CPU | GPU thread（UAR 映射進 GPU 位址空間） |
| CPU 角色 | 控制 + 同步 | 只剩 setup |

使用者：NVSHMEM IBGDA transport、DeepEP。NCCL 傳統上走 CPU proxy thread。

### 6.4 產線測試陷阱：PCIe 拓撲
* NIC 與 GPU 最好在同一顆 PCIe switch 下；`nvidia-smi topo -m`：PIX / PXB 好，PHB 普通，SYS 差。
* **ACS** 開著會把 P2P 強制繞上 root complex → 變慢或失敗；需關 ACS 或開 ATS。
* BAR1 大小限制可註冊的 GPU 記憶體（BIOS：Above 4G decoding / Resizable BAR）。
* 驗證：`ib_write_bw --use_cuda=<gpu>`。**設錯多半不報錯、只是變慢**，測項要判頻寬數字，不是只看 pass/fail。

---

## 7. RDMA 網卡 vs. DPU：卸載的是傳輸，還是基礎設施

> 以下場景與 Vera Rubin 配置憑記憶整理，未查證。

### 7.1 核心區分
| | RDMA 網卡（ConnectX / SuperNIC） | DPU（BlueField） |
| :--- | :--- | :--- |
| **卸載什麼** | **傳輸**：資料怎麼搬（transport、zero-copy、GDR） | **基礎設施軟體**：原本跑在 host CPU 的網路虛擬化、儲存、安全服務 |
| **有沒有自己的 CPU / OS** | 沒有 | 有（ARM 核心，跑自己的 Linux） |
| **典型位置** | GPU 後端 east-west fabric（all-reduce） | 前端 north-south 網路、儲存櫃 |

**判斷法**：要的只是頻寬和 RDMA → ConnectX；要把「服務」從 host 拿走 → BlueField。在 GPUDirect RDMA 流程（第 6 節）中，BF 的 CPU 完全沒用到。

### 7.2 BF-4 的使用場景
1. **多租戶前端網路**：OVS / VXLAN / 防火牆跑在 DPU；bare-metal 租戶拿 host root，也碰不到 DPU 上的業者管控軟體（信任邊界）。
2. **儲存虛擬化（SNAP）**：對 host 模擬本機 NVMe，後端走 NVMe-oF 到遠端。
3. **儲存控制器**：放在儲存櫃取代 x86 儲存伺服器，例如 NVIDIA CMX。
4. **安全與遙測**：host 被攻破時，DPU 上的監控不會跟著淪陷。

### 7.3 產線測試的差異
* **ConnectX**：link、頻寬、GDR（見 6.4）。
* **BlueField**：當成一台小電腦來測：ARM 開機、BFB / DOCA image 燒錄、host 和 DPU 之間的通道、BMC 怎麼管理它。
* Rubin 機櫃中兩者是否並存（compute tray 上 ConnectX-9 做 scale-out、BF-4 做前端和儲存），要以客戶 BOM 或 NVIDIA 參考設計為準。
