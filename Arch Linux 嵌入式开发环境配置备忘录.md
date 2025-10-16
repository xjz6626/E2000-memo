# **Arch Linux 嵌入式开发环境配置备忘录**

本文档记录了从零开始，在 Arch Linux 主机上配置连接嵌入式开发板（E2000-Ubuntu）并共享网络的完整调试流程。

## **第一部分：串口连接调试 (Serial Port)**

目标：通过 minicom 或 picocom 等工具，成功通过串口连接到开发板的控制台。

### **1.1 安装串口工具**

\# 安装经典的 minicom  
sudo pacman \-S minicom

\# 安装轻量级的 picocom (推荐)  
sudo pacman \-S picocom

### **1.2 解决串口权限问题**

直接连接通常会失败，提示 Permission denied。

#### **1.2.1 确认设备名和用户组**

首先，插入USB转串口设备，然后用 dmesg 查找设备名。

dmesg | grep tty  
\# 输出通常会包含 ttyUSB0 或 ttyACM0

接着，查看设备文件的权限。

ls \-l /dev/ttyUSB0  
\# crw-rw---- 1 root uucp ...

**关键点**：注意上面的用户组是 uucp。在很多其他发行版中是 dialout，但在我们的 Arch 系统上是 uucp。这意味着我们的用户需要被添加到 uucp 组。

#### **1.2.2 将用户添加到 uucp 组**

\# 将当前用户添加到 uucp 组  
sudo usermod \-a \-G uucp $USER

**重要**：执行此命令后，必须**完全注销并重新登录**桌面环境，新的用户组权限才会生效。

### **1.3 成功连接**

完成权限配置后，使用 picocom 连接。

picocom \--baud 115200 /dev/ttyUSB0

* **退出 Picocom**: 按 Ctrl+A 然后按 Ctrl+X。

## **第二部分：共享主机网络给开发板**

目标：让 Arch 主机通过 Wi-Fi 上网，然后通过有线网口将网络共享给开发板。

### **2.1 识别网卡硬件与驱动问题**

首先发现图形化网络设置中没有“有线连接”选项。

#### **2.1.1 检查系统网络接口**

ip link show 命令的输出中没有 eth0 或 enp... 等有线网卡设备。

#### **2.1.2 识别硬件型号**

使用 lspci 查找硬件，发现系统未能加载驱动。

lspci \-k | grep \-i ethernet \-A 3  
\# 输出显示:  
\# 02:00.0 Ethernet controller: Motorcomm Microelectronics. YT6801 Gigabit Ethernet Controller  
\# (并且该条目下没有 "Kernel driver in use" 字样)

**结论**: 系统缺少 Motorcomm YT6801 网卡的驱动。

### **2.2 从 AUR 安装驱动**

Arch Linux 的官方仓库没有这个驱动，需要从 AUR (Arch User Repository) 安装。

#### **2.2.1 安装 AUR 助手 yay (若无)**

sudo pacman \-S \--needed base-devel git  
git clone \[https://aur.archlinux.org/yay.git\](https://aur.archlinux.org/yay.git)  
cd yay  
makepkg \-si  
cd .. \# 返回上级目录

#### **2.2.2 安装内核头文件**

编译内核驱动需要对应版本的头文件。假设使用的是 zen 内核。

\# uname \-r  (可查看当前内核版本)  
sudo pacman \-S linux-zen-headers \# 根据你的内核版本选择，例如 linux-headers

#### **2.2.3 安装 DKMS 驱动**

yay \-S yt6801-dkms

#### **2.2.4 加载并验证驱动**

sudo modprobe yt6801  
ip link show

此时应该能看到 enp2s0 等有线网卡接口出现了。重启电脑是确保驱动正确加载的最稳妥方法。

### **2.3 配置 NetworkManager 共享**

#### **2.3.1 安装 dnsmasq 依赖**

NetworkManager 的共享功能依赖 dnsmasq 来提供 DHCP 服务。

sudo pacman \-S dnsmasq

#### **2.3.2 重启 NetworkManager**

sudo systemctl restart NetworkManager

#### **2.3.3 在图形界面设置共享**

1. 用网线连接两台电脑。  
2. 进入“设置” \-\> “网络”。  
3. 此时“有线连接”应该已出现。点击其旁边的齿轮图标 ⚙。  
4. 进入 “IPv4” 选项卡，将 “Method” 设置为 “**与其他计算机共享**”。  
5. 点击“应用”。

### **2.4 手动配置防火墙 (iptables)**

此时客户端能获取 IP (10.42.0.x)，DNS也能通，但 ping 不通外网。这是因为缺少 NAT 转发规则。

#### **2.4.1 开启内核 IP 转发**

\# 临时生效  
sudo sysctl net.ipv4.ip\_forward=1  
\# 永久生效  
echo 'net.ipv4.ip\_forward \= 1' | sudo tee /etc/sysctl.d/99-forwarding.conf

#### **2.4.2 添加 iptables 规则**

\# 规则1：允许已建立的连接通过  
sudo iptables \-A FORWARD \-m conntrack \--ctstate RELATED,ESTABLISHED \-j ACCEPT

\# 规则2：允许从共享网口到外网的新连接  
sudo iptables \-A FORWARD \-i enp2s0 \-o wlan0 \-j ACCEPT

\# 规则3：设置NAT地址伪装  
sudo iptables \-t nat \-A POSTROUTING \-s 10.42.0.0/24 \-o wlan0 \-j MASQUERADE

**注意**：-i enp2s0 是你的有线网卡名，-o wlan0 是你的 Wi-Fi 网卡名。请根据 ip a 的实际输出进行调整。

#### **2.4.3 永久保存 iptables 规则**

\# 安装用于保存规则的服务包 (安装时按 'y' 同意替换旧的 iptables)  
sudo pacman \-S iptables-nft

\# 保存当前生效的规则到文件  
sudo iptables-save | sudo tee /etc/iptables/iptables.rules

\# 设置开机自启服务  
sudo systemctl enable iptables.service

## **第三部分：VS Code 远程 SSH 连接**

目标：在 Arch 主机上使用 VS Code，通过 SSH 远程开发 E2000 板子。

### **3.1 安装主机 SSH 客户端**

VS Code 需要使用系统的 ssh 命令。

\# openssh 包同时提供了客户端和服务端  
sudo pacman \-S openssh

### **3.2 安装 VS Code 扩展**

在 VS Code 的扩展商店中，搜索并安装由 Microsoft 官方发布的 **Remote \- SSH** 扩展。

### **3.3 连接步骤**

1. 点击 VS Code 左下角的绿色 \>\< 图标。  
2. 选择 “Connect to Host...”。  
3. 选择 “+ Add New SSH Host...”。  
4. 输入连接命令：ssh root@10.42.0.102。  
5. 选择默认的 SSH 配置文件保存。  
6. 点击 “Connect”，在弹出的窗口中输入 root 用户的密码。  
7. 连接成功后，左下角会显示 SSH: 10.42.0.102。

至此，整个开发环境配置完成。