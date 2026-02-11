# Arch FAT Boot Failure

**问题**：在 Arch Linux 系统中，旧Linux内核版本为6.17.8-arch1-1，尝试运行 `sudo pacman -Syu` 更新了系统，新内核版本为6.18.8-arch2-2。重启后无法进入图形界面，直接进入了一个**紧急救援模式**，显示`failed to mount /boot`。此 Shell 具有 `root` 权限，但多数服务未运行（如NetworkManager）。在shell中将内核版本回退至旧版本，可以正常启动。

---

### Phase 1：在紧急 Shell 中尝试手动挂载

```bash
# 在紧急 Shell 中执行
mount /boot
```

报错 `vfat` 文件系统模块丢失。
```
mount: /boot: unknown filesystem type 'vfat'.
```
分析原因可能是**新内核与模块版本不匹配**。系统尝试用**新内核**启动，但在引导的早期阶段失败了，然后 `systemd` 启动了紧急 Shell。然而，这个紧急 Shell 环境可能继承了引导失败时的内核状态，或者其 `initramfs` 本身就不完整，导致无法加载 `vfat` 模块。

---

### Phase 2：尝试联网重新安装一遍新内核

在紧急 Shell 中，网络服务并未运行。直接使用 `nmcli` 连接网络提示命令不存在`could not connect: No such file or directory`。

```bash
nmcli device wifi list
```
尝试手动启动 `NetworkManager` 服务。

```bash
systemctl start NetworkManager
```
结果命令直接**卡死**。怀疑是救援模式下的 `systemd` 状态不完整，无法支撑像 NetworkManager 这样的复杂服务。

最终用手机usb共享网络，利用系统内置的`systemd-networkd` 服务联网。

```bash
# 1. 确认 USB 网卡名
ip link
# 2. 创建临时配置文件，告知 systemd-networkd 使用 DHCP
sudo nano /etc/systemd/network/20-usb.network
# 输入以下内容
[Match]
Name=enp0s20u1u2（刚才的网卡名）
[Network]
DHCP=yes

# 3. 手动指定 DNS
sudo nano /etc/resolv.conf
# 输入以下内容
nameserver 8.8.8.8

# 4. 启动 systemd-networkd
sudo systemctl start systemd-networkd

# 检测是否获取到IP
ip addr
```
---

### Phase 3：在线修复尝试失败

尝试在救援模式下直接修复系统。运行 `pacman -Syu`，尝试重新同步和安装所有软件包，并未报错。重启后依然未解决。

此时，怀疑 `/boot` 无法自动挂载是 `systemd` 的启动时序问题（即硬件未就绪）。 检查：

```bash
# 1. 检查 VFAT 文件系统，发现文件系统健康。 sudo fsck.vfat -r -w /dev/nvme0n1p1 
# 2. 为排除时序问题，修改 /etc/fstab，为 /boot 添加超时选项 
# 修改前
UUID=XXXX-XXXX    /boot    vfat    rw,relatime,fmask=...    0 2
# 修改后
UUID=... /boot vfat defaults,nofail,x-systemd.device-timeout=10 0 2
```

**这个修改的目的是让 `systemd` 在启动时为 `/boot` 设备等待 10 秒，如果失败则跳过而不是让系统崩溃。**

尝试重新生成 `initramfs` 来应用新配置。 ```sudo mkinitcpio -P ``` 。更新 GRUB 配置：

```bash 
# 1. 刷新 systemd 配置以识别修改后的 fstab 
sudo systemctl daemon-reload 
# 2. 挂载 /boot 并生成 GRUB 配置 
sudo mount /boot 
sudo grub-mkconfig -o /boot/grub/grub.cfg 
```

重启电脑。 **观察到系统启动时确实暂停了**，但等待结束后挂载依然失败。很有趣的是最终系统放弃启动新内核，自动回退并成功启动了旧内核。

怀疑`initramfs` 文件本身存在缺陷（很可能是在第一次更新时，在一个不稳定的环境中生成的），导致新内核在引导的最初阶段就缺少必要的驱动（如 NVMe 或 VFAT），因此即使等待也无法识别和挂载 `/boot` 分区。在救援模式下进行的所有修复操作都继承了这种缺陷。

---

### Phase 4：使用 Arch Live USB 进行 Chroot 外部修复

制作 Arch Linux 安装 U 盘（**见附录**），用U盘启动。在 Live 环境中挂载故障系统。

```bash
# 1. 联网 (iwctl)
iwctl
[iwd]# device list (查看你的无线网卡名, 如wlan0)
[iwd]# station wlan0 scan
[iwd]# station wlan0 get-networks
[iwd]# station wlan0 connect 你的WiFi名 
# (输入密码)
[iwd]# exit

# 2. 挂载根分区
mount /dev/nvme0n1p6 /mnt
# 3. 挂载引导分区
mount /dev/nvme0n1p1 /mnt/boot
```

使用 `arch-chroot` 进入故障系统，获得一个功能完整的 Shell。

```bash
arch-chroot /mnt
```

在这个健康的 Chroot 环境中，强制重装关键组件。
```bash
# 1. 强制重新安装内核。这将调用 mkinitcpio 在一个健康的环境下，创建一个完整、正确的 initramfs。
pacman -S linux

# 2. 彻底重写 GRUB 引导文件到 EFI 分区
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

# 3. 重新生成最终的 GRUB 配置文件
grub-mkconfig -o /boot/grub/grub.cfg
```

退出 Chroot，卸载分区，拔掉 U 盘并重启。

```bash
exit
umount -R /mnt
reboot
```

**最终成功:** 系统正常启动，进入了最新的内核。`uname -r` 显示新版本，`lsblk` 显示 `/boot` 已自动挂载。问题彻底解决。



### 附录：制作 Arch Linux 修复 U 盘

为了执行 Chroot 外部修复，需要一个可启动的 Arch Linux U 盘。

**1. 下载镜像:**

   - 前往 Arch Linux 官方网站：[https://archlinux.org/download/](https://archlinux.org/download/)
   - 从任意镜像站点下载最新的 `.iso` 文件。

**2. 准备 U 盘:**

   - 准备一个容量至少 8GB 的 U 盘。
   - **警告：** U 盘上的所有数据都将被完全擦除。

**3. 写入镜像 (以 Windows 平台为例):**

   - 下载 **Rufus** ([https://rufus.ie/](https://rufus.ie/))。
   - 在 Rufus 中：
     - **设备 :** 选择 U 盘。
     - **引导类型选择:** 点击 "选择" (SELECT)，加载下载的 Arch Linux `.iso` 文件。
     - **分区类型**：MBR；目标系统类型：BIOS或UEFI
     - **文件系统**：Large FAT32
     - **开始:** 点击开始。当弹出窗口询问写入模式时，**选择 `以  镜像模式写入` (Write in DD Image mode)**。
     - 确认所有警告，等待进度条完成。

制作完成后，此 U 盘即可用于启动任何电脑进入 Arch Linux 的 Live 环境，以进行系统安装或修复。