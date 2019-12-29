在 ArchLinux 上使用 KVM 虛擬 Windows 打電動
======

## 硬體需求

* [檢查硬體是否能跑 KVM](https://wiki.archlinux.org/index.php/KVM#Hardware_support)
* [PCI passthrough via OVMF Prerequisites](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Prerequisites)

我的機器大概長這樣

* 主機板: GA-H170N-WIFI
* CPU: i5-6400
* RAM: 16G with 16G swap
* GPU:
  * Intel HD Graphics 530 for linux host
  * NVIDIA GTX970 (dGPU) for windows guest
* host OS: [Arch Linux](https://www.archlinux.org/)
  * desktop environment: [KDE Plasma](https://kde.org/plasma-desktop)

> 有兩個 GPU 比較適合這樣玩，因為把 GPU 給 guest 之後要有辦法操作 host OS，而且 host OS 已經跑起來（甚至有 GUI）再把 GPU hot-swap 到 guest OS 也是一項非常困難的事情

這篇文章假設你已對 Linux 有一定程度的了解（權限，檔案系統，設定檔等等的慣例）如果你日常生活是使用 ArchLinux，相信對 Linux 各個方面已經有一定程度的了解，確認一下自己的機器覺得適合這樣玩的話，以下是我整理出來的設定步驟：

## BIOS settings (GA-H170N-WIFI as example)

讓預設 GPU 使用內顯避免用到 dGPU

![integrated graphic](https://i.imgur.com/nZnsfZX.jpg?1)

把 vt-d (或是任何 Virtualization 的功能) 打開

![vt-d](https://i.imgur.com/t0yHqcA.jpg)

## Kernel features

### 在 grub 開機參數中啟用 iommu

加入 `intel_iommu=on iommu=pt` 到 [`/etc/default/grub`](https://github.com/pastleo/kvm-setups/blob/master/example-etc/default/grub#L6) 環境變數  `GRUB_CMDLINE_LINUX_DEFAULT`, 像是這樣:

```
...
GRUB_DISTRIBUTOR="Arch"
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
GRUB_CMDLINE_LINUX=""
...
```

接著執行 `grub-mkconfig` 重新產生 grub config 檔案:

```shell=
grub-mkconfig -o /boot/grub/grub.cfg
```

### 啟用 kernel modules

add [`/etc/modules-load.d/vm.conf`](https://github.com/pastleo/kvm-setups/blob/master/example-etc/modules-load.d/vm.conf):

```
vfio
vfio_pci
kvm
kvm_intel
virtio-pci
virtio-net
virtio-blk
virtio-balloon
virtio-ring
virtio
```

### 設定 kernel module 參數

add [`/etc/modprobe.d/vm.conf`](https://github.com/pastleo/kvm-setups/blob/master/example-etc/modprobe.d/vm.conf):

```
options kvm_intel nested=1
```

### kernel 相關的設定完成之後重新啟動

如果可以成功開機，檢查看看功能是否正常：https://wiki.archlinux.org/index.php/KVM#Kernel_support

## 使 dGPU 分離 host OS 以便進行 passthrough

在開機的時候就把準備給 guest OS 的 dGPU 從 host OS 中分離，以保持 dGPU 是乾淨沒有被使用的狀態，同時會依照硬體限制把 PCI hardware 分成一些 IOMMU group，必須以 IOMMU group 為單位做分離

### 檢查 iommu group

這個 repo 有個 [`ls_iommu.sh`](https://github.com/pastleo/kvm-setups/blob/master/ls_iommu.sh) 可以用來看 PCI hardware 分別在哪個 IOMMU group:

```
IOMMU Group 1 00:01.0 PCI bridge [0604]: Intel Corporation Xeon E3-1200 v5/E3-1500 v5/6th Gen Core Processor PCIe Controller (x16) [8086:1901] (rev 07)
IOMMU Group 1 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM204 [GeForce GTX 970] [10de:13c2] (rev a1)
IOMMU Group 1 01:00.1 Audio device [0403]: NVIDIA Corporation GM204 High Definition Audio Controller [10de:0fbb] (rev a1)
8086:1901,10de:13c2,10de:0fbb
```

我這張 dGPU 上面同時還有聲音輸出的 PCI device，加上 PCI bridge 會一起被分離

### 加入要分離 dGPU 的 grub 開機選項

我用 [Ruby](https://www.ruby-lang.org/zh_tw/) 寫了一個 script 來幫忙偵測並產生分離 dGPU 的開機選項：https://github.com/pastleo/kvm-setups/blob/master/example-etc/grub.d/11_gpu_vfio_linux

> 用 Ruby 寫的意思就是要先把 [ruby](https://www.archlinux.org/packages/extra/x86_64/ruby/) 安裝好：pacman -S ruby

1. 把這個檔案下載回去並放在 `/etc/grub.d/11_gpu_vfio_linux`
2. 給予執行權限: `chmod +x /etc/grub.d/11_gpu_vfio_linux`
3. 依照狀況修改 `vim /etc/grub.d/11_gpu_vfio_linux`，尤其是 [`VFIO_WANTED_PCI_KEYWORDS`](https://github.com/pastleo/kvm-setups/blob/master/example-etc/grub.d/11_gpu_vfio_linux#L5)：

```
# Configurations:
VFIO_WANTED_PCI_KEYWORDS = ['NVIDIA']
```

這個 script 會尋找包含 `VFIO_WANTED_PCI_KEYWORDS` 的 PCI device 並產生分離同個 IOMMU group PCI devices 的 grub 開機選項，設定好之後執行 `grub-mkconfig` 重新產生 grub config 檔案:

```shell=
grub-mkconfig -o /boot/grub/grub.cfg
```

完成之後重新開機應該可以看到 `VFIO with 'NVIDIA' Linux` 的開機選項，選擇該開機選項開機的時候就會以分離 dGPU 的模式開機，***連接該 dGPU 的螢幕就不會有畫面了***，使用 [Bumblebee](https://wiki.archlinux.org/index.php/Bumblebee) 之類技術的筆電我個人沒試過不知道會發生什麼事

> 因為 Nvidia 對 Linux 的相容性非常糟糕，我已經修改 grub 預設開機選項 `grub-set-default 2` (從 0 開始，這樣代表把預設設定成第三個選項)，常態性把這張顯示卡處於分離的狀態

## 安裝並設定好 libvirt, QEMU

```shell=
yay -S qemu libvirt ovmf virt-manager ebtables dnsmasq bridge-utils
```

### 設定權限

> 這邊的 `[USER]` 請替換成自己 GUI session 用的 user

```shell=
usermod -a -G libvirt [USER]
usermod -a -G input [USER]
```

重新登入 user session 讓設定生效

### 設定 [`/etc/libvirt/qemu.conf`](https://github.com/pastleo/kvm-setups/blob/master/example-etc/libvirt/qemu.conf)

```shell=
vim /etc/libvirt/qemu.conf
```

#### 1. 設定 qemu 執行時使用的使用者

讓虛擬機用 GUI session 的身份來執行：

```
...
user = "[USER]"
...
group = "[USER]"
...
```

#### 2. 允許存取鍵盤滑鼠 `evdev`

我用同組鍵盤滑鼠來操作 guest / host OS，同時兼具效能又不用兩組鍵盤滑鼠的解決方案就是 [evdev passthrough](https://passthroughpo.st/using-evdev-passthrough-seamless-vm-input/)，在這邊先設定 `cgroup_device_acl` 允許 qemu 使用這些 devices:

```
group_device_acl = [
    "/dev/null", "/dev/full", "/dev/zero",
    "/dev/random", "/dev/urandom",
    "/dev/ptmx", "/dev/kvm",
    "/dev/rtc","/dev/hpet",
    "/dev/input/by-id/[some_mouse_device]-event-mouse",
    "/dev/input/by-id/[some_keyboard_device]-event-kbd"
]
```

* 前面的 `"/dev/null", "/dev/full" ...` 是必要的
* `"/dev/input/by-id/[some_mouse_device]-event-mouse", "/dev/input/by-id/[some_keyboard_device]-event-kbd"` 換成鍵盤滑鼠對應之 `evdev` path
  * `cd /dev/input/by-id/`
  * `cat ./[some_mouse_device,some_keyboard_device]-event-{mouse,kbd}`
  * 動動滑鼠，敲敲鍵盤確認哪個 `evdev` 是要 passthrough 的

改完之後 `:wq` 存檔離開

### `/usr/bin/qemu-kvm` 不存在的問題

個人認為這個算是 libvirt 的 bug ，先幫忙處理一下（都過這麼久了...）：

```shell=
ln -s /usr/bin/qemu-{system-x86_64,kvm}
```

### 啟動 libvirt service:

```shell=
systemctl enable libvirtd
systemctl restart libvirtd
```

### 啟用 virtual network

這個動作我們透過 `virt-manager` GUI 來做，應該可以在 desktop environment 的應用程式清單找到 `Virtual Machine Manager`，要不然在 terminal 輸入 `virt-manager` 啟動:

![](https://i.imgur.com/OGlG7ac.png)

`Edit` -> `Connection Details` -> `Virtual Networks`, `default`, 句選 `Autostart` 然後按下播放圖示啟動 virtual network:

![enable-virt-network](https://i.imgur.com/tVAbzeT.png)

#### 如果有需要也可以設定一些固定 IP

[StackExchange](https://serverfault.com/questions/627238/kvm-libvirt-how-to-configure-static-guest-ip-addresses-on-the-virtualisation-ho)

```shell=
virsh net-edit default
```

```xml=
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
      <host mac='52:xx:xx:xx:xx:xx' name='vm-1-name' ip='192.168.122.53'/>
      <host mac='52:xx:xx:xx:xx:xx' name='vm-2-name' ip='192.168.122.54'/>
    </dhcp>
  </ip>
```

## 建立 Windows Gaming 虛擬機

### 準備 UEFI firmware `OVMF`

必須使用 [OVMF](https://github.com/tianocore/tianocore.github.io/wiki/OVMF) UEFI firmware 才能支援 dGPU passthrough，而且 UEFI firmware 需要搭配快閃記憶體 `nvram` ，我們從 template 複製出來準備好：

```shell=
mkdir -p /var/lib/libvirt/qemu/nvram
cp /usr/share/ovmf/x64/OVMF_VARS.fd /var/lib/libvirt/qemu/nvram/[vm_name]_VARS.fd
```

### 建立 VM

透過 `virt-manager` GUI 建立虛擬機，注意 hypervisor (Connection) 要選擇 `QEMU/KVM`，這邊的設定就很麻瓜了，最後一步比較需要注意的：

* name 欄位對應上面跟之後寫的 `[vm_name]`
* 句選 `Customize configuration before install`

按下建立之後開始做細部的設定

#### `Overview` => `Hypervisor Details`

* Chipset: `i440FX`
* Firmware: 先選 `BIOS` 就好，不用動

#### CPU & 記憶體

* CPU: 設定要幾顆 CPU，句選 `Copy host CPU configuration`，`Topology` 也設定一下避免 guest OS 沒偵測到全部的 vCPU
* 記憶體: 就看要分多少給 guest OS，guest OS 在啟動的瞬間就會直接把 `Current allocation` 直接吃走

#### 硬碟

這部份就看要怎麼弄，如果打算用預設的方式（也就是建立一個 `qcow2` image 檔案）就不太需要改什麼，我這邊打算直接把實體硬碟的一個 partition 分配給虛擬機使用

![](https://i.imgur.com/aNAoQgC.png)

> 可以用 `ls -l /dev/disk/by-id` / `ls -l /dev/disk/by-partuuid` 來看要用哪個 disk/partition

我這邊選擇 Bus type 為 `VirtIO`，[官方推薦效能較好](https://www.linux-kvm.org/page/Tuning_KVM)，但是接下來在 Windows 安裝時會需要 [virtio driver](https://docs.fedoraproject.org/en-US/quick-docs/creating-windows-virtual-machines-using-virtio-drivers/#virtio-win-direct-downloads)

下載 iso 回來並且增加 CDROM Storage 並指向 `virtio-win.iso`

#### 其他硬體設定

* 聲音 `Model` 選用 `AC97`，這個需要特別安裝驅動程式
* 網路 `Device model` 選 `virtio`，也[是官方推薦效能較好](https://www.linux-kvm.org/page/Tuning_KVM)，不過也會需要 [virtio driver](https://docs.fedoraproject.org/en-US/quick-docs/creating-windows-virtual-machines-using-virtio-drivers/#virtio-win-direct-downloads)
* `+ Add Hardware` => `PCI Host Device` 把 dGPU 加入！

> 聲音的部份有稍微實驗一下 `HDA (ICH9)` 跟 `AC97` 的差別，兩者其實都會有一些延遲（大概 250ms 左右，還算可接受），但是 `AC97` 不會有破音的狀況發生

### 接著要直接去修改設定檔

按下 `Begin Installation` 建立機器，他會幫你把機器啟動，請直接關閉 (force off)，然後用這個指令開始手動修改（需要 `sudo`）：

```shell=
virsh edit [vm_name]
```

> 會用 `vi` 開啟設定檔

#### 1. CPU 綁定(pinning)

設定這個來綁定 vCPU 對應哪顆 CPU，可以增加虛擬機的效能，不過當然得看清楚自己電腦 CPU 的狀況來設定，可以參考 [ArchLinux wiki 上的教學](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#CPU_pinning)，我的機器是 4 核心 CPU 因此設定成：

```xml
<domain>
  ...
  <cputune>
    <vcpupin vcpu='0' cpuset='0'/>
    <vcpupin vcpu='1' cpuset='1'/>
    <vcpupin vcpu='2' cpuset='2'/>
    <vcpupin vcpu='3' cpuset='3'/>
  </cputune>
  ...
</domain>
```

之後機器啟動可以用 `virsh vcpuinfo [vm_name]` 來觀察 CPU time 跟綁定狀況

#### 2. 設定 UEFI firmware 以及 nvram

```xml
<domain>
  ...
  <os>
    ...
    <loader readonly='yes' type='pflash'>/usr/share/ovmf/x64/OVMF_CODE.fd</loader>
    <nvram>/var/lib/libvirt/qemu/nvram/[vm_name]_VARS.fd</nvram>
    ...
  </os>
  ...
</domain>
```


#### 3. 避免 Nvidia GPU Error 43

```xml
<domain>
  ...
  <features>
    ...
    <hyperv>
      ...
      <vendor_id state='on' value='xxxxxxxx'/>
      ...
    </hyperv>
    ...
  </features>
  ...
  <kvm>
    <hidden state='on'/>
  </kvm>
  ...
</domain>
```

`xxxxxxxx` 填寫一個隨機的英數字串就可以

#### 4. 鍵盤滑鼠 `evdev` passthrough

首先要把第一行改成這樣：

```diff
- <domain type='kvm'>
+ <domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
```

加入 `<qemu:commandline>`，`input-linux` 指定上面設定過允許要 passthrough 的鍵盤滑鼠 `evdev`：

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  ...
  <qemu:commandline>
    <qemu:arg value='-object'/>
    <qemu:arg value='input-linux,id=mouse1,evdev=/dev/input/by-id/[some_mouse_device]-event-mouse'/>
    <qemu:arg value='-object'/>
    <qemu:arg value='input-linux,id=kbd1,evdev=/dev/input/by-id/[some_keyboard_device]-event-kbd,grab_all=on'/>
  </qemu:commandline>
</domain>
```

#### 5. 指定聲音輸出到 userspace 的 PulseAudio server 而非透過 `virt-manager` 視窗

我個人稍微實驗了一下，這步不是必要的，也不會讓效能比較好，加上這些設定是讓虛擬機的 `virt-manager` 視窗關閉的時候依然可以輸出聲音到 userspace 的 PulseAudio server

加入 `QEMU_AUDIO_DRV`, `QEMU_PA_SERVER` 環境變數指定 PulseAudio server，可以用`ls -l /run/user/1000/pulse/native` 確認一下 PulseAudio server，如果 uid 不是 `1000` 有可能就不在這個位置上

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  ...
  <qemu:commandline>
    ...
    <qemu:env name='QEMU_AUDIO_DRV' value='pa'/>
    <qemu:env name='QEMU_PA_SERVER' value='/run/user/1000/pulse/native'/>
  </qemu:commandline>
</domain>
```

改完 `:wq` 存檔，應該可以看到 `Domain [vm_name] XML configuration edited` 表示成功修改

### 開機，安裝 Windows

用 `virt-manager` GUI 啟動虛擬機，

#### 因為設定了 `evdev` passthrough，按下啟動的瞬間鍵盤滑鼠會被虛擬機吃掉，虛擬機啟動之後 按下左 Ctrl 加 右 Ctrl 來切換操作 host/guest

同時應該可以看到接在 passthrough dGPU 的螢幕會亮起來顯示 `Tianocore` 的 logo，不過在 driver 安裝完成之前操作都還是在 `virt-manager` 視窗裡

進入 Windows 安裝程式開始安裝，如果你像我一樣在硬碟 bus 的地方選用 `virtio`，你會看到：

![virtio-disk-driver-1](https://i.imgur.com/E6tIjFn.png)

載入驅動程式，如果有加入 `virtio-win.iso` 應該可以自動搜尋到 driver，如果沒有再自己找：

![virtio-disk-driver-2](https://i.imgur.com/0YSW4sT.png)

![virtio-disk-driver-3](https://i.imgur.com/klrqFNE.png)

之後就是正常的 Windows 安裝程序，安裝完成開到 Windows

### 在 guest OS 上安裝驅動程式

Windows 開起來之後，在 Windows 圖案上按下右鍵，打開 `裝置管理員`，會有幾個裝置上面有驚嘆號表示沒有驅動程式：

* 網路卡：按下右鍵選更新驅動程式，然後把搜尋目錄設定在 `virtio-win.iso` CD-ROM 根目錄應該可以找的到
* 顯示卡：網路卡正常運作後，顯示卡應該直接按下右鍵選更新驅動程式，然後讓他自動去抓就可以了
* AC97 音效卡：
  * 從 [Realtek](https://www.realtek.com/en/component/zoo/category/pc-audio-codecs-ac-97-audio-codecs-software) 下載驅動程式並且解壓放好
    * 我的 Guest OS 是 Windows 10，用 `Vista/Win7 (32/64 bits) Driver only (ZIP file)` 可行
    * 需要填寫 Email 才能下載
  * 顯然硬體不是官方的...需要關閉驅動程式簽章檢查才能安裝...請見這個影片：https://www.youtube.com/watch?v=5-Y-oq3DMMA
* 可能還會有其他有驚嘆號的裝置，理論上都是 virtio 相關的硬體，一樣按下右鍵選更新驅動程式，然後把搜尋目錄設定在 `virtio-win.iso` CD-ROM 根目錄應該可以找的到

把 GPU driver 裝起來應該可以看到順順的畫面從 dGPU 輸出到螢幕上囉！

#### 後記：一個方便的螢幕接法

Windows 虛擬機沒啟動的時候，這樣接使用兩個螢幕：

```
Intel Graphics --- Monitor 1
               \
                \
                 \
Graphic card   --- Monitor 2
```

如果要啟動 Windows 虛擬機，也不用一直把線拔來拔去，修改顯示設定讓 host OS 不要輸出畫面到 Monitor 2 即可

## macOS High Sierra VM

thanks to [kholia/OSX-KVM](https://github.com/kholia/OSX-KVM/tree/master/HighSierra), this can be done easily, and I prefer to use virt-manager

#### Prepare installation iso

follow [Installation Preparation](https://github.com/kholia/OSX-KVM/tree/master/HighSierra#preparation-steps-on-your-current-macos-installation), using `create_iso_highsierra.sh` to create iso and copy to linux host

#### Prepare UEFI firmwares and clover image

```
cd to/some/dir
git clone https://github.com/kholia/OSX-KVM.git
# at the time git commit sha is cfd120dd3092fb38a89544785b2a97bc93668b44
cd OSX-KVM
sudo cp OVMF_CODE.fd /usr/share/ovmf/x64/OVMF_CODE_MACOS_HS.fd
sudo cp OVMF_VARS-1024x768.fd /var/lib/libvirt/qemu/nvram/macos-high-sierra_VARS.fd
sudo cp Clover.qcow2 /var/lib/libvirt/images/Clover.qcow2
```

this firmware will make vm mac screen only 1024x768, visit [Preparation steps on your QEMU system in kholia/OSX-KVM](https://github.com/kholia/OSX-KVM/tree/master/HighSierra#preparation-steps-on-your-qemu-system) for more info

#### Create Mac VM via XML

download [example-etc/libvirt/qemu/macos-high-sierra.xml](https://github.com/pastleo/kvm-setups/blob/master/example-etc/libvirt/qemu/macos-high-sierra.xml)

```shell=
vim macos-high-sierra.xml # change lines marked by CHANGEME
virsh define macos-high-sierra.xml
```

this xml is modified from [https://github.com/kholia/OSX-KVM/blob/cfd120dd3092fb38a89544785b2a97bc93668b44/macOS-HS-libvirt.xml](https://github.com/kholia/OSX-KVM/blob/cfd120dd3092fb38a89544785b2a97bc93668b44/macOS-HS-libvirt.xml)

#### Configure Mac VM

using virt-manager GUI:

* CPU and RAM: make sure they fits hardware
* Main storage: `Add Hardware` -> `Storage` -> Create or Select image, Disk type: `Disk device` and Bus type: `SATA`
* network: xml already set to use virt default NAT

#### Install macOS

Set CDROM to use the iso from `create_iso_highsierra.sh`, then just follow [instructions from kholia/OSX-KVM](https://github.com/kholia/OSX-KVM/tree/master/HighSierra#installer-steps)
