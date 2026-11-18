下載複製貼上套件
sudo apt -y install open-vm-tools-desktop

掛載HW4
ls /mnt/hgfs/hw4
若 /mnt/hgfs 下沒有東西，執行這行掛載：
sudo vmhgfs-fuse .host:/ /mnt/hgfs -o allow_other
確認有無掛載HW4
ls /mnt/hgfs/hw4

在虛擬機內安裝 ARM 交叉編譯器與必備套件
如果你的 VMware 跑的是 Ubuntu / Debian：
sudo apt-get update
sudo apt-get install -y git make gcc gcc-arm-linux-gnueabihf libssl-dev libncurses5-dev bc bison flex u-boot-tools

比對板子版本、編譯 hw2_driver.ko 並傳回 Windows
1. 下載 Xilinx Linux Kernel 原始碼 (約需 1~2 分鐘)
cd ~
git clone --depth 1 -b xilinx-v2015.4 https://github.com/Xilinx/linux-xlnx.git
cd ~/linux-xlnx

2. 產生 Zynq 核心設定與標頭檔
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabihf-
make xilinx_zynq_defconfig
make modules_prepare

接著對齊核心版本字串（這步最關鍵！）：
將下面雙引號內的 "4.9.0-xilinx" 改成你在 PuTTY 輸入 uname -r 看到的「一模一樣」的字串！
echo '#define UTS_RELEASE "4.0.0"' > ~/linux-xlnx/include/generated/utsrelease.h

步驟 1：在 Windows 桌面的 hw4 資料夾建立 build.sh 腳本
回到你的 Windows 桌面，打開 hw4 資料夾。

在資料夾空白處按右鍵 -> 新增 -> 文字文件，將檔名（連同 .txt 副檔名）改為 build.sh。

對著 build.sh 按右鍵 -> 用 記事本 (Notepad) 打開，將以下整段內容完整複製並貼上，按 Ctrl + S 存檔並關閉：

