# 在 PVE 中安装 RouterOS x86：原帖整理版

> 根据恩山无线论坛 AstroByte808 的帖子整理，原帖最后编辑于 2025-09-06：[查看原帖](https://www.right.com.cn/forum/forum.php?mod=viewthread&tid=8423698)。本文记录原作者的操作过程，未在实际设备上验证；版本及授权结果可能随环境而异。

![原帖概览](routeros-article-images/095914rietiv4sss5id4b0.png)

## 准备工作

原教程使用 PVE 创建虚拟机，并准备以下软件及镜像：

- FinalShell，用于通过 SSH 管理 PVE：[官网](https://www.hostbuf.com/)。
- Winbox，用于连接 RouterOS：[MikroTik 官网](https://mikrotik.com/)。原帖要求使用 Winbox 3 系列。
- RouterOS 7.12.1 x86 ISO 镜像。
- Ubuntu Desktop ISO 镜像，原帖使用 `ubuntu-24.04.2-desktop-amd64.iso`。
https://download.mikrotik.com/routeros/winbox/3.41/winbox64.exe
经验证的ISO：
https://download.mikrotik.com/routeros/7.12.1/mikrotik-7.12.1.iso
MD5:e5e1ee8754ec25367faffa594f78f100

先将两个 ISO 上传到 PVE 存储中的 ISO 镜像目录。按 PVE 的创建向导新建虚拟机，记下虚拟机 ID；下文以 `XXX` 表示该 ID。创建选项见以下原帖截图。

![上传镜像与创建虚拟机（原图 05）](img/05.png)

![上传镜像与创建虚拟机（原图 07）](img/07.png)

![上传镜像与创建虚拟机（原图 10）](img/10.png)

![上传镜像与创建虚拟机（原图 11）](img/11.png)

![上传镜像与创建虚拟机（原图 12）](img/12.png)

![上传镜像与创建虚拟机（原图 13）](img/13.png)

![上传镜像与创建虚拟机（原图 14）](img/14.png)

![上传镜像与创建虚拟机（原图 15）](img/15.png)

![上传镜像与创建虚拟机（原图 16）](img/16.png)

![上传镜像与创建虚拟机（原图 17）](img/17.png)

## 创建并连接虚拟磁盘

原帖使用大小为 **6,442,450,944 字节（6 GiB）**的 QCOW2 磁盘。作者指出，如果使用 `local` 存储，应在 PVE 的「数据中心 → 存储」中确认 `local` 允许存放磁盘镜像。

通过 SSH 登录 PVE，将下面命令中的 `XXX` 替换为实际虚拟机 ID：

```bash
mkdir -p /var/lib/vz/images/XXX
qemu-img create -f qcow2 /var/lib/vz/images/XXX/vm-XXX-disk-0.qcow2 6442450944
qm disk rescan --vmid XXX
```

原帖特别说明网页中的 `qemu-img` 命令显示异常，上面按上下文整理为 `-f qcow2`，请对照下图确认。

![创建磁盘（原图 18）](img/18.png)

回到 PVE 的虚拟机硬件页面，找到扫描出的「未使用的磁盘」，编辑并添加；原帖要求总线选择 IDE。随后编辑 `/etc/pve/local/qemu-server/XXX.conf`，在磁盘配置末尾追加型号和序列号。原帖示例为：

```text
ide0: local:100/vm-100-disk-0.qcow2,size=6G,model=VMware%20Virtual%20IDE%20Hard%20Drive,serial=00000000000000000001
```

示例中的虚拟机 ID 和存储路径需要对应实际环境，不应直接覆盖整个配置文件。

![添加磁盘（原图 19）](img/19.png)

![添加磁盘（原图 20）](img/20.png)

![添加磁盘（原图 22）](img/22.png)

## 安装 RouterOS

1. 在虚拟机的 CD/DVD 驱动器中挂载 RouterOS ISO，启动虚拟机。
2. 原帖称在安装界面输入 `i`，再输入 `y` 确认安装。
3. 安装完成后关闭虚拟机，将启动顺序改为硬盘优先。
4. 从硬盘启动一次，使 RouterOS 完成首次启动，然后关闭虚拟机。

安装界面的具体状态及提示应以实际屏幕为准。

![安装系统（原图 23）](img/23.png)

![安装系统（原图 24）](img/24.png)

![安装系统（原图 25）](img/25.png)

## 原帖中的 MBR 操作与授权

原帖接着使用 Ubuntu 桌面环境、`fdisk`、`dd` 和 GHex 修改虚拟磁盘的 MBR，再导入帖子提供的密钥。写错磁盘或扇区可能导致系统无法启动或数据丢失；请先备份并确认设备路径。此方法的授权有效性未核实。

### 启动 Ubuntu 并准备工具

关闭虚拟机，把 CD/DVD 驱动器换成 Ubuntu ISO，设置光驱优先启动。启动后选择 **Try or Install Ubuntu**，进入桌面并打开终端。

![启动 Ubuntu（原图 26）](img/26.png)

![启动 Ubuntu（原图 27）](img/27.png)

![启动 Ubuntu（原图 28）](img/28.png)

![启动 Ubuntu（原图 29）](img/29.png)

更新软件源并安装十六进制编辑器：

```bash
sudo apt update
sudo apt install ghex -y
```

![安装工具（原图 31）](img/31.png)

![安装工具（原图 32）](img/32.png)

### 导出并编辑引导扇区

运行 `sudo fdisk -l`，根据 6,442,450,944 字节的容量确认目标虚拟磁盘。下面的 `/dev/sda` 是原帖示例，必须替换成实际确认的设备路径。

![确认磁盘（原图 33）](img/33.png)

![确认磁盘（原图 34）](img/34.png)

在终端进入存放文件的目录，将磁盘的前 512 字节导出为 `abc`：

```bash
sudo dd if=/dev/sda of=abc bs=512 count=1
```

![导出扇区（原图 35）](img/35.png)

用 GHex 打开 `abc`。原帖要求对照下图，将对应的随机码区域改为 `BD` 和 `E8`，然后另存为 `xxx`。截图以每行 16 字节显示；原帖没有用文字标明偏移，需仔细对照图片。

![编辑扇区（原图 36）](img/36.png)

![编辑扇区（原图 37）](img/37.png)

![编辑扇区（原图 38）](img/38.png)

![编辑扇区（原图 39）](img/39.png)

![编辑扇区（原图 40）](img/40.png)

![编辑扇区（原图 41）](img/41.png)

![编辑扇区（原图 42）](img/42.png)

确认修改内容和目标设备后，在 `xxx` 所在目录执行原帖的写回命令，再关闭虚拟机：

```bash
sudo dd if=xxx of=/dev/sda bs=512 count=1
```

![写回扇区（原图 43）](img/43.png)

### 启动 RouterOS 并导入原帖密钥

移除光驱或取消挂载 ISO，将硬盘设为第一启动项并启动 RouterOS。使用 Winbox 连接；原帖标注首次登录用户名为 `admin`，密码为空，并建议尽快修改密码。

![启动与登录（原图 44）](img/44.png)

![启动与登录（原图 45）](img/45.png)

![启动与登录（原图 46）](img/46.png)

![启动与登录（原图 47）](img/47.png)

![启动与登录（原图 48）](img/48.png)

原帖提供以下密钥，要求保存为 `TI09-7WK3.key`，在 Winbox 中导入，然后按提示重启。该密钥与操作方法能否在其他环境工作、是否具有有效授权，本文无法验证。

```text
-----BEGIN MIKROTIK SOFTWARE KEY------------
mr3jH5qhn9irtF53ZICFTN7Tk7wIx7ZkxdAxJ19ydASY
ShhFteHMntBTyaS8wuNdIJJPidJxbuNPLTvCsv7zLA==
-----END MIKROTIK SOFTWARE KEY--------------
```

![导入密钥（原图 49）](img/49.png)

![导入密钥（原图 50）](img/50.png)

![导入密钥（原图 51）](img/51.png)

## 后续配置与原帖更新

原帖建议首次登录后尽快修改默认账号密码，并在完成网络设置后更新系统。作者另提供了基础配置资料：

- [RouterOS 配置笔记](https://gitee.com/callmer/routeros_toss_notes)
- [RouterOS 基础配置视频](https://www.bilibili.com/video/BV1KD421H75o)

原帖还讨论了 RouterOS 配合 Linux、MosDNS 和 Sing-Box 的网络方案，并在 2025-08-21 更新了对 TUN、Alpine Linux 和 AdGuard Home 的个人使用意见。这些内容属于作者当时的经验，不能替代对当前版本的测试。

## 图片归档状态

正文图使用 `img/` 下的相对路径；原帖封面使用 `routeros-article-images/`。原帖中的 `01.png`、`03.png`、`21.png` 未出现在目前提供的图片目录中，其余正文截图已按原帖顺序插入。迁移到 GitHub 时请一并保留这两个图片目录。
