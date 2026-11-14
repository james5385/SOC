# HW3: Embedded Linux 掛載與 SD 卡開機 (EGO-XZ7)

本作業於 **E-Elements EGO-XZ7 (Xilinx Zynq-7000 XC7Z020-CLG484-1)** 開發板上，透過 MicroSD 卡載入 Bootloader、Linux Kernel Image 與根檔案系統 (Root File System, FS)，完成嵌入式 Linux 系統啟動與磁區掛載。

---

## 一、 基本要求與繳交檔案說明

本資料夾包含透過 SD 卡啟動 Linux 所需的核心檔案：

| 專案項目 | 檔案名稱 | 功能說明 |
| :--- | :--- | :--- |
| **Bootloader** | `BOOT.BIN` | 包含第一階段啟動程式 (FSBL)、FPGA Bitstream (`.bit`) 與第二階段啟動程式 (`u-boot.elf`)，負責初始化 Zynq PS 端硬體與載入核心。 |
| **image.ub** |  `image.ub` | 內含Kernel Image, Device Tree 和 File System。 |

內含在 image.ub 的檔案：

| 專案項目 | 檔案名稱 | 功能說明 |
| :--- | :--- | :--- |
| **Kernel Image** |  `uImage` | 作業系統的核心引擎。負責管理 CPU 排程、DDR 記憶體配置、檔案系統讀寫以及所有硬體的驅動程式（Drivers）。 |
| **Device Tree** | `devicetree.dtb` | 告訴 Kernel「這塊板子有哪些周邊、接在哪個記憶體位址、用哪號中斷（IRQ）」。沒有它，Kernel 就算有驅動也不知道硬體在哪裡。 |
| **File System (FS)** | `uramdisk.image.gz` | 提供使用者看得到的目錄（/bin, /etc, /dev）與常用指令（ls, cd, mount）。Kernel 啟動到最後一步必須掛載它，你才能登入打指令。 |

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
  * `Switch 6 (MIO[7])`：**皆可** — 無功能

### 2. 硬體連接與序列埠 (UART) 設定
1. 將格式化為 `FAT32` 並放入上述開機檔案的 MicroSD 卡插入板載 **`J15`** 卡槽。
2. 將 Micro-USB 傳輸線連接至板載 **`J1` (USB-JTAG&UART)** 接口。
3. 開啟電腦端 PuTTY 終端機，設定連線參數：
   * **Connection type**: `Serial`
   * **Serial line**: `COM3` *(依裝置管理員實際顯示之埠號)*
   * **Speed (Baud rate)**: `115200` (8-N-1, Flow control: `None`)

---

1. Bootloader (BOOT.BIN) 先醒來，把 SD 卡裡的 Kernel (uImage)、Device Tree (devicetree.dtb)、FS (uramdisk.image.gz) 分別搬到 DDR3 記憶體的三個不同位址。
2. U-Boot 執行 bootm 指令，啟動 Kernel (uImage)，並把 Device Tree 與 FS 的記憶體位址告訴它。
3. Kernel 讀取 Device Tree 認識板子上的 UART (J1) 與 SD 卡槽 (J15)，把開機訊息印到你的 PuTTY 畫面上。
4. Kernel 最後掛載 FS，啟動 Shell，在 PuTTY 印出 login: 讓你登入！   

## 三、 實驗結果與系統掛載驗證

### 1. 系統開機與登入畫面
開啟電源開關 `S1` 後，U-Boot 自動載入 **① Kernel Image** 與 **② FS**，成功進入嵌入式 Linux 終端機畫面並以 `root` 身分登入：

<!-- 請將開機與登入截圖上傳至同目錄，並修改下方圖片檔名 -->
<img width="661" height="419" alt="image" src="https://github.com/user-attachments/assets/2cf3dcfd-4fa3-4d71-b6b3-abf3203ad33e" />

<img width="810" height="1028" alt="image" src="https://github.com/user-attachments/assets/8f0c10e9-5f4e-4787-8df5-41da81a3e147" />

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
