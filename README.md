GPU Passthrough with QEMU/KVM and virt-manager
======

I basically follow this arch tutorial: [PCI passthrough via OVMF](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF), please follow steps inside to check if system is capable of doing this

my system spec that works perfectly with iommu and so on:

* motherboard: GA-H170N-WIFI
* CPU: i5-6400
* RAM: 16GM with 16G swap
* storage: 256G SSD * 2
* GPU:
  * Intel HD Graphics 530 for linux host
  * NVIDIA GTX970 for windows guest
* OS: Arch Linux

## BIOS settings

set boot graphic chipset to integrated graphic:

![integrated graphic](https://i.imgur.com/t0yHqcA.jpg)

set vt-d (intel virtualization stuff) on

![vt-d](https://i.imgur.com/nZnsfZX.jpg?1)

## turn on kernel features

#### add linux boot parameters to enable iommu

add `intel_iommu=on iommu=pt` to `GRUB_CMDLINE_LINUX_DEFAULT` in `/etc/default/grub`, then run `grub-mkconfig -o /boot/grub/grub.cfg`

> see `example-etc/default/grub`

#### enable kernel modules

add `/etc/modules-load.d/vm.conf`:

```
vfio
vfio_pci
kvm
kvm_intel
```

> see `example-etc/modules-load.d/vm.conf`

#### add kernel module parameters

add `/etc/modprobe.d/vm.conf` for better performance:

```
options kvm_intel nested=1
```

> see `example-etc/modprobe.d/vm.conf`

then reboot.

## passthrough GPU

#### Identify GPU and iommu ids

run `ls_iommu.sh` of this repo, find the iommu group that GPU is in:

```
IOMMU Group 1 00:01.0 PCI bridge [0604]: Intel Corporation Xeon E3-1200 v5/E3-1500 v5/6th Gen Core Processor PCIe Controller (x16) [8086:1901] (rev 07)
IOMMU Group 1 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM204 [GeForce GTX 970] [10de:13c2] (rev a1)
IOMMU Group 1 01:00.1 Audio device [0403]: NVIDIA Corporation GM204 High Definition Audio Controller [10de:0fbb] (rev a1)
8086:1901,10de:13c2,10de:0fbb
```

all devices in the same IOMMU group will need to be passed at the same time

#### Add booting entry with VFIO enabled in grub menu

copy `example-etc/grub.d/11_gpu_vfio_linux` to `etc/grub.d/11_gpu_vfio_linux`, `chmod +x etc/grub.d/11_gpu_vfio_linux`, and modify it if needed:

```
# Configurations:
VFIO_WANTED_PCI_KEYWORDS = ['NVIDIA']
```

this script will find the group with `NVIDIA` from `ls_iommu.sh` output and add an entry on grub menu booting with the group passthrough

then `grub-mkconfig -o /boot/grub/grub.cfg` and reboot, choose "VFIO '...' Linux" to boot

> devices being passthrough will not be available for host OS

## Create windows VM

#### Add a bridge network via nmcli before adding VM

I want my windows vm to be able to access local network, just follow [this tutorial](https://www.cyberciti.biz/faq/how-to-add-network-bridge-with-nmcli-networkmanager-on-linux/)

then allow qemu to use the bridge created:

```bash
$ sudo mkdir /etc/qemu
$ sudo echo 'allow br0' >> /etc/qemu/bridge.conf
```

#### Configure libvirt

follow [this section](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Setting_up_an_OVMF-based_guest_VM)

```bash
yaourt -S qemu libvirt ovmf virt-manager
vim /etc/libvirt/qemu.conf
# nvram = [
#   "/usr/share/ovmf/x64/OVMF_CODE.fd:/usr/share/ovmf/x64/OVMF_VARS.fd"
#   ...
# ]
systemctl enable libvirtd
systemctl restart libvirtd
usermod -G libvirtd pastleo
```

then re-login.

#### Create OVMF VM

Use the pretty GUI virt-manager to create vm, basically follow [this section](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Setting_up_the_guest_OS):

1. start the GUI, click `File` -> `Add Connection` -> Hypervisor: `QEMU/KVM`
2. add VM, click `File` -> `New Virtual Machine`
3. Connection: `QEMU/KVM`, Forward.
4. Use ISO Image, Browse, Browse Local, find windows install iso file, OS type: `Windows`
5. set RAM and CPU
6. Select or create custom storage, Manage, Browse Local, find `/dev/sdx` to give whole disk
7. check `Customize configuration before install`, Network selection: `Specify shared device name`, Bridge name: `br0`

## Configure VM

#### OVMF UEFI firmware

Overview, set firmware to `UEFI`:

![ovmf uefi](https://i.imgur.com/HHaYN4Y.png?1)

> required, and the GPU have to support UEFI boot as well

#### CPU

CPUs, Configuration, check `Copy host CPU configuration`

#### Using whole disk

Set bus type to Virtio:

![virtio disk](https://i.imgur.com/8I1IFfC.png?1)

Download virtio windows driver iso from [fedora](https://docs.fedoraproject.org/quick-docs/en-US/creating-windows-virtual-machines-using-virtio-drivers.html), Add hardware, Storage, Select or create custom storage, Manage, Browse Local, select the virtio windows driver iso, Device type: `CDROM device`, Bus type: `IDE`

Boot Options, Enable boot menu, IDE CDROM 1 first, VirtIO Disk 1 second

#### Attatch GPU to VM

* Add hardware, PCI Host Device, choose GPU
* Add hardware, PCI Host Device, choose GPU audio

#### Audio

Model: `ac97`:

![audio-ac97](https://i.imgur.com/Xx0hFJh.png)

## Install windows

Begin Installation, use windows installation iso to boot

#### virtio disk driver

virtio driver is required to detect disk:

![virtio-disk-driver-1](https://i.imgur.com/E6tIjFn.png)

![virtio-disk-driver-2](https://i.imgur.com/0YSW4sT.png)

![virtio-disk-driver-3](https://i.imgur.com/klrqFNE.png)

#### AC97 driver

after windows installation, reboot without driver signature verification, [follow Option Two of this post](https://www.howtogeek.com/167723/how-to-disable-driver-signature-verification-on-64-bit-windows-8.1-so-that-you-can-install-unsigned-drivers/)

Download and install AC97 driver from [realtek](http://www.realtek.com.tw/downloads/downloadsView.aspx?Langid=1&PNid=14&PFid=23&Level=4&Conn=3&DownTypeID=3&GetDown=false), ignore signature verification failure warn

> refer to [this video](https://www.youtube.com/watch?v=5-Y-oq3DMMA)

## options cannot be configured by GUI

```bash
vim /etc/libvirt/qemu/vm_name.xml
systemctl restart libvirtd
```

> see `example-etc/libvirt/qemu/win10.xml`

#### Mouse and keyboard

> refer to https://passthroughpo.st/using-evdev-passthrough-seamless-vm-input/

```bash
# find devices want to pass
cd /dev/input/by-id
cat dev_id # check which one

vim /etc/libvirt/qemu.conf
# user = "username"
# cgroup_device_acl = [
#   "/dev/input/by-id/input_dev_id",
# ]

vim /etc/libvirt/qemu/vm_name.xml
# modify top level <domain>:
# <domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
# add qemu parameter before </domain>:
# <qemu:commandline>
#   <qemu:arg value='-object'/>
#   <qemu:arg value='input-linux,id=mouse1,evdev=/dev/input/by-id/...'/>
#   <qemu:arg value='-object'/>
#   <qemu:arg value='input-linux,id=kbd1,evdev=/dev/input/by-id/...,grab_all=on,repeat=on'/>
# </qemu:commandline>
```

reboot the vm, *press left ctrl and right ctrl* to switch between host and vm

#### Prevent GPU error 43

```xml
      <vendor_id state='on' value='123456789ab'/>
```

#### fool windows

```xml
    <kvm>
      <hidden state='on'/>
    </kvm>
```

```xml
  <cpu mode='host-passthrough' check='none'/>
```