""" script:
#!/bin/bash
set -e

echo "=== [1/4] 準備工作目錄 ~/build_hw4 ==="
mkdir -p ~/build_hw4
cd ~/build_hw4

echo "=== [2/4] 產生 hw2_driver.c 與 Makefile ==="
cat << 'EOF' > hw2_driver.c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/fs.h>
#include <linux/init.h>
#include <linux/io.h>
#include <linux/uaccess.h>

#define DEVICE_NAME  "hw2_ip"
#define MAJOR_NUM    240
#define HW2_PHY_ADDR 0x43C00000
#define HW2_REG_SIZE 0x10000

static void __iomem *hw2_base_addr;

static int hw2_open(struct inode *inode, struct file *file) {
    printk(KERN_INFO "hw2_driver: Device opened\n");
    return 0;
}

static int hw2_release(struct inode *inode, struct file *file) {
    printk(KERN_INFO "hw2_driver: Device closed\n");
    return 0;
}

static ssize_t hw2_read(struct file *file, char __user *buf, size_t count, loff_t *ppos) {
    u32 val = ioread32(hw2_base_addr);
    if (copy_to_user(buf, &val, sizeof(u32))) return -EFAULT;
    return sizeof(u32);
}

static ssize_t hw2_write(struct file *file, const char __user *buf, size_t count, loff_t *ppos) {
    u32 val;
    if (copy_from_user(&val, buf, sizeof(u32))) return -EFAULT;
    iowrite32(val, hw2_base_addr);
    return sizeof(u32);
}

static struct file_operations hw2_fops = {
    .owner   = THIS_MODULE,
    .open    = hw2_open,
    .release = hw2_release,
    .read    = hw2_read,
    .write   = hw2_write,
};

static int __init hw2_init(void) {
    int ret = register_chrdev(MAJOR_NUM, DEVICE_NAME, &hw2_fops);
    if (ret < 0) return ret;
    hw2_base_addr = ioremap(HW2_PHY_ADDR, HW2_REG_SIZE);
    printk(KERN_INFO "hw2_driver: Module loaded at 0x%x\n", HW2_PHY_ADDR);
    return 0;
}

static void __exit hw2_exit(void) {
    iounmap(hw2_base_addr);
    unregister_chrdev(MAJOR_NUM, DEVICE_NAME);
    printk(KERN_INFO "hw2_driver: Module unloaded\n");
}

module_init(hw2_init);
module_exit(hw2_exit);
MODULE_LICENSE("GPL");
EOF

# 加上 ccflags-y := -march=armv7-a 解決 isb/dsb 指令集報錯
printf "obj-m += hw2_driver.o\nccflags-y := -march=armv7-a -Wno-missing-attributes\nKDIR := \$(HOME)/linux-xlnx\nPWD := \$(shell pwd)\nall:\n\tmake -C \$(KDIR) M=\$(PWD) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- HOSTCFLAGS=\"-fcommon\" modules\n" > Makefile

echo "=== [3/4] 編譯 hw2_driver.ko ==="
make

echo "=== [4/4] 產生並編譯 User-Space 主程式 app ==="
cat << 'EOF' > app.c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdint.h>

int main(int argc, char *argv[]) {
    uint32_t write_val = (argc >= 2) ? strtoul(argv[1], NULL, 0) : 1;
    uint32_t read_val  = 0;
    int fd = open("/dev/hw2_ip", O_RDWR);
    if (fd < 0) {
        perror("Failed to open /dev/hw2_ip");
        return -1;
    }
    printf("Device /dev/hw2_ip opened successfully!\n");
    write(fd, &write_val, sizeof(uint32_t));
    printf("Wrote 0x%08X to /dev/hw2_ip\n", write_val);
    read(fd, &read_val, sizeof(uint32_t));
    printf("Read  0x%08X from /dev/hw2_ip\n", read_val);
    close(fd);
    return 0;
}
EOF

arm-linux-gnueabihf-gcc -static app.c -o app

echo "=== 編譯完成！正在將檔案複製回 Windows 桌面的 hw4 資料夾 ==="
cp -f hw2_driver.ko app /mnt/hgfs/hw4/
ls -lh /mnt/hgfs/hw4/hw2_driver.ko /mnt/hgfs/hw4/app
echo "=== 全部成功！請直接從 Windows 桌面的 hw4 資料夾把檔案放進 SD 卡 ==="
"""

請切換到你的 VMware Ubuntu 終端機，先確定共用資料夾已經掛載，然後輸入這一行指令：
1. 若剛才還沒掛載 /mnt/hgfs，先執行這兩行（若已掛載可跳過）：
sudo mkdir -p /mnt/hgfs
sudo vmhgfs-fuse .host:/ /mnt/hgfs -o allow_other

2. 將 Windows 寫好的 build.sh 轉換換行符號並直接餵給 bash 自動執行！
tr -d '\r' < /mnt/hgfs/hw4/build.sh | bash

複製到 MicroSD 卡並在板子 PuTTY 載入測試
將 Windows 桌面 hw4 資料夾中最新的 hw2_driver.ko 複製進 SD 卡根目錄，插回 EGO-XZ7 開發板開機，在 PuTTY 依序執行：
關閉 HDMI 螢幕背景輪詢
echo 0 > /sys/module/drm_kms_helper/parameters/poll
查看 xiic-i2c 底下綁定的裝置名稱
ls /sys/bus/platform/drivers/xiic-i2c/
驗證方式：畫面上除了 bind、unbind、uevent 之外，會看到一個帶有位址數字的裝置資料夾（例如 41600000.i2c 或 43c00000.i2c）。
將該裝置名稱寫入 unbind 解除綁定
cd /sys/bus/platform/drivers/xiic-i2c
echo 41600000.i2c > unbind
(此時請到電腦 Vivado 點 Program Device 燒入你的 HW2.2 .bit 檔)
mount /dev/mmcblk0p1 /mnt
cd /mnt
insmod hw2_driver.ko
lsmod
mknod /dev/hw2_ip c 240 0
./app 1












