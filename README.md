
---

# 3455PVE方案分享 

---

## 前言：
**全文围绕如何用J3455做一个装了群晖系统的HTPC。**

如果你需要kodi，或者需要用到j3455物理端口输出视频和音频，同时又舍不得pve的扩展性。那么请继续浏览下去。

本文采用的方案是ProxmoxVE做为宿主，Libreelec作为显卡、声卡直通的客户端，群晖作为直通sata控制器的客户端。（如果将libreelec改为windows，更可以用上jellyfin的硬解，不过我不需要。）

这样的方案具备几点优势：

1. 声卡和显卡得到充分的利用，硬解4K，音频光纤口输出，满足大部分到片源播放需求。
2. 板载的sata控制器直通到群晖，硬盘可读smart信息，亦可根据当前任务，进入休眠状态。休眠后的功耗在15W以内。
3. 得益于PVE，具备极佳的扩展性。

数日的折腾，走过了不少坑，包括但不限于，显卡直通无视频输出，群晖内部显示52M的引导启动盘、读不到ip、无法休眠。。

在此特别感谢 qq群606939663内大佬的帮助，尤其是杭州-nv，howard，漫步。。

## PVE
相关的安装教程很多，也简单，我选择的版本是近期发布的**pve6.0**,搭配**漫步发来的kernel。**

关于pve底层配置，最关键的就是如何搞定pcie直通，显卡直通。我只贴配置，粗体的是命令，引用的代码需粘贴进去。具体的说明请参考PVE官方的[pci-passthrough](https://pve.proxmox.com/wiki/Pci_passthrough)章节，以及tgfcer的[J3455安装PVE折腾记录（直通GPU至Libreelec当HTPC+黑群+OMV）20190703更新简易版](https://club.tgfcer.com/viewthread.php?tid=7657483&extra=&page=1)。

* **nano /etc/default/grub**
	* ```GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on video=efifb:off,vesafb:off"```
* **update-grub**
* **nano /etc/modules**
```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```
* **nano /etc/modprobe.d/blacklist.conf**
```
blacklist snd_hda_intel
blacklist snd_soc_skl
blacklist snd_hda_codec_hdmi
blacklist i915
```
* **echo "options vfio-pci ids=8086:5a85,8086:5a98" > /etc/modprobe.d/vfio.conf**
* **echo "options vfio_iommu_type1 allow_unsafe_interrupts=1" > /etc/modprobe.d/iommu_unsafe_interrupts.conf**
* **echo "options kvm ignore_msrs=1" > /etc/modprobe.d/kvm.conf**
* **update-initramfs -u**
* wget https://raw.githubusercontent.com/wanghuangjie/Perfect_PVE_3455/master/hd500.bin -P /usr/share/kvm/

至此pve宿主完工。


## 虚拟机
### Libreelec
引导镜像 LibreELEC-Generic.x86_64-9.1.002.img
常规操作,img2kvm导入磁盘后，以磁盘为启动盘。

**重点: nano /etc/pve/qemu-server/[vmid].conf** 添加

```args: -device vfio-pci,host=00:02.0,addr=0x02,x-igd-gms=1,romfile=j3455_hd500.bin```

```hostpci0: 00:0e,rombar=0```

```vga: none```

安装启动。

### 群晖
网上的教程一大票，6.2版本也可引导，需改网卡为1000e。因此我选择的版本是6.17。不再赘述，只提几点。
以下内容非唯一，但可以实现硬盘单独休眠。即便跑个docker，也不影响硬盘休眠。这点于我而言，很！重！要！

* 选对文件，引导镜像**DS3617xs DSM6.1_Broadwell**，安装包**DSM DS3617xs_15266.pat**
* pve虚拟的磁盘用于安装套件，载为sata0，引导盘载为sata1，改grub内`set extra_args_3617='DiskIdxMap=0000 SataPortMap=12'`
* 安装完毕群晖后，（再次感谢杭州nv提供的神操作，如此一番便不会因为一个硬盘有操作影响其他磁盘的休眠）
	* 移除直通的sata控制器，开机，再关机。
	* 再次添加直通的sata控制器，开机。


## 结语
感谢阅读，亦感激群内大佬的帮助。
