# 「折腾日记」Gentoo之旅 安装篇

<font size=5>***长文注意⚠️***</font>

## 前言

### 岚的声明
<font size=4>本文为**岚**的**折腾日记**，它只是因为我**想**而写下的，**不具备**权威性，我也**无法**保证正确，**仅供参考**</font>

### 闲扯
最近闲来无事，突然想玩Linux，于是历经4次重装试错后，便有了这篇文章


由于实际上我只尝试了一次OpenRC + Niri的组合，且在成功进入Niri后便放弃了OpenRC的使用，因此这篇文章是以Systemd为主的，不过我也尽量把我使用的OpenRC时了解过的内容作为小小的分支放在一些章节里了


这篇文章基本都是我个人的经验之谈，而且近2万字的篇幅也让文章有些杂乱了，因此别把这篇文章当硬核指南，最好只是作为某人的经验教训日志来读


## Stage3与Profile
Gentoo与其他发行版的一个明显不同之处是：用户可以通过USE flag裁剪/控制软件包的编译，从而达到剔除不必要功能、减少攻击面的效果，但由于Gentoo不是LFS，所以实际上我们还是需要一个经过发行版深度定制过的地基作为开始，我们把这个地基称为stage3

stage3是一个不含内核/模块/驱动的基础系统，说人话就是Chroot环境，Gentoo提供了多种不同的stage3，它们分别为不同Profile做了定制化，例如：`stage3-amd64-systemd`基于Profile`default/linux/amd64/23.0/systemd`

Profile是Gentoo官方提供系统级的Portage配置值，它提供一系列为某一环境特化的USE，例如前面提到的`default/linux/amd64/23.0/systemd`对应的环境是：AMD64平台下的Systemd init环境

