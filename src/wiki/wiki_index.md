# Rust for Linux Wiki

本项目的 Wiki 主要聚焦 Rust for Linux 的代码实现，并介绍各模块功能，该 Wiki 会随着版本实时更新。

在开始之前，需要准备一个可运行 Rust for Linux 的环境。下文先说明笔者当前的系统环境：

``` bash
OS: Fedora 42 (Server Edition)
Gcc: gcc version 15.2.1 20251022 (Red Hat 15.2.1-3) (GCC)
Clang: clang version 20.1.8 (Fedora 20.1.8-4.fc42)
Rust: rustc 1.91.0 (f8297e351 2025-10-28)
LLVM: LLVM version 20.1.8
LLD: LLD 20.1.8 (compatible with GNU linkers)
```

## Rust for Linux 环境搭建

我们将通过编译内核并使用 QEMU 启动编译产物来搭建运行环境。为了简化流程，除 QEMU 与编译的内核外，rootfs 与 disk image 等均采用预先准备好的在线镜像。

### 安装 Rust

在 Rust for Linux 的[官方文档](https://docs.kernel.org/rust/quick-start.html)中，推荐使用发行版的包管理器安装；本文改用 Rust 官方安装脚本。**只需确保 `rustc` 版本 ≥ 1.80 即可**。

因此这里仅给出使用 Rust 官方脚本的安装方式：

> 对于国内用户，可以采用 [rsproxy](https://rsproxy.cn/) 更换镜像源后进行安装

``` bash
curl --proto '=https' --tlsv1.2 -sSf https://rsproxy.cn/rustup-init.sh | sh
```

安装完成后，检查版本确保 `rustc` 版本 ≥ 1.80。

``` bash
rustc --version
rustc 1.91.0 (f8297e351 2025-10-28)
```

当然，你可以通过如下代码进行简单验证：

``` rust
# fn main() {
    println!("Hello Rust Version Done! Version: {:?}", std::env::var("CARGO_PKG_VERSION"))
# }
```

通过官方安装脚本安装时，通常也会安装 `rustfmt` 和 `rust-clippy`。可通过以下命令验证是否安装成功：

``` bash
rustfmt --version
rustfmt 1.8.0-stable (f8297e351a 2025-10-28)

cargo clippy --version
clippy 0.1.91 (f8297e351a 2025-10-28)
```

如若没有安装，请通过如下指令进行安装：

``` bash
rustup component add rust-src rustfmt clippy
```

进行 Rust for Linux 开发还需安装 `bindgen`，用于从 C 头文件生成 Rust FFI 绑定。

``` bash
cargo install --locked bindgen-cli
```

### 安装 QEMU

后续将通过 QEMU 启动镜像，因此需要先安装 QEMU。

对于 QEMU 的安装而言，通常分为两种方式：包管理安装和手动编译。笔者通常喜欢手动编译，但下方会给出两种安装方式，读者可以自行选择其中一种进行安装。

#### 包管理安装

包管理安装可以直接通过各自的系统命令进行安装，下方给出两种常见发行版本安装命令：

- Ubuntu/Debian（全量安装）

``` bash
sudo apt install qemu -y
```

- Ubuntu/Debian（精简安装）

``` bash
sudo apt install qemu-system-x86 -y
```

- Fedora/Rocky（全量安装）

``` bash
sudo dnf install qemu -y
```

- Fedora/Rocky（精简安装）

``` bash
sudo dnf install qemu-system-x86 -y
```

#### 源码编译安装

这里简单阐述一下为什么选择源码编译，因为源码编译可以自行选择安装版本，避免版本过久问题。

首先安装所需的依赖包：

``` bash
sudo dnf in ninja-build python3-sphinx python3-sphinx_rtd_theme glib2-devel libslirp-devel -y
```

然后下载对应的源码并解压：

``` bash
wget https://download.qemu.org/qemu-10.1.2.tar.xz
tar Jxvf qemu-10.1.2.tar.xz
```

进入对应的目录进行配置：

``` bash
./configure --prefix=/opt/qemu --enable-kvm --enable-debug
make -j$(nproc)
```

> 精简安装：本教程主要在 `x86` 环境上进行操作，因此如果读者的资源有限或认为全量编译过慢，可以通过下方指令进行精简编译
> ``` bash
> ./configure --prefix=/opt/qemu --target-list=x86_64-softmmu --enable-kvm
> ```

编译完成后，安装到指定目录，并配置环境变量；然后进行验证：

``` bash
sudo make install
echo "export PATH=$PATH:/opt/qemu/bin" >> $HOME/.bashrc
source $HOME/.bashrc

qemu-system-x86_64 --version
QEMU emulator version 10.1.2
Copyright (c) 2003-2025 Fabrice Bellard and the QEMU Project developers
```

### 构建 Linux 内核（vmlinux）

提前准备好 Linux 编译所需要的依赖包：

``` bash
sudo dnf in gcc clang clang-tools-extra llvm lld gpg2 git gzip make openssl perl rsync binutils ncurses-devel flex bison openssl-devel elfutils-libelf-devel rpm-build
```

准备好 Rust 环境后，我们将通过编译内核源码生成内核产物（`vmlinux`）。

首先，我们直接克隆 Rust for Linux 主线分支 (**Linux 内核主线源码也是可行的，只是相较于 Rust for Linux 主线分支支持的特性稍微少一些**)。

``` bash
git clone https://github.com/Rust-for-Linux/linux.git rust-for-linux --depth=1
```

进入到 `rust-for-linux` 目录中 (这里将 Rust for Linux 主线代码重命名为 `rust-for-linux` 这是为了避免与 `linux` 内核主线重名)。然后通过如下命令检查 Rust 依赖和版本是否正确。

> **需要特别注意：编译 Rust for Linux 要求 LLVM 工具链版本 ≥ 15.0.0**

``` bash
make LLVM=1 rustavailable
Rust is available!
```

只有输出结果为 `Rust is available!` 时，才证明当前主机环境能够正常通过 Rust 编译内核。现在，我们就可以开始编译 Rust for Linux 的内核源码。

``` bash
make LLVM=1 defconfig
make LLVM=1 menuconfig
```

首先通过 `defconfig` 生成默认配置，然后使用 `menuconfig` 启用 `RUST` 和 `SAMPLES_RUST`，参数位置如下：


``` bash
config RUST
Location:
  -> General setup
    -> Rust support (RUST [=y])

config SAMPLES_RUST
Location:
  -> Kernel hacking
    -> Sample kernel code (SAMPLES [=y])
      -> Rust samples (SAMPLES_RUST [=y])
```

配置完成后，就能够正常启用 Rust 进行编译内核，然后开始编译。

``` bash
make LLVM=1 -j$(nproc)
```

在编译时，可以通过编译的日志查看到一些 Rust 编译信息：

``` bash
  RUSTC L rust/core.o
  BINDGEN rust/bindings/bindings_generated.rs
  BINDGEN rust/bindings/bindings_helpers_generated.rs
  CC      rust/helpers/helpers.o
  RUSTC P rust/libpin_init_internal.so
  RUSTC P rust/libmacros.so
```

> **对于 Rust 是必须开启 `LLVM=1` 选项的，因此不能够省略，如果想要省略，可以在编译配置前使用如下命令**:
> ``` bash
> export LLVM=1
> ```

编译完成后，可以在源码目录中看到生成的内核产物。其中：
- `vmlinux` 是未压缩的 ELF 内核文件，包含完整符号信息，主要用于调试与分析；

下文在 QEMU 启动场景中以 `vmlinux` 作为内核镜像进行说明。

### 运行 Rust for Linux 镜像

之前提到，为了简化流程，我们会直接从网络上下载已有镜像进行运行，因此需要运行如下命令：

``` bash
wget https://cdimage.debian.org/images/cloud/bookworm/20250316-2053/debian-12-nocloud-amd64-20250316-2053.qcow2 -O debian-12.qcow2
```

> 如果想要下载其他架构，请参考最下方的参考链接中的 Rust Exercises Ferrous System

为了简化下载的方便，所下载的镜像的大小不会太大，因此内部的空间不足以让我们后续替换内核镜像和加载内核模块。所以在启动之前需要进行扩容，启动后在内部修改磁盘大小以支持镜像更换。

``` bash
(host-machine) qemu-img resize debian-12.qcow2 +32G
```

至此，前期的准备工作均准备完毕，运行以下命令启动虚拟机：

``` bash
(host-machine) qemu-system-x86_64 -m 8G -M q35 -accel kvm -smp 8 \
  -hda debian-12.qcow2 \
  -device e1000,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp:127.0.0.1:5555-:22 \
  -nographic \
  -serial telnet:localhost:4321,server,wait
```

此时，该终端会一直挂起，读者需要另起一个新终端，通过 `telnet` 命令进行连接：

``` bash
telnet localhost 4321
[  OK  ] Finished systemd-user-sess…ervice - Permit User Sessions.
[  OK  ] Started getty@tty1.service - Getty on tty1.
[  OK  ] Started serial-getty@ttyS0…rvice - Serial Getty on ttyS0.
[  OK  ] Reached target getty.target - Login Prompts.
[  OK  ] Started dbus.service - D-Bus System Message Bus.
[  OK  ] Started systemd-logind.service - User Login Management.
[  OK  ] Started unattended-upgrade…0m - Unattended Upgrades Shutdown.
[  OK  ] Finished e2scrub_reap.serv…ine ext4 Metadata Check Snapshots.
[  OK  ] Reached target multi-user.target - Multi-User System.
[  OK  ] Reached target graphical.target - Graphical Interface.
         Starting systemd-update-ut… Record Runlevel Change in UTMP...
[  OK  ] Finished systemd-update-ut… - Record Runlevel Change in UTMP.

Debian GNU/Linux 12 localhost ttyS0

localhost login:
```

此时，就启动了对应的内核镜像，通过 `root` 用户登录 (无密码)。当前启动的内核是自带内核，因此需要更换内核镜像。在这之前，我们还需要进行扩容操作，并且需要配置好 `ssh` 以方便将内核镜像传入到虚拟机中。

``` bash
(virt-machine) uname -a
Linux localhost 6.1.0-32-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.129-1 (2025-03-06) x86_64 GNU/Linux

(virt-machine) apt update
(virt-machine) apt install fdisk
(virt-machine) cfdisk /dev/*da
```

![disk resize](../imgs/png/disk_resize.png)

执行上述命令后，可以看到如上所示的界面。首先选择 `Sort`，此时你会发现 `Linux root` 往下移动，与 `Free space` 近邻，然后如下所示，选择 `Linux root` 对应的磁盘，并且选择 `Resize` 选项。

![disk resize 2](../imgs/png/disk_resize_2.png)

然后键入两次回车，发现 `Linux root` 的大小更改后，选择 `Write` 选项，然后键入 `yes` 确定修改，如下所示：

![disk resize 3](../imgs/png/disk_resize_3.png)

至此，选择 `Quit` 选项即可，使用 `reboot` 重启虚拟机，查看是否更改成功：

``` bash
(virt-machine) reboot

(virt-machine) df -h
Filesystem      Size  Used Avail Use% Mounted on
...
/dev/sda3        38G  1.2G   35G   4% /
...
```

修改完成后，接下来需要配置 `ssh`：

``` bash
(virt-machine) apt install openssh-server
```

并将主机的 `ssh` 公钥复制到虚拟机中的 `.ssh/authorized_keys` 文件中。接着将上方我们编译的内核传入到虚拟机中：

``` bash
(host-machine) scp -P 5555 rust-for-linux/vmlinux root@localhost:/root
```

此时，能够在虚拟机中发现对应的文件，将其添加可执行文件权限，并移动到 `/boot` 下

``` bash
(virt-machine) chmod +x vmlinux
(virt-machine) cp /boot/vmlinuz-6.1.0-32-amd64 /boot/vmlinuz-6.1.0-32-amd64-bak
(virt-machine) cp vmlinux /boot/vmlinuz-6.1.0-32-amd64
(virt-machine) reboot
```

重启后，如无任何报错信息，则更换内核完毕，通过如下命令查看：

``` bash
(virt-machine) uname -a
```

至此，启动一个 Rust for Linux 编译的内核镜像就成功完成了。

### Tips：启用 rust-analyzer

Rust for Linux 提供了 `make rust-analyzer` 命令，便于启用 `rust-analyzer` 以更高效地编写和查看 Rust 代码。

通常我会按如下步骤启用 `clangd` 和 `rust-analyzer`，以同时支持内核 C 源码与 Rust 代码：

``` bash
bear -- make LLVM=1 -j`nproc`
make LLVM=1 rust-analyzer
```

---

参考链接

- [Rust Exercises Ferrous System](https://rust-exercises.ferrous-systems.com/latest/book/building-linux-kernel-driver)
- [Rust for Linux Quick Start](https://docs.kernel.org/rust/quick-start.html)
