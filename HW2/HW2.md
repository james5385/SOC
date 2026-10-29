## 題目一 ANS

### 參考文件

**UG585**
Zynq-7000 Technical Reference Manual

---

### 1. Board Support Packages (BSP)
* **頁數**：第 26 頁
* **所在章節**：Chapter 1: Introduction 的 1.1 節 (Overview)
* **內容與解釋**：
  * 針對 Zynq-7000 系列，Xilinx 為 PS（處理系統）與 PL（可程式化邏輯）中的周邊設備提供了 Stand-alone（裸機）與 Linux 的裝置驅動程式。
  * 額外的作業系統 (OS) 以及板級支援套件 (BSPs) 則是由 Xilinx 的合作夥伴提供。
  * 這表示 UG585 手冊主要聚焦於硬體架構，若需要 BSP 的軟體支援與套件，通常來自第三方合作夥伴的資源。

### 2. Interrupt Service Routine (ISR)
在 UG585 檔案的目錄中，指示了硬體開發者在編寫特定周邊驅動程式時，可以前往以下頁面查看中斷服務常式 (ISR) 的程式設計指南：

#### DMA 控制器的 ISR
* **頁數**：第 261 頁
* **所在章節**：Chapter 9: DMA Controller 內的「9.3 Programming Guide for DMA Controller」底下，子章節為「9.3.3 Interrupt Service Routine」。
* **內容與解釋**：
  * 這是針對 DMA（直接記憶體存取）控制器的程式設計指南。
  * 該章節會指導開發者如何在軟體中設定與處理 DMA 傳輸完成或發生錯誤時所觸發的中斷。
## 題目二 ANS
### 1. 中斷相關 Port 訊號（PS ↔ PL 介面）

在 PS（Processing System）與 PL（Programmable Logic）之間，主要包含以下中斷連接埠訊號[1][2]：

* **PL 傳送至 PS 的中斷訊號（PL-to-PS Interrupts）**：
  * **IRQF2P[15:0]**（16 支輸入訊號）：映射至 GIC 的共享外設中斷（SPI, Shared Peripheral Interrupts），對應 IRQ ID #61～68 及 #84～91[2]。
  * **IRQF2P[19:16]**（4 支輸入訊號）：反相後直接連接至 CPU 的私有外設中斷（PPI, Private Peripheral Interrupts）單元[1]。
    * IRQF2P[7]：CPU0 nIRQ（IRQ ID #31）[6]
    * IRQF2P[8]：CPU1 nIRQ（IRQ ID #31）[6]
    * IRQF2P[9]：CPU0 nFIQ（IRQ ID #28）[6]
    * IRQF2P[10]：CPU1 nFIQ（IRQ ID #28）[6]

* **PS 傳送至 PL 的中斷訊號（PS-to-PL Interrupts）**：
  * **IRQP2F[27:0]**（28 支輸出訊號）：將 PS 內部周邊（例如 GPIO、USB、Ethernet、SDIO、I2C、SPI、UART、CAN、DMAC 等）發出的中斷訊號轉發至 PL[2]。
<img width="1427" height="388" alt="image" src="https://github.com/user-attachments/assets/648ea1de-b295-441b-a03f-14ceacb434c9" />

---

### 2. 中斷控制器（GIC）位址與相關暫存器

Zynq-7000 的通用中斷控制器（GIC, Generic Interrupt Controller）基於 ARM GIC 虛擬化架構，透過 CPU 私有匯流排存取[12][13]。其暫存器記憶體映圖如下[14]：

#### (1) GIC CPU 介面暫存器（ICC, CPU Interface Registers）
* **位址範圍**：`0xF8F0_0100` ～ `0xF8F0_01FF`[14]
* **關鍵暫存器**[15]：
  * **ICCICR** (`0xF8F0_0100`)：CPU 介面控制暫存器（CPU Interface Control）
  * **ICCPMR** (`0xF8F0_0104`)：中斷優先級遮罩暫存器（Interrupt Priority Mask）
  * **ICCBPR** (`0xF8F0_0108`)：二進位點暫存器（Binary Point）
  * **ICCIAR** (`0xF8F0_010C`)：中斷應答/確認暫存器（Interrupt Acknowledge）
  * **ICCEOIR** (`0xF8F0_0110`)：中斷結束暫存器（End of Interrupt）
  * **ICCHPIR** (`0xF8F0_0118`)：最高優先級懸掛中斷暫存器（Highest Pending Interrupt）
<img width="1422" height="148" alt="image" src="https://github.com/user-attachments/assets/276532fb-62b5-4f47-a729-4ad62dc85961" />
<img width="1414" height="523" alt="image" src="https://github.com/user-attachments/assets/b62d6206-22cd-4197-a26f-57f6edb3508e" />

#### (2) GIC 分發器暫存器（ICD, Interrupt Distributor Registers）
* **位址範圍**：`0xF8F0_1000` ～ `0xF8F0_1FFF`[14]
* **關鍵暫存器**[16]：
  * **ICDDCR** (`0xF8F0_1000`)：分發器控制暫存器（Distributor Control）
  * **ICDICTR** (`0xF8F0_1004`)：中斷控制器類型暫存器（Controller Type）
  * **ICDISER[2:0] / ICDICER[2:0]** (`0xF8F0_1100` / `0xF8F0_1180`)：中斷啟用 / 清除啟用設定暫存器（Interrupt Set-Enable / Clear-Enable）
  * **ICDISPR[2:0] / ICDICPR[2:0]** (`0xF8F0_1200` / `0xF8F0_1280`)：中斷懸掛設定 / 清除懸掛暫存器（Interrupt Set-Pending / Clear-Pending）
  * **ICDIPR[23:0]** (`0xF8F0_1400`)：中斷優先級設定暫存器（Interrupt Priority）
  * **ICDIPTR[23:0]** (`0xF8F0_1800`)：中斷目標 CPU 設定暫存器（Interrupt Processor Targets）
  * **ICDICFR[5:0]** (`0xF8F0_1C00`)：中斷敏感度與觸發型態設定暫存器（Interrupt Configuration, 如觸發方式為上升沿觸發或高電位觸發）[6][19]
  * **ICDSGIR** (`0xF8F0_1F00`)：軟體觸發中斷暫存器（Software-Generated Interrupt）
<img width="941" height="157" alt="image" src="https://github.com/user-attachments/assets/9e0485cc-fb92-428e-a498-904d77a5dd82" />
<img width="944" height="797" alt="image" src="https://github.com/user-attachments/assets/0f443430-d1ef-4847-ad18-10ca0c59166c" />
<img width="944" height="372" alt="image" src="https://github.com/user-attachments/assets/5d4b563e-18ba-419c-a7b1-5365f8a670f6" />
<img width="946" height="725" alt="image" src="https://github.com/user-attachments/assets/17ae3063-b6b8-406c-a637-b0942c8665cd" />
<img width="945" height="363" alt="image" src="https://github.com/user-attachments/assets/78ec1cdb-19fb-4b8d-b915-bb01b571601c" />
<img width="942" height="64" alt="image" src="https://github.com/user-attachments/assets/bec75d38-04ad-4450-9a36-b9051e7b3ee9" />

---

## 題目三 ANS