> [!info]- 需要返回？
> 如果你是从stage3部分而来，在此返回[部署基本系统](#部署基本系统-stage3)

Profile并不是一成不变的，虽然`stage3-amd64-desktop-systemd`默认Profile是`default/linux/amd64/23.0/desktop/systemd`，但仍可通过`eselect`的profile模块切换到其他Profile，例如，在刚进入Chroot安装阶段时，如果对你刚解压的stage3的Profile不满意，可以通过如下命令变更：
```bash { class="is-expanded" }
eselect profile list

Available profile symlink targets:
  [1]   default/linux/amd64/23.0 (stable)
  [2]   default/linux/amd64/23.0/systemd (stable) * # 当前使用此Profile
  [3]   default/linux/amd64/23.0/desktop (stable)
  [4]   default/linux/amd64/23.0/desktop/systemd (stable)
  [5]   default/linux/amd64/23.0/desktop/gnome (stable)
  [6]   default/linux/amd64/23.0/desktop/gnome/systemd (stable)
  [7]   default/linux/amd64/23.0/desktop/plasma (stable)
  [8]   default/linux/amd64/23.0/desktop/plasma/systemd (stable)
  [9]   default/linux/amd64/23.0/no-multilib (stable)
  ......

# 切换至default/linux/amd64/23.0/desktop/systemd
eselect profile set 4
```

> [!warning]+ 不建议切着玩
> 虽然切换Profile是可行的，但这并不代表你可以闲着没事干来回切，因为要使Profile完全生效，需要重新编译整个系统中所有USE发生变更的软件包，并引入/移除某些软件包，这往往会出现重大功能变更甚至改变init系统，也会让你的系统更加混乱

### 几个Profile关键词
|关键词|说明|
|-|-|
|systemd|使用Systemd init系统（若无则使用OpenRC）|
|desktop|桌面环境|
|gnome|Gnome桌面环境|
|plasma|KDE Plasma桌面环境|
|no-multilib|仅64位（若无则为64位和32位）|

### 如何查看某个Profile的配置
有两种方法获取某个Profile的配置

#### emerge --info
这是最简单粗暴的方法，使用`eselect`切换到某个Profile后（无需重编，切换就行），可以使用如下命令查看当前Portage的详细配置：
```bash
# 获取完整输出
emerge --info

# 或者获取特定关键词行
# 以下命令通过grep筛选以USE开头的行
emerge --info | grep "^USE"
```

#### Gentoo软件仓库
由Gentoo官方维护的ebuild仓库储存了所有Profile的配置文件，可以通过如下方式查看：

- 已同步Gentoo仓库的系统：
```bash
# 查看基础的make配置
cat /var/db/repos/gentoo/profiles/base/make.defaults

# 查看default/linux/amd64/23.0/systemd的mask
cat /var/db/repos/gentoo/profiles/default/linux/amd64/23.0/systemd/use.mask
```

- 如果没有正在运行Gentoo的机器，可以从官方git仓库查看：
    + [Gentoo官方gitweb](https://gitweb.gentoo.org/repo/gentoo.git/tree/profiles) （不是很推荐，因为Gentoo的服务器经常502）
    + [Gentoo Github镜像](https://github.com/gentoo/gentoo/tree/master/profiles)

> [!note]~
> Gentoo仓库里的是原始的、未经整合的分散配置

### OpenRC与Systemd
Gentoo提供两种init系统选择，OpenRC与Systemd均负责系统加载，但它们的设计理念和生态有很大不同

#### OpenRC
OpenRC是Gentoo Project开发者创造的init系统，同时也是Gentoo的“Default”，它是一种基于依赖关系的init系统，其设计简单可靠，不过许多功能依赖社区组件，在开箱即用的体验上不如Systemd。就个人体验而言：如果是想完全手动配置完整环境的话，那么OpenRC的设计哲学将会是个很好的选择

#### Systemd
Systemd是目前多数发行版的默认init系统，它拥有更高的组件集成度与更广泛的桌面生态支持，不过相对的，它也就比OpenRC更复杂、更“臃肿”，攻击面也更大。Systemd是现代化桌面级系统的首选，许多软件包对Systemd的适配性要更优

> [!note]~
> 本文中，我选择的stage3/Profile是`stage3-amd64-systemd`

## 关于Portage的软件安装形式
Portage的主要软件安装形式是依据ebuild脚本编译并部署，ebuild的维护者们通常会将软件包可以修剪的部分制作成USE flag供用户选择，甚至任何人都能自建仓库编写ebuild从而完全控制软件包安装，这是Gentoo包管理与其他发行版包管理的显著不同，同时也正是Portage的灵魂所在

Gentoo官方目前也提供预编译的二进制包，但覆盖的数量很少，大多数包依旧需要自行编译；而且预编译的二进制包有两个重要的缺点：其一，USE是死的，这意味着你无法通过USE flag修剪软件包；其二，装别人编译好的二进制包意味着放弃编译器对你硬件的特殊优化

这实际上是一个很重要的权衡点，请自行考量能否接受编译安装这一形式，因为实际上使用预编译包的话，作为Arch教徒，我更推荐Arch

## 准备阶段

### 所需
- Gentoo安装映像（建议）或任意Linux环境
    + 在此获取[Gentoo ISO](https://get.gentoo.org/)
    + 或查询官方提供的[镜像站列表](https://www.gentoo.org/downloads/mirrors)
        >本文中我将默认使用最小安装映像
- U盘或其他可用于镜像写入的设备
- 写入工具（推荐Rufus）
- 一部额外的设备，用于查看文档/检索资料（虽然在安装设备上使用links等CLI浏览器查看文档是可行的，但不是很建议）
    + [Gentoo手册](https://wiki.gentoo.org/wiki/Handbook:Main_Page)
- 至少45GiB空余磁盘空间（如果可以，建议100GiB+，因为Portage默认会缓存软件包源码）
    - 注意：前面提过Portage安装软件的形式是编译，而编译会占用大量RAM，因此十分建议分配大小可观的交换空间（用磁盘空间充当RAM的后备，因此视情况需要更多的磁盘空间）。当然，如果你的RAM本身就在32GiB+这个挡位，可以当我没说

> [!tip]+ 务必确保镜像未被篡改！
> 请自行验证ISO文件可信性，Gentoo官方提供了OpenPGP和sha256两种验证方式
>   - Gentoo官方的OpenPGP公钥见此处[Gentoo公钥列表](https://www.gentoo.org/downloads/signatures/)
>   - gpg的验证方法可见下文[示例](#验证stage3可信性)
>   - Gentoo官网没有给出sha256，因为它藏在[Distfiles](https://distfiles.gentoo.org/)下ISO文件的同级目录里

### 启动安装环境
将镜像写入设备后，重启引导即可，在引导菜单中，直接选择第一项即可，随后系统init后会提示选择键盘布局，直接回车或者超时将默认US键盘布局

### 连接至互联网

#### 配置网络
为确保安装能够正常进行，首先需要配置网络（有线网络通常会自动连接，请跳到[连通性测试](#连通性测试)），Gentoo安装映像提供了多种网络配置工具，其中最简单易用是NetworkManager，首先，检查网卡是否被正确识别/驱动：
```bash
nmcli device show
```
此命令会列出所有网络适配器，可使用↑/↓(j/k)翻页，如果其中包含`GENERAL.TYPE`为`wifi`的适配器，则说明网卡被正确识别（有线网卡是`ethernet`），如果没有可用的网络设备，则大概率是网卡驱动不被安装映像包含，请自行排障（如：尝试在搜索你网卡型号的相关内容）或选择其他环境安装系统

> [!tip]~
> `lo`是本地回环，它不是真正能连接网络的设备

使用`nmtui`来打开TUI工具：
```bash
nmtui
```
进入TUI界面后，选择`Activate a connection`选项，然后`Rescan Wi-Fi`，选中无线网络后回车输入密码即可连接

#### 连通性测试
首先检查网络状态：
```bash
ifconfig
```
在一个正常联网的适配器属性中，应该能看到`inet`或者`inet6`属性，它们表示设备拿到了IP地址

测试网络访问，这里通过直接ping服务器ip来排除域名解析错误：
```bash
ping -c 4 119.29.29.29

# 正常输出
PING 119.29.29.29 (119.29.29.29) 56(84) bytes of data.
64 bytes from 119.29.29.29: icmp_seq=1 ttl=52 time=24.7 ms
64 bytes from 119.29.29.29: icmp_seq=2 ttl=52 time=24.6 ms
64 bytes from 119.29.29.29: icmp_seq=3 ttl=52 time=24.4 ms
64 bytes from 119.29.29.29: icmp_seq=4 ttl=52 time=24.1 ms

--- 119.29.29.29 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 24.070/24.435/24.666/0.240 ms
```

测试DNS解析与连接可用性：
```bash
curl gentoo.org

# 如果想要详细输出
curl -v gentoo.org
```
只要curl没报错则说明正常

#### 配置代理（可选）
将proxy地址导出为变量即可，简单配置代理的例子：
```bash
# 无鉴权
export http_proxy="http://192.168.31.33:7777"
export https_proxy="http://192.168.31.33:7777"

# 有鉴权
export http_proxy="http://user:passwd@192.168.31.33:7777"
export https_proxy="http://user:passwd@192.168.31.33:7777"
```

### 校准时钟
为保证安装环境时间正常，请运行date来确认：
```bash
date

Wed Oct 21 16:34:23 UTC 2026
```
若时间与标准时间不符合（注意：Gentoo安装映像默认是UTC+0时区！）或者不确定，运行chronyd联网同步时间：
```bash
chronyd -q
```

## 安装阶段

### 磁盘分区

#### 设计分区方案
这里简要说下两种分区表，只是想抄作业可以跳到[个人方案](#个人方案)

- 一个UEFI设备的GPT分区表如下所示：
    |挂载点|文件系统|说明|
    |-|-|-|
    |/|ext4|系统根分区|
    |/home|ext4|用于存放/home目录下的文件|
    |/efi|fat32|用于存放EFI文件的分区，可与其他系统共用同一EFI分区|
    |swap|-|交换分区，也可不分出交换分区，使用交换文件替代之|
    > [!note]- 关于Linux内核对FAT家族的驱动
    > Linux内核使用的vfat驱动实际上用于驱动所有FAT家族成员

- 一个Legacy BIOS设备的MBR分区表如下所示：
    |挂载点|文件系统|说明|
    |-|-|-|
    |/boot|ext4|用于存放/boot目录下文件，此分区用于在多分区下供引导识别，并避免可能的文件系统读取问题|
    |/|ext4|系统根分区|
    |swap|-|交换分区，也可不分出交换分区，使用交换文件替代之|
    > [!note]- 关于MBR与引导
    > MBR（Master Boot Record）中文名主引导记录。在MBR分区表中，磁盘（注意是磁盘本身，不是分区）的第一个扇区即为MBR所在的引导扇区[^1]。当设备以BIOS引导时，固件会依照启动顺序尝试磁盘，并引导它匹配到的第一个MBR；随后MBR将加载bootloader（例如GRUB），并将自身的控制权移交给bootloader[^2]
    
    > [!warning]~
    > MBR分区表下，将挂载到/boot的分区大小建议至少1G！因为它将储存initramfs等映像！

- Linux在系统分区上没有太多死要求，因此分区方案极其灵活，例如：实际上`/home`完全可以直接合并到根分区，或是你也可以将`/var`独立分区来存放Portage的缓存
    > [!tip]- Btrfs与Swapfile
    > Btrfs的实现与Swapfile存在一些冲突，解决方法详见[Btrfs文档-Swapfile](https://btrfs.readthedocs.io/en/latest/Swapfile.html)
    
    > [!warning]+ OpenRC的卸载顺序问题
    > OpenRC用户请注意，将`/var`独立分区将导致rc在关机时先把`/var`卸了，然后再尝试往`/var`里写log

#### 个人方案
我个人喜欢用Btrfs + 子卷的方式：

|挂载点|文件系统|说明|btrfs子卷|
|-|-|-|-|
|/|btrfs|系统根分区|/@|
|/home|btrfs|用于存放/home目录下的文件|/@home|
|/var|btrfs|用于存放/var目录下的文件（这也会导致OpenRC出现log失败！Systemd无影响）|/@var|
|/efi|fat32|UEFI引导分区|-|

#### 分区
Gentoo安装映像提供了完备的工具链用于磁盘分区，这里使用兼用性更好的parted编辑分区表：

- 首先查看所有磁盘：
    ```bash
    parted -l

    # 示例输出
    Model: XXXXXXXXX (nvme)
    Disk /dev/nvme1n1: 512GB
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    Disk Flags:

    Number  Start   End    Size   File system  Name   Flags
     1      1049kB  164GB  164GB  ntfs         Maple  msftdata
     2      164GB   512GB  348GB  btrfs

    Model: XXXXXXXXX (scsi)
    Disk /dev/sda: 512GB
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    Disk Flags:

    Number  Start   End    Size   File system  Name                  Flags
     1      2097kB  512GB  512GB  ntfs         Basic data partition  msftdata
    ......
    ```
    上面的输出列出了两块磁盘，其中/dev/nvme说明这是一块nvme接口下的硬盘，而/dev/sda是通过其他接口连接的硬盘

- 编辑磁盘分区表：
    ```bash
    parted /dev/nvme0n1
    ```

- 进入parted后，可通过命令help来查看帮助，输入print回车查看当前编辑磁盘的详情：
    ```bash
    (parted) print

    Model: XXXXXXXXX (nvme)
    Disk /dev/nvme1n1: 512GB
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    Disk Flags:

    Number  Start   End    Size   File system  Name   Flags
     1      1049kB  164GB  164GB  ntfs         Maple  msftdata
     2      164GB   512GB  348GB  btrfs
    ```

- 使用命令mklabel新建一个分区表（这将覆盖该磁盘现有分区表！）：
    ```bash { group="new disk label", name="GPT" }
    (parted) mklabel gpt
    ```
    
    ```bash { group="new disk label", name="DOS(MBR)" }
    (parted) mklabel msdos
    ```

- 使用命令mkpart新建分区：
    ```bash { group="new boot partition", name="ESP(GPT)" }
    # 新建一个名为ESP的主分区，文件系统为FAT32，大小为300MiB
    (parted) mkpart primary ESP fat32 0% 300MiB
    # 新建一个名为Root的主分区，文件系统为Btrfs，大小为磁盘所有剩余空间
    (parted) mkpart primary Root btrfs 300MiB 100%
    ```
    
    ```bash { group="new boot partition", name="MBR" }
    # 新建一个主分区，文件系统为ext4，大小为1GiB
    (parted) mkpart primary ext4 0% 1GiB
    # 新建一个主分区， 文件系统为ext4，大小为磁盘所有剩余空间
    (parted) mkpart primary ext4 1GiB 100%
    ```
    > [!tip]- 如果需要交换分区
    > 例：
    > ````bash
    > (parted) mkpart primary ESP fat32 0% 300MiB
    > # 一个大小为32GiB的交换分区
    > (parted) mkpart primary Swap linux-swap 300MiB 32.3GiB
    > (parted) mkpart primary Root btrfs 32.3GiB 100%
    > ````

- 使用命令set修改分区标记：
    > [!note]~
    > 建议先使用print查看具体分区编号
    ```bash { group="partition flags", name="ESP(GPT)" }
    # 给ESP分区打上标记
    (parted) set 1 esp on
    ```
    
    ```bash { group="partition flags", name="MBR" }
    # 给/boot分区打上标记
    (parted) set 1 boot on
    ```
    > [!tip]- 如果使用交换分区
    > ````bash
    > # 给交换分区打上标记
    > (parted) set 2 swap on
    > ````

- 使用命令print确认修改后的分区表，然后输入quit回车退出

#### 格式化分区
此时分区表已经写入磁盘，但分区尚未格式化，通过mkfs命令来格式化分区：
```bash
# 将EFI分区格式化为fat32
mkfs.fat -F 32 /dev/nvme0n1p1
# 将根分区格式化为btrfs
mkfs.btrfs /dev/nvme0n1p2
```
> 如果你使用其他文件系统，请使用对应的mkfs命令
>
> Swap分区使用mkswap命令来创建

#### 创建Btrfs子卷（非Btrfs请跳转至[挂载分区](#挂载分区)）
Btrfs文件系统支持将一个Btrfs分区分为多个子卷，为了创建子卷，首先临时挂载Btrfs分区：
```bash
# 新建挂载点
mkdir -p /mnt/gentoo
# 将btrfs根分区挂载
mount /dev/nvme0n1p2 /mnt/gentoo
```

创建Btrfs子卷需要btrfs CLI工具，Gentoo安装映像已包含，其余安装环境请自行安装：
```bash
# /@子卷
btrfs subvolume create /mnt/gentoo/@
# /@home子卷
btrfs subvolume create /mnt/gentoo/@home
# /@var子卷
btrfs subvolume create /mnt/gentoo/@var
```

现在子卷已经创建完毕，卸载Btrfs分区：
```bash
umount /mnt/gentoo
```

#### 挂载分区
一般来说，使用mount挂载磁盘只需要指定挂载点和磁盘分区即可，mount会自己处理分区文件系统类型和默认挂载参数，例如，一个简单挂载ext4根分区用于Chroot的例子：
```bash
# 首先创建挂载点（若没有）
mkdir -p /mnt/gentoo

# 简单地将挂载/dev/sda1到/mnt/gentoo
mount /dev/sda1 /mnt/gentoo
```
> [!note]- 关于/mnt
> 某些发行版的Chroot安装可能会建议直接将分区挂载到`/mnt`，虽然这也是没问题的（只要`/mnt`是空目录），但通常来说，我们常常使用它作为挂载点的父目录来管理挂载点，因此更建议使用`/mnt/gentoo`

为了使用Btrfs子卷，需要使用`-t`和`-o`参数来分别指定分区类型和挂载参数（内核对某些文件系统提供了它们独有的特殊参数）：
```bash
# 挂载/dev/nvme0n1p2的子卷/@到/mnt/gentoo
mount -t btrfs -o defaults,subvol=/@ /dev/nvme0n1p2 /mnt/gentoo
```

下面是我个人的挂载方案，它包含一些对我个人使用场景的优化：
```bash
# /@到/mnt/gentoo
mount -t btrfs -o defaults,noatime,compress=zstd:1,subvol=/@ /dev/nvme0n1p2 /mnt/gentoo

# /@home到/mnt/gentoo/home
mkdir -p /mnt/gentoo/home
mount -t btrfs -o defaults,noatime,compress=zstd:1,subvol=/@home /dev/nvme0n1p2 /mnt/gentoo/home

# /@var到/mnt/gentoo/var
mkdir -p /mnt/gentoo/var
mount -t btrfs -o defaults,noatime,compress=zstd:1,subvol=/@var /dev/nvme0n1p2 /mnt/gentoo/var
```
|参数|说明|
|-|-|
|defaults|添加默认参数|
|noatime|禁用文件的访问时间写入|
|compress=zstd:1|使用zstd算法压缩文件系统，压缩等级为1（若并不是很在意CPU负载/速度，可以自行提高压缩率。范围1~15）|
|subvol|指定一个子卷挂载|

如果使用交换分区，用swapon命令激活它，例如：
```bash
swapon /dev/sda3
```

其余的EFI或者/boot分区，将在部署stage3之后再挂载

> [!tip]~
> 可使用命令`findmnt`检查挂载情况

### 部署基本系统-stage3
首先切换到`/mnt/gentoo`：
```bash
cd /mnt/gentoo
```

#### 获取stage3
Gentoo安装映像提供了`links`用于浏览网页，使用它打开镜像列表：
```bash
links https://www.gentoo.org/downloads/mirrors

# 或者能直连官方源的话
links https://distfiles.gentoo.org
```
使用键盘的↑/↓/←/→来导航，回车键打开超链接，选择合适的镜像后，导航到`releases/设备架构/autobuilds/`，然后选择心仪的stage3-\*.tar.xz，（关于stage3的说明，请跳转至文章开头的[Stage3与Profile](#stage3与profile)），回车键后将打开下载对话框，文件会保存在当前目录
> [!note]~
> 本文使用`/releases/amd64/autobuilds/current-stage3-amd64-systemd/`下的stage3

#### 验证stage3可信性
Gentoo官方映像附带了gentoo-release.asc，使用gpg命令导入：
```bash
gpg --import /usr/share/openpgp-keys/gentoo-release.asc

# 或者通过网络keyserver导入
gpg --keyserver hkps://keys.gentoo.org --recv-keys <key fingerprint>
```

然后使用与stage3同级目录下的同名.asc文件验证签名：
```bash
gpg --verify stage3-amd64-systemd-<date>.tar.xz.asc stage3-amd64-systemd-<date>.tar.xz

# 输出示例
gpg: Signature made Mon Oct 26 04:21:04 2026 CST
gpg:                using RSA key 534E4209AB49EEE1C19D96162C44695DB9F6043D
gpg: Good signature from "Gentoo Linux Release Engineering (Automated Weekly Release Key) <releng@gentoo.org>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 13EB BDBE DE7A 1277 5DFD  B1BA BB57 2E0E 2D18 2910
     Subkey fingerprint: 534E 4209 AB49 EEE1 C19D  9616 2C44 695D B9F6 043D
```
如果输出中带有Good signature且密钥指纹与[Gentoo官方公钥列表](https://www.gentoo.org/downloads/signatures)一致，则表此文件由Gentoo官方私钥签名

> [!info]- 需要返回？
> 在此返回[准备阶段](#准备阶段)

#### 解压stage3
```bash
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner -C /mnt/gentoo
```
此命令解压(x)文件(f)stage3-\*.tar.xz(\*是通配符)，且保留权限(p)、所有命名空间存储的扩展属性与用户组及ID，并指定解压目录为`/mnt/gentoo`

### 配置基础系统
通过解压stage3，此时已将一个可用Chroot系统部署在根分区中，在Chroot前，首先配置`make.conf`

#### make.conf
`make.conf`是全局的Portage配置文件，位于`/etc/portage/make.conf`，它控制着Portage的行为，它的配置适用于所有软件包，一个`make.conf`通常包含：编译器参数（flags）、全局接受的协议、全局接受的关键词、全局USE flag、make参数、emerge参数、以及其他Portage变量；通过合理的配置，能够有效地控制系统

> [!tip]- 不要盲目乱抄配置
> ***配置没有最好的，只有最适合的***
>
> 在这里，需要说明一点，`make.conf`的配置项不是越多越好，有些你拿不准的项还不如保持默认值，例如我的`make.conf`加上注释和空行也就30行，而我的配置原则是：需求浮现->评估是否需要作为全局配置->写入配置

而除了全局配置，Gentoo还有为单个包配置USE、协议等设置的渠道，例如`/etc/portage/package.use`目录/文件用于给单个包配置USE，这些位置既可以是一个集合了所有包的独立配置的文件，也可是一个包含多个独立配置文件的目录[^3]

通过nano来编辑`make.conf`：
```bash
nano /mnt/gentoo/etc/portage/make.conf
```
关于`make.conf`的详细配置，请查看[Gentoo Wiki](https://wiki.gentoo.org/wiki/Make.conf)

下面是我个人的`make.conf`片段：
```conf
# These settings were set by the catalyst build script that automatically
# built this stage.
# Please consult /usr/share/portage/config/make.conf.example for a more
# detailed example.
COMMON_FLAGS="-march=native -O2 -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"

RUSTFLAGS="${RUSTFLAGS} -C target-cpu=native"

# NOTE: This stage was built with the bindist USE flag enabled

# This sets the language of build output to English.
# Please keep this setting intact when reporting bugs.
LC_MESSAGES=C.UTF-8

ACCEPT_LICENSE="* -@EULA"
USE="X wayland vulkan opengl policykit dbus udisks networkmanager wifi bluetooth alsa pulseaudio pipewire -doc -examples"

MAKEOPTS="-j13 -l13"
```
这份`make.conf`是我个人使用环境的产物，它不一定适合所有人，下面是对它的简单说明：
- 为一些语言的编译器启用了`native`，即针对本机CPU的优化
- 让emerge使用C.UTF-8（英文）本地化输出，防止在virtual console下出现字符渲染错误
- 接受除EULA（最终用户协议）以外的所有协议
- 启用所有图形平台/协议的USE、启用policykit、dbus等桌面组件USE、启用网络、蓝牙、音频服务USE、禁用doc、examples USE以避免不必要的文件落盘
- make使用13并发，系统平均负载阈值为13，我的CPU是12线程，此配置能够保证即使是在编译web浏览器内核的同时，多数软件仍然可以正常工作
    + 强烈不建议照抄我的并发配置，请根据自己的CPU线程数与RAM增减！！

#### Chroot
Chroot，顾名思义，其作用就是改变root（根目录），通过重新挂载与绑定文件系统的路径，使我们得以借助当前内核运行时来交互刚刚部署的、还没有可引导内核的基本系统


不过在此之前，为保证切换到新环境后DNS能够正常解析，先复制当前DNS到Chroot环境里：
```bash
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
```
> [!tip]~
> 使用dereference参数以保证无论如何都复制文件本体而不是指向它的符号链接

##### arch-chroot

如果使用Gentoo官方安装映像，可以通过`arch-chroot`代替手动变更文件系统：
```bash
arch-chroot /mnt/gentoo
# 在命令提示符中加上(chroot)来提醒：现在位于Chroot环境
# 这是可选的，因为只是个视觉效果
export PS1="(chroot) ${PS1}"
```

##### 手动Chroot

手动变更文件系统的方式如下所示：
```bash
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run
```
上面的命令将运行时文件系统（`/proc`、`/sys`、`/run`）以及包含所有设备的`/dev`重新挂载到新的根目录
> [!tip]~
> make-rslave对于systemd支持是必需的
>> [!note]- 关于挂载本身的特性[^4]
>> 对于内核来说，一个挂载可以具有以下几种特性之一：
>>  - shared：使镜像内的挂载和卸载操作会传播到其他镜像，在Chroot中，相当于宿主与Chroot的双向传播
>>  - slave：接收来自其主挂载的传播，但反之则不然，在Chroot中，相当于宿主到Chroot的单向传播，在本例中，它掐断了Chroot内systemd影响宿主挂载的可能
>>  - private：不携带任何传播能力
>>  - unbindable：一种无法通过绑定操作克隆的private挂载
>>
>> PS：前面加个r是遍历更改挂载点下所有挂载的意思

> [!warning]+ 关于非Gentoo安装映像环境
> 据Gentoo Wiki所言[^5]，某些发行版将`/dev/shm`作为指向`/run/shm`的符号链接，在此情况下，我们需要在Chroot前重建`/dev/shm`：
> ````bash
> # 重建/dev/shm
> test -L /dev/shm && rm /dev/shm && mkdir /dev/shm
> # 重新挂载tmpfs
> mount --types tmpfs --options nosuid,nodev,noexec shm /dev/shm
> # 设置正确的权限
> chmod 1777 /dev/shm /run/shm
> ````
> 对于此，我并未实验过，仅转述Wiki内容

接下来使用chroot命令变更根目录：
```bash
chroot /mnt/gentoo /bin/bash
# 进入新环境后，首先更新环境变量
source /etc/profile
# 可选的(chroot)标识
export PS1="(chroot) ${PS1}"
```

此时，我们第一次成功地进入了刚刚部署的基本系统，不过在进一步配置系统之前，先把引导需要的分区给挂上：
```bash { group="mount boot partition", name="UEFI" }
# 创建挂载点（若没有）
mkdir /efi
mount /dev/nvme0n1p1 /efi
```

```bash { group="mount boot partition", name="Legacy BIOS" }
mount /dev/sda2 /boot
```
> [!note]~
> 因为已经通过Chroot改变了根目录，所以往后直接使用`/`操作Chroot环境的根目录

#### 配置Portage
首先，无论如何，先从源码镜像把官方仓库快照以及附带的配置文件拉下来，可以有效避免许多同步问题：
```bash
emerge-webrsync
```
emerge可能会打印一堆Warning或者其他建议（例如阅读news、升级软件包等），只要没报错就暂时先不管，先来配置镜像，然后再决定是否升级

##### Distfiles仓库镜像
Gentoo官方提供了名为`mirrorselect`的软件包，用于从官方Distfiles镜像列表中选择镜像，它也很有可能成为你通过Portage获取的第一个软件包：
```bash
emerge -av1 app-portage/mirrorselect
```
> [!tip]- emerge技巧
> 上面的命令是通过emerge临时安装软件包的推荐方式，使用这种方式安装的软件包不会被Portage记录在world集合中，命令参数说明：
>   - `-a`：长格式为`--ask`，告诉emerge不要轻举妄动，任何修改操作都要先问过用户是否执行
>   - `-v`：长格式为`--verbose`，让emerge打印详细输出
>   - `-1`: 长格式为`--oneshot`，告诉emerge别把这个软件包加到world集合里

现在选择一个镜像源添加到`make.conf`：
```bash
mirrorselect -i -o >> /etc/portage/make.conf
```
该命令通过`-o`参数使被选择的镜像输出为`GENTOO_MIRRORS="<mirror>"`，然后通过`>>`告诉shell将输出追加进`make.conf`而不是打印出来
> [!danger] 注意区分`>`与`>>`！
> 不要把`>>`写成`>`，它们两个的行为截然不同：`>>`会将输出追加到文件中，而`>`将覆盖文件的原始内容！

> [!tip]- 我想要的镜像不在官方列表里
> 没关系，完全可以手动编辑`make.conf`达到同样的目的：
> ````bash
> nano /etc/portage/make.conf
> ````
> 
> 然后在`make.conf`中添加：
> ````conf
> GENTOO_MIRRORS="<link of your mirror>"
> # 可选：官方源兜底，如果没有的话依旧会FallBack到Ebuild的SRC_URI，因此可选
> GENTOO_MIRRORS="${GENTOO_MIRRORS} https://distfiles.gentoo.org"
> ````

##### ebuild仓库镜像
用于下载Distfiles的镜像已经配置完毕，但同步镜像依旧指向Gentoo官方仓库，在`/etc/portage/repos.conf`目录下创建`.conf`文件来配置ebuild仓库，不过不要白手起家，系统里有现成的模板：
```bash
# 创建/etc/portage/repos.conf
mkdir /etc/portage/repos.conf
# 复制系统配置
cp /usr/share/portage/config/repos.conf /etc/portage/repos.conf/gentoo.conf
# 编辑gentoo.conf
nano /etc/portage/repos.conf/gentoo.conf
```

在继续修改`gentoo.conf`之前，需要先了解Portage的同步协议。当前，Portage支持多种同步协议（rsync、git、cvs等等），其中常用的是`rsync`和`git`，在这里简单介绍二者区别

###### rsync
rsync是一种高性能的、支持远程位置的文件复制工具，Portage为其组装了一系列流程，以确保其在ebuild仓库场景下工作正常，当同步时：
1. Portage会首先通过rsync获取时间戳文件并与本地时间戳文件对比，如果远端时间戳比本地新，或者任一时间戳无效/缺失，Portage才会开始真正的同步（以文件为粒度）[^6]
2. 如果启用了Manifest验证，Portage会调用gemato[^7]验证：
    1. 下载的根Manifest
    2. 下载的根Manifest的OpenPGP签名
    3. 下载的根Manifest内的时间戳
    4. 若配置了最大年龄则验证同步的时间戳是否过期
    5. 如果相较于本地仓库发生内容变更，递归验证整仓Manifest哈希
3. 一切成功的话，才会合并变更到本地仓库

个人使用体验：相比git同步来说要慢一些，但是它能确保仓库的所有Manifest哈希与根Manifest记录一致，任何不匹配都会拒绝同步

###### git
git是一种广泛使用的分布式版本控制系统，同时也是其CLI工具的名称，Portage基于git自身的特性为其制订了一系列同步细则和处理，以尽可能地确保它的使用对于用户来说足够直观，由于git的版本控制较为复杂，这里仅介绍大概流程：
1. Portage首先会通过git(clone/fetch)获取最新提交（默认depth=1，可配置）
2. 若配置了最大年龄则验证远端分支的`timestamp.chk`是否过期，未过期则继续
3. 若启用了提交签名验证，Portage会验证远端分支HEAD的OpenPGP签名是否有效（自定义密钥需要gemato）
4. 若一切成功，Portage会依据`volatile`设置与本地仓库的状态来选择是否执行破坏性合并或其他合并操作
5. 比较同步前后的HEAD，判断这次同步是否实际产生了新提交

个人使用体验：git的同步更快，但是它不会去验证仓库Manifest哈希，因为它的信任模型不依赖gentoo自己建立的Manifest信任网络，在git的信任模型中，git本身用哈希把这些对象串起来。如果HEAD被有效签名，那么理论上它覆盖了这个commit指向的整个树

> [!note]- Portage不会改变仓库分支
> "Update existing git repository, and ignore the syncuri. We are going to trust the user and assume that the user is in the branch that he/she wants updated. We'll let the user manage branches with git directly."[^8]
>
> 用户如果想管理分支，应自己用git管理

###### repos.conf
官方建议的Gentoo仓库标准配置如下所示：
```conf
[DEFAULT]
main-repo = gentoo

[gentoo]
location = /var/db/repos/gentoo
sync-type = rsync
sync-uri = rsync://rsync.gentoo.org/gentoo-portage
auto-sync = yes
sync-rsync-verify-jobs = 1
sync-rsync-verify-metamanifest = yes
sync-rsync-verify-max-age = 3
sync-openpgp-key-package = sec-keys/openpgp-keys-gentoo-release
sync-openpgp-key-path = /usr/share/openpgp-keys/gentoo-release.asc
sync-openpgp-keyserver = hkps://keys.gentoo.org
sync-openpgp-key-refresh-retry-count = 40
sync-openpgp-key-refresh-retry-overall-timeout = 1200
sync-openpgp-key-refresh-retry-delay-exp-base = 2
sync-openpgp-key-refresh-retry-delay-max = 60
sync-openpgp-key-refresh-retry-delay-mult = 4
sync-webrsync-verify-signature = yes
sync-git-verify-commit-signature = true
```
这里简单说其中主要的项[^3]：
- location：仓库在本地的位置，一般为`/var/db/repos/<name>`
- sync-type：使用哪种同步模块，建议rsync/git
- sync-uri：远程地址，必须为sync-type支持的链接
    + 对于Gentoo主仓库的rsync镜像，使用如下命令：
        > [!note]~
        > 首先移除旧的sync-uri！！
        ```bash
        mirrorselect -i -r -o >> /etc/portage/repos.conf/gentoo.conf
        ```
    + 对于官方的Git同步镜像，请尝试Github仓库：[gentoo-mirror/gentoo](https://github.com/gentoo-mirror/gentoo.git)
        + 不是很推荐官方的[自有Git托管](https://anongit.gentoo.org/git/repo/gentoo.git)，因为它能不能连上看心情，同时，它是开发仓库，而非公共镜像，因此没有Gentoo Release Key签名
        + 同样的，Github仓库：[gentoo/gentoo](https://github.com/gentoo/gentoo)是上面提到的开发仓库的镜像，它也没有Gentoo Release Key密钥签名
    + 对于中国大陆地区用户，可在[MirrorZ-校园网联合镜像站](https://help.mirrors.cernet.edu.cn/)寻找可用的rsync/git镜像
- sync-rsync-verify-metamanifest：启用此项，则在rsync同步中执行Manifest验证
- sync-openpgp-key-package：含OpenPGP Key的Portage软件包，`sec-keys/openpgp-keys-gentoo-release`是Gentoo官方的发布公钥
- sync-openpgp-key-path：OpenPGP公钥路径，`/usr/share/openpgp-keys/gentoo-release.asc`是上面提到的Gentoo发布密钥的默认安装路径
- sync-openpgp-keyserver：用于下载公钥的服务器
- sync-git-verify-commit-signature：启用此项，则在git同步中执行HEAD签名验证
- sync-webrsync-verify-signature：启用此项，则在webrsync中执行签名验证

> [!tip]- 藏在手册和代码里的默认值
> - 手册[^3]里的默认值：
>   + 在Portage手册中可以得知，几乎所有选项的默认值都是关或者无，例如：如果不显式启用验证，Portage不会有任何可信性验证行为，Portage将只负责把文件完整拉下来合并
>
> - 代码里的默认值：
>   + Portage在构建最终配置时，会将系统配置与用户配置合并，对于Gentoo主仓库，上文复制的就是它的系统配置，因此只有用户在`/etc/portage/repos.conf`显式覆盖主仓库的某项，最终配置才会较系统配置发生改变

###### 首次同步（若更换了镜像）
在更换新的镜像后，为避免出现同步问题（尤其是更换了sync-type后），建议通过如下命令更新本地仓库：
```bash { group="sync repo", name="仅换源" }
# 重新同步Gentoo主仓库
emaint sync -r gentoo
```
```bash { group="sync repo", name="变更sync-type" .active }
# 获取eselect的repository模块
emerge -av app-eselect/eselect-repository
# 备份修改后的配置文件
cp /etc/portage/repos.conf/gentoo.conf ~/gentoo.conf
# 移除现有仓库
eselect repository remove -f gentoo
# 恢复配置文件
mv ~/gentoo.conf /etc/portage/repos.conf/gentoo.conf
# 使用新的协议同步
emaint sync -r gentoo
```

> [!question]- 从其他sync-type换回rsync时，Manifest验证失败怎么办？
> 由于在启用验证后，rsync会尝试验证Manifest，但当本地仓库被Remove时，rsync会报错找不到Manifest：
> ````bash
> * Verifying /var/db/repos/gentoo/.tmp-unverified-download-quarantine ...!!! Manifest verification failed:
> Manifest mismatch for metadata/news/Manifest
> BLAKE2B: expected: 1635706d5f00bee348207548ed3d286a33bf3da3266f0cdad1cb7a1a02d7fa071225ff13957d5cd4469156551c573a694299703a901605450722d388220fa112, have: 4412a3ffde958dd3bf69e2e15830dd5ba3b2b554fde5d7f02e31bab78f01cb82833a4ef4ece2ef500b87860528a46faf5bec65c3a0b60679f013c57e9aeb1ca4
> SHA512: expected: c70b73ea9a54985cfb71e5932a3d230d7d3e7059d928669d5bf4a93841a6b558db7f13432889d4a0d281372b7d68bc15ed8a697f9929546b67c6a2c41a8b46f6, have: e6af70f9c5254ae1bbd560e5e7253b60beb35e67e437ec91b46aeb56fb79c6bf8dcb8874715b2afdea2299e7488ec3d59ab03a69320e7e950d34dd557d6789b4
> [!!]
> ````
> 为了避免这一问题，首先使用webrsync，它会从GENTOO_MIRRORS的Gentoo镜像获取一份官方的仓库快照：
> ````bash 
> eselect repository remove -f gentoo
> # 恢复配置文件
> mv ~/gentoo.conf /etc/portage/repos.conf/gentoo.conf
> # 从Gentoo源码镜像拉一份快照
> emerge-webrsync
> ````
> 随后再增量同步：
> ````bash
> emaint sync -r gentoo
> ````

##### binhost镜像（可选）
当前，stage3在构建时默认启用了binhost，并且生成了`/etc/portage/binrepos.conf/gentoo.conf`，直接编辑它：
```bash
nano /etc/portage/binrepos.conf/gentoo.conf
```

修改sync-uri：
```bash
# These settings were set by the catalyst build script that automatically
# built this stage.
# Please consider using a local mirror.

[gentoo]
# 优先级，按需
priority = 1
sync-uri = <your binhost mirror>
location = /var/cache/binhost/gentoo
verify-signature = true
```

> [!tip]- 如何安装预编译的二进制包
> ````bash
> emerge -avg <package>
> ````
> 使用`-g(--getbinpkg)`参数，emerge会尝试获取二进制包（如果存在）

#### 切换Profile（可选且不建议）
> [!danger] 慎重！
> 强 烈 不 建 议
>
> 比起现在吃后悔药，不如在十几分钟前选择stage3时慎重

值此之时，stage3刚解压、还未安装新的软件包，相对而言是一个绝佳的“后悔药”节点，可以通过eselect切换到其他Profile：
```bash { class="is-expanded" }
eselect profile list

Available profile symlink targets:
  [1]   default/linux/amd64/23.0 (stable)
  [2]   default/linux/amd64/23.0/systemd (stable) * # 当前使用此Profile
  [3]   default/linux/amd64/23.0/desktop (stable)
  [4]   default/linux/amd64/23.0/desktop/systemd (stable)
  [5]   default/linux/amd64/23.0/desktop/gnome (stable)
  [6]   default/linux/amd64/23.0/desktop/gnome/systemd (stable)
  [7]   default/linux/amd64/23.0/desktop/plasma (stable)
  [8]   default/linux/amd64/23.0/desktop/plasma/systemd (stable)
  [9]   default/linux/amd64/23.0/no-multilib (stable)
  ......

# 切换至default/linux/amd64/23.0/desktop/systemd
eselect profile set 4
```

#### 硬件flags
Portage的一些包含有硬件相关的USE flag，手动编辑CPU flag不太方便，使用`cpuid2cpuflags`生成它：
```bash
# 安装工具包
emerge -av1 app-portage/cpuid2cpuflags
# 生成CPU flags
echo "$(cpuid2cpuflags)" > /etc/portage/package.use/00cpu-flags
```
结果示例：
```bash
cat /etc/portage/package.use/00cpu-flags

*/* CPU_FLAGS_X86: aes avx avx2 avx512_bitalg ......
```
其中`*/*`是通配符，表示此USE配置匹配任意包（即对所有包生效）


接下来编辑显卡USE flag（如果有）：
```bash
nano /etc/portage/package.use/00video_cards
```
示例：
```
*/* VIDEO_CARDS: -* <GPU DRIVER NAME>
```
其中`*/*`作用已经解释过了，另一个通配符`-*`的含义是：减去所有GPU驱动（清空VIDEO_CARDS:的值），以便只包含后面追加的GPU驱动

这里的\<GPU DRIVER NAME\>需要正确填写GPU驱动名称，而不是GPU型号，下面是常见的GPU驱动：
|GPU DRIVER NAME|说明|
|-|-|
|nvidia|闭源NVIDIA驱动|
|nouveau|开源NVIDIA驱动|
|amdgpu radeonsi|AMD GPU驱动|
|intel|Intel GPU驱动|

#### 必备知识：USE与Autounmask
有些时候，emerge可能会给出类似这样的输出：
```bash
# required by media-gfx/gimp-3.2.4::gentoo
# required by gimp (argument)
>=app-text/poppler-26.09.0 cairo
# required by media-gfx/gimp-3.2.4::gentoo
# required by gimp (argument)
>=media-libs/babl-0.1.128 lcms
# required by media-gfx/gimp-3.2.4::gentoo
# required by gimp (argument)
>=media-libs/gegl-0.4.70 cairo lcms
# required by x11-libs/pango-1.57.1::gentoo
# required by dev-libs/appstream-glib-0.8.3::gentoo
# required by dev-libs/appstream-1.0.6::gentoo
# required by media-gfx/gimp-3.2.4::gentoo
# required by gimp (argument)
>=media-libs/freetype-2.14.3 harfbuzz
# required by x11-libs/gtk+-3.24.52::gentoo
# required by x11-themes/adwaita-icon-theme-legacy-46.2::gentoo
# required by x11-themes/adwaita-icon-theme-49.0::gentoo
>=x11-libs/cairo-1.18.4-r1 X

Would you like to add these changes to your config files? [Yes/No]
```

此输出是由于某些包需要启用这些包的USE flag，emerge会自动生成一份待写入的patch，为了应用它需要使用dispatch-conf工具：
```bash
dispatch-conf
```

该工具会检测所有待写入的配置，然后打开diff工具：
```bash
--- /etc/portage/package.use/tests      2026-11-05 21:30:06.769988963 +0800
+++ /etc/portage/package.use/._cfg0000_tests    2026-11-08 21:13:08.189672665 +0800
@@ -22,3 +22,22 @@
 # required by kde-frameworks/knotifications-6.29.0::gentoo
 # required by kde-frameworks/knotifications (argument)
 >=app-text/xmlto-0.0.28-r11 text
+# required by media-gfx/gimp-3.2.4::gentoo
+# required by gimp (argument)
+>=app-text/poppler-26.09.0 cairo
+# required by media-gfx/gimp-3.2.4::gentoo
+# required by gimp (argument)
+>=media-libs/babl-0.1.128 lcms
+# required by media-gfx/gimp-3.2.4::gentoo
+# required by gimp (argument)
+>=media-libs/gegl-0.4.70 cairo lcms
+# required by x11-libs/pango-1.57.1::gentoo
+# required by dev-libs/appstream-glib-0.8.3::gentoo
+# required by dev-libs/appstream-1.0.6::gentoo
+# required by media-gfx/gimp-3.2.4::gentoo
+# required by gimp (argument)
+>=media-libs/freetype-2.14.3 harfbuzz
+# required by x11-libs/gtk+-3.24.52::gentoo
+# required by x11-themes/adwaita-icon-theme-legacy-46.2::gentoo
+# required by x11-themes/adwaita-icon-theme-49.0::gentoo
+>=x11-libs/cairo-1.18.4-r1 X

>> (1 of 1) -- /etc/portage/package.use/tests
>> q quit, h help, n next, e edit-new, z zap-new, u use-new
   m merge, t toggle-merge, l look-merge:
```
按下u将会把新内容写入文件，按下z丢弃新内容

> [!question]+ emerge Autounmask污染了我的配置，怎么办？
> 对于一个配置目录，程序通常采用优先级读取，例如前面将CPU flags文件命名为`00`开头，便是为了让Portage首先读取它，而emerge在autounmask时也遵循类似的规则，不过是反过来的：**autounmask会优先写入排序位于末尾的文件**，因此，可以给它准备一个专用的文件来达到防止污染配置的效果：
> ````bash
> touch /etc/portage/package.use/zzzauto
> ````
> 丢弃旧的patch再次让emerge生成新的patch
> ````bash
> Would you like to add these changes to your config files? [Yes/No] y
>
> Autounmask changes successfully written.
> * IMPORTANT: config file '/etc/portage/package.use/zzzauto' needs updating.
> * See the CONFIGURATION FILES and CONFIGURATION FILES UPDATE TOOLS
> * sections of the emerge man page to learn how to update config files.
> ````
> 可以看到emerge将patch的目标设置为了排序末尾的zzzauto

#### 更新@world集合（可选）
> [!warning]~
> 如果你前面改了Profile，这可能会花费更长时间（视选择而定）

```bash
emerge -auvDU @world
```
命令拆解：
- `-a`：长格式为`--ask`，告诉emerge不要轻举妄动，任何修改操作都要先问过用户是否执行
- `-u`: 长格式为`--update`，告诉emerge更新软件包
- `-v`: 长格式为`--verbose`，让emerge打印详细输出
- `-D`: 长格式为`--deep`，让emerge深度检索依赖树
- `-U`：长格式为`--changed-use`，让emerge检索发生USE变更的包
- `@world`：指定@world集合

>[!note]- \-\-changed-use VS \-\-newuse[^9]
> 二者均会触发USE变更的重建，区别在于：`--changed-use`会忽略非用户启用的USE增减（通常是ebuild内IUSE变更），而`--newuse`则会为非用户启用的USE增减触发重装

##### 建议：在@world更新后清理不需要的包
在更新@world后，某些包可能不再被依赖，通过`--depclean`参数可以让emerge自己分析并移除不再需要的包：
```bash
emerge -av --depclean
```

#### 本地化

##### 时区
Linux将不同时区的配置文件储存在`/usr/share/zoneinfo`下，如果需要，可以采用如下方式查看所有时区：
```bash
ls -R /usr/share/zoneinfo | less
```
> 在less中，使用`/`来检索

设置时区只需要将对应的配置文件链接到`/etc/localtime`即可：
```bash
ln -sf ../usr/share/zoneinfo/<region>/<city> /etc/localtime
```
> [!tip]- 关于../的含义
> 很多发行版的教程使用绝对路径链接时区文件，虽然这也是没问题的，但使用`../`开头的相对路径在Chroot中是更稳健的做法，该链接相当于：从`/etc`出发，`..`回到`/`，再进入`usr/share/...`


##### 语言
首先编辑`/etc/locale.gen`指定要生成的语言文件：
```bash
nano /etc/locale.gen
```

Gentoo`sys-apps/locale-gen`配置语法[^10]：
```conf
# 这三种格式被视为相同
en_US.UTF-8 UTF-8 # 显示指定UTF-8
en_US UTF-8 # 省略了冗余的字符映射
en_US # 不指定，则默认为UTF-8

# 使用其他字符集（兼容性未知，十分不建议）
en_US ISO-8859-1

# 不受支持的字符集映射是无效的
en_US GB2312
```

省流配置：
```conf
en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8
```

配置完成后，生成语言文件：
```bash
locale-gen

Found 2 locale declarations in '/etc/locale.gen'.
Compiling 3 locales with 3 workers ...
[1/3] Compiling locale: C.UTF-8
[2/3] Compiling locale: en_US.UTF-8
[3/3] Compiling locale: zh_CN.UTF-8
Waiting for active workers to finish their jobs ...
The location of the archive shall be '/usr/lib/locale/locale-archive'.
Adding 3 locales to the locale archive ...
Successfully installed an archive containing 3 locales, of 5.96 MiB in size
```

使用eselect模块切换语言环境变量：
```bash { class="is-expanded" }
eselect locale list
Available targets for the LANG variable:
  [1]   C
  [2]   C.UTF-8 *
  [3]   POSIX
  [4]   en_US.UTF-8
  [5]   zh_CN.UTF-8
  [ ]   (free form)

# 切换至en_US.UTF-8
eselect locale set 4
```
eselect会在`/etc/env.d/`下生成环境变量配置文件
> [!bug]~
> 不建议将系统语言配置为任何非英文字符集语言！Linux内核的virtual console对与字符渲染的支持有限，不受支持的字符将被渲染为豆腐块！（除非给内核的编译打补丁）

##### 更新环境变量
```bash
# 更新环境变量，并补上(chroot)标记
env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
```

### 安装固件驱动与内核

#### 固件

##### 通用固件包
Linux提供了一个包含多数设备固件的包（多数但不是所有）：
```bash
emerge -av sys-kernel/linux-firmware
```

##### 音频固件包
一些较新的CPU使用Sound Open Firmware驱动（具体列表参见[SOF Project](https://thesofproject.github.io/latest/platforms/index.html)）：
```bash
emerge -av sys-firmware/sof-firmware
```

##### 微码
微码是用于CPU的固件，AMD CPU的微码包含在`sys-kernel/linux-firmware`中，无需额外安装；Intel CPU需要安装如下包：
```bash
emerge -av sys-firmware/intel-microcode
```

#### sys-kernel/installkernel
通过sys-kernel/installkernel的USE flag，可以自动化一些内核安装/更新时的操作，在`/etc/portage/package.use`下新建配置文件以单独配置sys-kernel/installkernel的USE：
```bash
nano /etc/portage/package.use/installkernel
```

- 要使内核安装后时自动执行`grub-mkconfig`配置引导，启用grub USE flag：
    ```conf
    sys-kernel/installkernel grub
    ```

- 要使内核安装时自动执行`dracut`生成Initramfs，启用dracut USE flag:
    ```conf
    sys-kernel/installkernel dracut
    ```
    >[!note]+ 关于initramfs
    > initramfs是为了避免将大量驱动程序硬编码到内核导致内核过大、用于启动时推断根文件系统位置等问题而诞生的；它是一个临时根文件系统，在系统启动时加载，用于加载内核模块、挂载根文件系统和查找设备等
    >
    > 而dracut是诞生于Fedora Project的工具，用于生成initramfs，启用此USE，installkernel会自动调用dracut并将生成的initramfs置于/boot目录


安装sys-kernel/installkernel包（如果启用了相应USE，会自动安装所需软件包）：
```bash
emerge -av sys-kernel/installkernel
```

#### 内核
> [!info]~
> 本文不提供完全控制内核编译流程的教程，未来可能会出一篇文章（挖坑ing...）

##### 启用dist-kernel USE（可选）
当使用Gentoo官方的发行版内核时，建议通过启用`dist-kernel` USE来自动化内核升级后的重建操作，编辑`/etc/portage/make.conf`，在USE中新增：
```conf
USE="${USE} dist-kernel"
```

##### 使用预编译内核包
预编译的内核包无需额外操作，直接安装即可：
```bash
emerge -av sys-kernel/gentoo-kernel-bin
```

##### 使用Gentoo维护的ebuild自动编译内核

###### 模块签名
在本地编译的内核缺少可用签名，虽然在构建时会自动生成密钥，但更建议使用一个自己生成的、统一管理的密钥：
```bash
mkdir /root/PEM
# 新建密钥
# 自行填写OpenSSL询问的项即可，虽然很多项可以为空，但还是建议至少填写用于分辨的信息
openssl req -new -noenc -utf8 -sha256 -x509 -outform PEM -out /root/PEM/kernel.pem -keyout /root/PEM/kernel_key.pem
# 确保仅Root可操作密钥
chown -R root:root /root/PEM
chmod -R 400 /root/PEM
```
该命令将创建一对openssl密钥，输出为PEM格式，公钥为`/root/PEM/kernel.pem`，私钥为`/root/PEM/kernel_key.pem`

接着在`/etc/portage/make.conf`中新增`module-sign` USE与密钥配置：
```conf
USE="${USE} module-sign"

# 私钥
MODULES_SIGN_KEY="/root/PEM/kernel_key.pem"
# 公钥
MODULES_SIGN_CERT="/root/PEM/kernel.pem"

# 签名哈希算法（可选，默认即长哈希sha512）
MODULES_SIGN_HASH="sha512"
```

###### 使用Portage自动编译内核
配置好密钥后，直接安装`sys-kernel/gentoo-kernel`即可，官方的ebuild会自动处理内核编译，这可能需要几十分钟，等待即可：
```bash
emerge -av sys-kernel/gentoo-kernel
```

##### 手动重建内核模块/initramfs
如果有需要，可以通过如下方法重建内核模块或者initramfs：
```bash
# 重建内核模块
emerge -av @module-rebuild
# 重建initramfs
emerge --config sys-kernel/gentoo-kernel
# 或者使用预编译内核
emerge --config sys-kernel/gentoo-kernel-bin
```

### 进一步配置系统
此时，系统已经有了可以引导的内核以及基本的固件、驱动，但机器的引导尚且不知道硬盘里有这号系统，更不知道该怎么引导它，接下来的工作就是配置引导、系统加载以及必须软件包

#### Fstab
Linux通过`/etc/fstab`文件管理系统启动时如何挂载磁盘分区，一个fstab文件通常列出所有可用的磁盘分区和其他类型的文件系统以及可能不是基于磁盘的数据源，并告诉它们如何初始化或以其他方式集成到更大的文件系统结构中；一般而言，`/etc/fstab`文件中每行包含6个字段，分别是：
- \<device\>：用于指明要挂载的设备或远程文件系统，可以是设备路径、UUID、分区标签（推荐使用PARTUUID）
- \<dir\>：用于指明挂载点，即 \<device\> 挂载到何处
- \<type\>：用于指明文件系统类型，eg. ext4、vfat、swap等
- \<options\>：用于指明mount命令的参数，用半角逗号分隔，如 defaults，noatime，nodiratime，不同文件系统支持的参数不同，请参考对应文件系统文档
- \<dump\>：用于指明是否使用dump备份文件系统，0表示不备份，1表示备份
- \<fsck\>：用于指明文件系统检查的顺序，0表示不检查，1表示优先检查，2表示次优先检查

不幸的是，Gentoo并没有像Arch那样的fstab生成脚本，这意味着用户需要手动编辑fstab：
- 使用`lsblk`命令可以获取所有分区的详细信息，这里提供的命令会给出fstab需要的PARTUUID等数据：
    ```bash
    lsblk -o KNAME,PARTUUID,MOUNTPOINTS,FSTYPE
    
    KNAME          PARTUUID                             MOUNTPOINTS FSTYPE
    /dev/nvme0n1p1 dc436226-d8d7-47e8-a627-d3731e2d4744 /efi        vfat
    /dev/nvme0n1p2 661d2d46-ed5f-4868-972c-8ac01ef0362f /var        btrfs
                                                        /home
                                                        /
    ```

- I know, I know. 手写UUID很麻烦，没关系，这里有个比拍照更好的办法：
    ```bash
    lsblk -o KNAME,PARTUUID,MOUNTPOINTS,FSTYPE >> /etc/fstab
    ```
  + 上面的命令格式化了lsblk的输出，并让shell直接把输出追加进`/etc/fstab`方便编辑，同时，也可以使用nano的复制粘贴命令实现复制PARTUUID

>[!tip]- 更进一步：挂载参数我也不想写
> 实际上如果在前面已经正确挂载了所有所需分区，可以通过如下命令获得一份带挂载参数的输出（注意：该输出中的defaults已被展开，建议自行替换回defaults）：
> ````bash
> findmnt -o PARTUUID,TARGET,FSTYPE,FS-OPTIONS >> /etc/fstab
> ````

>[!warning]~
> 上面的命令只是方便编辑，它不会给你可用的fstab！！

- 随后就可以动手编辑`/etc/fstab`了，一个fstab的例子（不同项直接使用空格或者制表符分割均可）：
    ```fstab
    # <device>                                        <dir>    <type>  <options>                                            <dump>  <fsck>
    PARTUUID=661d2d46-ed5f-4868-972c-8ac01ef0362f /               btrfs   defaults,noatime,compress=zstd:1,subvol=/@        0       1
    PARTUUID=661d2d46-ed5f-4868-972c-8ac01ef0362f /home           btrfs   defaults,noatime,compress=zstd:1,subvol=/@home    0       2
    PARTUUID=661d2d46-ed5f-4868-972c-8ac01ef0362f /var            btrfs   defaults,noatime,compress=zstd:1,subvol=/@var     0       2
    PARTUUID=dc436226-d8d7-47e8-a627-d3731e2d4744 /efi            vfat    umask=0077,tz=UTC                                 0       2
    ```

#### 初始化init系统
```bash { group="init system" name="Systemd" }
# 生成随机machine id
systemd-machine-id-setup
# 配置主机名、键盘布局
systemd-firstboot --prompt
# 启用默认服务（开机自启）
systemctl preset-all --preset-mode=enable-only
```

```bash { group="init system" name="OpenRC" }
# 没啥要说的，里面有极其详细的注释，按个人喜好走就行
nano /etc/rc.conf
# 键盘布局
nano /etc/conf.d/keymaps
# 时钟格式（如果硬件时钟格式不是UTC，配置clock="local"可以防止系统和它打架）
nano /etc/conf.d/hwclock
```

#### 网络管理

##### 主机名
直接往`/etc/hostname`里写一串即可：
```bash
# 例子：将主机名设为ohmypc
echo ohmypc > /etc/hostname
```

##### NetworkManager
Linux的网络管理工具很多，个人喜欢NetworkManager，自带TUI界面，首先在`/etc/portage/make.conf`添加 networkmanager USE：
```conf
# 若照抄我的make.conf，可能已经包含，请勿重复添加！
USE="${USE} networkmanager"
```
接着安装NetworkManager：
```bash
emerge -av net-misc/networkmanager
```
启用服务（开机自启）：
``` { group="NetworkManager" name="Systemd" }
systemctl enable NetworkManager
```

``` { group="NetworkManager" name="OpenRC" }
rc-update add NetworkManager default
```

#### 个人建议的工具（可选）
- chrony：时间同步工具
    ```bash
    emerge -av net-misc/chrony
    # Systemd
    systemctl enable chronyd.service
    # OpenRC
    rc-update add chronyd default
    ```

- 文件系统工具包：
|文件系统|Gentoo包|
|-|-|
|XFS|sys-fs/xfsprogs|
|ext4|sys-fs/e2fsprogs|
|FAT家族|sys-fs/dosfstools|
|Btrfs|sys-fs/btrfs-progs|
|F2FS|sys-fs/f2fs-tools|
|NTFS|sys-fs/ntfs3g|
|ZFS|sys-fs/zfs|

- udev规则：nvme硬盘调度
    ```bash
    emerge -av sys-block/io-scheduler-udev-rules
    ```

>[!tip]- 对于OpenRC用户
> 由于OpenRC自身没有log组件，需要自行安装日志工具，对于初学者和想要省事的用户，可以使用`app-admin/sysklogd`：
> ````bash
> emerge -av app-admin/sysklogd
> # 启用服务
> rc-update add sysklogd default
> ````
>>[!tip]- 对于OpenRC桌面用户
>> 在OpenRC下使用桌面环境时，建议安装`sys-auth/elogind`来管理桌面会话需要的XDG变量和运行时目录：
>> ````bash
>> emerge -av sys-auth/elogind
>> ````

#### 引导程序
Linux可用的引导程序众多，这里介绍的是通用性较好的GRUB2

##### 安装GRUB
首先安装grub：
```bash
emerge -av sys-boot/grub
```

随后运行grub的安装命令：
```bash { group="grub install" name="UEFI" }
# 针对x86_64(AMD64)平台的UEFI安装
# 使用GRUB作为引导项名称
# efi目录为/efi，即挂载的ESP分区
grub-install --target=x86_64-efi --bootloader-id=GRUB --efi-directory=/efi
```

```bash { group="grub install" name="Legacy BIOS" }
# 针对i386-pc平台的Legacy BIOS安装
# 目标硬盘应为用于引导的磁盘
grub-install --target=i386-pc /dev/sda
```

如果需要引导多系统，需要安装额外的os-prober用于其他系统的扫描：
```bash
emerge -av sys-boot/os-prober
```

##### 配置GRUB
如果需要引导多系统，首先编辑配置文件：
```bash
nano /etc/default/grub
```

增加如下条目，取消os-prober的禁用：
```
GRUB_DISABLE_OS_PROBER=false
```

使用如下命令基于配置文件生成GRUB的最终Boot配置：
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

#### 收尾工作
收尾之前，先盘算下自己是不是做了所有必要步骤，要不然启动失败不就尴尬了

- Root密码设置了吗？没有的话记得设置：
    ```bash
    passwd
    ```

- Fstab正确吗？Cat验一下：
    ```bash
    cat /etc/fstab | less
    ```

- 内核确认装上了吗？看看/boot里有没有：
    ```bash { group="kernel verify" name="Systemd" }
    ls /boot
    
    # 示例输出
    grub                            initramfs-<version>-gentoo-dist.img
    kernel-<version>-gentoo-dist
    ```
    
    ```bash { group="kernel verify" name="OpenRC" }
    ls /boot
    
    # 示例输出
    grub                            initramfs-<version>-gentoo-dist.img
    vmlinuz-<version>-gentoo-dist
    ```
    >[!note]- kernel\- VS vmlinuz\-[^11]
    > 视`SYSTEMD_KERNEL_INSTALL`变量的不同，installkernel会使用不同的布局。在Gentoo中，该变量可由`systemd` USE flag控制，该USE在Systemd的Profile中全局启用，而OpenRC的Profile则未启用，因此最终表现出的结果就是上面的效果

- GRUB确认装上了吗？如果是UEFI，可以验证GRUB的efi文件是否存在：
    ```bash
    ls -R /efi/ | less
    ```

如果确认一切就绪，那么收尾：
```bash
# 退出Chroot，返回Live环境
exit
# 回到Live环境根目录
cd
# 正确卸载文件系统以防止出现可能的错误
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -R /mnt/gentoo
```

#### 重启
恭喜你一路走到了这里，到这里，安装的工作算是告一段落了，接下来只需要重启尝试引导系统即可，强烈建议重启后直接进BIOS里调整引导项顺序，防止Boot到其他引导项


不过正式重启前先泼冷水，有可能会出现启动失败的情况，如果遇到了，重新启动LiveCD，然后从挂载分区开始排障，或者干脆重来都是可以的


不过记住，请以[Gentoo Handbook](https://wiki.gentoo.org/wiki/Handbook:Main_Page)或者[Gentoo Wiki](https://wiki.gentoo.org/)为准，这篇文章说实在的后面质量有些下滑，而且Gentoo Wiki对其他内容有更详细的分页，实在找不到相关内容，隔壁[Arch Wiki](https://wiki.archlinux.org/)也可以碰碰运气


废话有点多了，总之，重启：
```bash
reboot
```

[^1]: [Wikipedia:Master boot record#Overview](https://en.wikipedia.org/wiki/Master_boot_record#Overview)
[^2]: [Wikipedia:BIOS#Boot_process](https://en.wikipedia.org/wiki/BIOS#Boot_process)
[^3]: 详见[Manual:portage(5)](https://dev.gentoo.org/~zmedico/portage/doc/man/portage.5.html)
[^4]: [Manual:mount(8)#Shared_subtree_operations](https://man.archlinux.org/man/mount.8#Shared_subtree_operations)
[^5]: [Gentoo Handbook:Mounting_the_necessary_filesystems](https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Base#Mounting_the_necessary_filesystems)
[^6]: 需要注意是，Portage禁用了rsync的字节差异同步，强制同步完整文件[Portage:module rsync](https://gitweb.gentoo.org/proj/portage.git/tree/lib/portage/sync/modules/rsync/rsync.py)
[^7]: [Gentoo Manifest Tool](https://wiki.gentoo.org/wiki/Gemato)
[^8]: [Portage:module git](https://gitweb.gentoo.org/proj/portage.git/tree/lib/portage/sync/modules/git/git.py)
[^9]: [Manual:emerge(1)](https://dev.gentoo.org/~zmedico/portage/doc/man/emerge.1.html)
[^10]: 详见[Manual:locale.gen(5)](https://gitweb.gentoo.org/proj/locale-gen.git/tree/locale.gen.5)
[^11]: [Gentoo Wiki:Installkernel](https://wiki.gentoo.org/wiki/Installkernel)


---

> 作者: Shirano  
> URL: https://shiranosama.github.io/posts/5e0cdc0/  

