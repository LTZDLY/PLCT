# WSL2上通过qemu虚拟机运行riscv-FreeBSD

准备工作

WSL2的安装：略

QEMU的安装：略

## 准备U-Boot文件

u-boot.bin引导加载文件一般存放于/usr/local/share/u-boot/u-boot-qemu-riscv64/u-boot.bin目录下，如果没有此文件，则需要自行编译生成。过程如下：

```bash
# 安装编译工具包
user@wsl:~$ sudo apt install gcc-riscv64-linux-gnu
# 安装OpenSSL源码包
user@wsl:~$ sudo apt-get install libssl-dev

# 从仓库下载源码
user@wsl:~$ git clone https://source.denx.de/u-boot/u-boot.git
user@wsl:~$ cd u-boot
# 开始编译
user@wsl:~/u-boot$ export CROSS_COMPILE=riscv64-linux-gnu-
user@wsl:~/u-boot$ make qemu-riscv64_smode_defconfig
user@wsl:~/u-boot$ make -j16
```

编译成功后会在该目录下生成一个u-boot.bin文件。记住他的位置或将其放置在上述文件目录下。

## 准备OpenSBI

fw_jump.elf是opensbi的一种引导启动模式，一般存放于/usr/local/share/opensbi/lp64/generic/firmware/fw_jump.elf目录下。若没有此文件，则需要自行编译生成。过程如下：

```bash
# 安装编译工具包
user@wsl:~$ sudo apt install gcc-riscv64-unknown-elf

# 从仓库下载源码
user@wsl:~$ git clone https://github.com/riscv/opensbi.git
user@wsl:~$ cd opensbi
# 开始编译
user@wsl:~/opensbi$ export CROSS_COMPILE=riscv64-unknown-elf-
user@wsl:~/opensbi$ make PLATFORM=generic clean
user@wsl:~/opensbi$ make PLATFORM=generic FW_JUMP_ADDR=0x80200000
```

编译成功后会在build/platform/generic/firmware目录下生成三个*.elf文件，记住其中fw_jump.elf的位置或将所有文件移动至上述文件目录下。

## 准备FreeBSD镜像文件并尝试运行

```bash
user@wsl:~$ mkdir freebsd
user@wsl:~$ cd freebsd

# 下载镜像文件
user@wsl:~/freebsd$ wget https://download.freebsd.org/ftp/snapshots/VM-IMAGES/14.0-CURRENT/riscv64/Latest/FreeBSD-14.0-CURRENT-riscv-riscv64.raw.xz
user@wsl:~/freebsd$ xz --decompress FreeBSD-14.0-CURRENT-riscv-riscv64.raw.xz
```

启动命令过长，考虑创建一个bash文件专门用于FreeBSD的启动。文件内容如下所示：

./start_freebsd.sh

```bash
#! /bin/bash

# 下面两个填写分配的虚拟cpu核数和内存大小，单位GB
vcpu=4
memory=4
memory_append=`expr $memory \* 1024`
# 下面填写自己的三个文件所在目录
drive="FreeBSD-14.0-CURRENT-riscv-riscv64.raw"
fw="fw_jump.elf"
bin="u-boot.bin"
# 下面填写转发端口号，FreeBSD默认为8022
ssh_port=8022

cmd="qemu-system-riscv64 \
	-nographic -machine virt \
	-smp "$vcpu" -m "$memory"G \
    -kernel "$bin" \
    -bios "$fw" \
    -drive file="$drive",format=raw,id=hd0 \
    -device virtio-blk-device,drive=hd0 \
    -netdev user,id=net0,ipv6=off,hostfwd=tcp::"$ssh_port"-:22 \
    -device virtio-net-device,netdev=net0
"

eval $cmd
```

运行start_freebsd.sh

```bash
user@wsl:~/freebsd$ bash start_freebsd.sh
```

观察到FreeBSD的开机动画并等待加载，不出意外的话即可正常启动。默认账户root，默认密码为空。

*可以先使用fetch命令保存一个网页来测试网络连接是否正常。*

## FreeBSD配置ssh

FreeBSD成功启动后，首先初始化pkg，并且设置用户密码

```bash
root@freebsd:~ # pkg
The package management tool is not yet installed on your system.
Do you want to fetch and install it now? [y/N]: y

root@freebsd:~ # passwd
Changing local password for root
New Password:
Retype New Password:
```

配置ssh过程如下所示：

>1. 使用任意文本编辑器编辑/etc/inetd.conf，去掉ssh前的#，保存退出。**FreeBSD自带文本编辑器为ee，若不会用可以考虑在配置pkg后安装其他文本编辑器，如nano。**
>
>2. 编辑/etc/rc.conf，添加一行sshd_enable="YES"
>
>3. 编辑/etc/ssh/sshd_config，将
>     \#PermitRootLogin no改为PermitRootLogin yes // 允许root登陆
>     \#PasswordAuthentication no改为PasswordAuthentication yes // 使用系统PAM认证
>     \#PermitEmptyPasswords no改为PermitEmptyPasswords no // 不允许空密码
>
>4. 修改端口号：/etc/ssh/sshd_config
>     \#Port=22改为 Port=22
>
>5. 保存退出
>
>6. 启动SSHD服务，/etc/rc.d/sshd start
>
>7. 查看服务是否启动，netstat -an，如果看到22端口有监听，恭喜！！！