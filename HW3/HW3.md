# HW3: Embedded Linux 掛載與 SD 卡開機 (EGO-XZ7)

本作業於 **E-Elements EGO-XZ7 (Xilinx Zynq-7000 XC7Z020-CLG484-1)** 開發板上，透過 MicroSD 卡載入 Bootloader、Linux Kernel Image 與根檔案系統 (Root File System, FS)，完成嵌入式 Linux 系統啟動與磁區掛載。

---

## 一、 基本要求與繳交檔案說明

本資料夾包含透過 SD 卡啟動 Linux 所需的核心檔案：

| 專案項目 | 檔案名稱 | 功能說明 |
| :--- | :--- | :--- |
| **Bootloader** | `BOOT.BIN` | 包含第一階段啟動程式 (FSBL)、FPGA Bitstream (`.bit`) 與第二階段啟動程式 (`u-boot.elf`)，負責初始化 Zynq PS 端硬體與載入核心。 |
| **① Kernel Image** | `uImage` (或 `image.ub`) | 針對 ARM Cortex-A9 架構編譯之 Linux 核心映像檔。 |
| **Device Tree** | `devicetree.dtb` | 硬體設備樹檔案，定義 Zynq PS 周邊（UART、SD、Ethernet、DDR3）與 AXI 記憶體映射位址。 |
| **② File System (FS)** | `uramdisk.image.gz` | 根檔案系統 (RootFS) 映像檔，於開機時掛載至記憶體中運行 Linux Shell 環境。 |

---

## 二、 硬體配置與開機設定 (EGO-XZ7)

### 1. 開機模式指撥開關 (`SW8`)
根據 EGO-XZ7 硬體手冊，將板載指撥開關 **`SW8`** 設定為 **MicroSD 卡啟動模式 (`SW8[4:2] = 110`)**：
* **電路特性**：撥至 `ON`（上方）為接地邏輯 `0`，撥至 `OFF`（下方數字側）為上拉邏輯 `1`。
* **實際撥法（由左至右 Switch 1 ~ 5）**：
  * `Switch 1 (MIO[2])`：**下 (`1`)** — Independent JTAG
  * `Switch 2 (MIO[3])`：**上 (`0`)** — Mode bit 0
  * `Switch 3 (MIO[4])`：**下 (`1`)** — Mode bit 1
  * `Switch 4 (MIO[5])`：**下 (`1`)** — Mode bit 2
  * `Switch 5 (MIO[6])`：**上 (`0`)** — Enable PS internal PLL
  * `Switch 5 (MIO[7])`：**皆可** — 無功能

### 2. 硬體連接與序列埠 (UART) 設定
1. 將格式化為 `FAT32` 並放入上述開機檔案的 MicroSD 卡插入板載 **`J15`** 卡槽。
2. 將 Micro-USB 傳輸線連接至板載 **`J1` (USB-JTAG&UART)** 接口。
3. 開啟電腦端 PuTTY 終端機，設定連線參數：
   * **Connection type**: `Serial`
   * **Serial line**: `COM7` *(依裝置管理員實際顯示之埠號)*
   * **Speed (Baud rate)**: `115200` (8-N-1, Flow control: `None`)

---

## 三、 實驗結果與系統掛載驗證

### 1. 系統開機與登入畫面
開啟電源開關 `S1` 後，U-Boot 自動載入 **① Kernel Image** 與 **② FS**，成功進入嵌入式 Linux 終端機畫面並以 `root` 身分登入：

<!-- 請將開機與登入截圖上傳至同目錄，並修改下方圖片檔名 -->
<img width="1514" height="1033" alt="image" src="https://github.com/user-attachments/assets/1d2ad828-71c8-4696-ad54-3d0cbadbc460" />

### 2. 核心版本與 SD 卡檔案系統掛載 (Mount)
於 Linux 終端機中執行以下指令，驗證 Kernel 版本並將 SD 卡分割區 (`/dev/mmcblk0p1`) 掛載至 `/mnt` 目錄下進行讀寫：

```bash
# 檢查 ① Kernel Image 版本資訊
uname -a

# 檢查 ② FS 根檔案系統空間與掛載狀態
df -h
ls /

# 將 SD 卡第一分割區掛載至 /mnt 並檢視開機檔案
mount /dev/mmcblk0p1 /mnt
ls -la /mntE.md           # 附上開機流程說明、PuTTY 驗證截圖 (若做 Bonus 則加上 Source URL)

## Demo
