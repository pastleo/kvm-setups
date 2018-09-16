My QEMU/KVM and virt-manager setups
======

## Hardware support

* least requirement: [KVM hardware support](https://wiki.archlinux.org/index.php/KVM#Hardware_support)
* GPU passthrough: [PCI passthrough via OVMF Prerequisites](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Prerequisites)

my system specs to run with GPU passthrough

* motherboard: GA-H170N-WIFI
* CPU: i5-6400
* RAM: 16G with 16G swap
* storage: 256G SSD * 2
* GPU:
  * Intel HD Graphics 530 for linux host
  * NVIDIA GTX970 for windows guest
* OS: Arch Linux

## BIOS settings (GA-H170N-WIFI as example)

set boot graphic chipset to integrated graphic (for GPU passthrough, avoid using dGPU):

![integrated graphic](https://i.imgur.com/nZnsfZX.jpg?1)

set vt-d (intel virtualization stuff) on

![vt-d](https://i.imgur.com/t0yHqcA.jpg)

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
virtio-pci
virtio-net
virtio-blk
virtio-balloon
virtio-ring
virtio
```

> see `example-etc/modules-load.d/vm.conf`

#### add kernel module parameters

add `/etc/modprobe.d/vm.conf` for better performance:

```
options kvm_intel nested=1
```

> see `example-etc/modprobe.d/vm.conf`

then reboot, [check if kernel features is enabled](https://wiki.archlinux.org/index.php/KVM#Kernel_support)

## passthrough GPU

#### Identify GPU and iommu ids

run `ls_iommu.sh` of this repo, find the iommu group that GPU is in:

```
IOMMU Group 1 00:01.0 PCI bridge [0604]: Intel Corporation Xeon E3-1200 v5/E3-1500 v5/6th Gen Core Processor PCIe Controller (x16) [8086:1901] (rev 07)
IOMMU Group 1 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM204 [GeForce GTX 970] [10de:13c2] (rev a1)
IOMMU Group 1 01:00.1 Audio device [0403]: NVIDIA Corporation GM204 High Definition Audio Controller [10de:0fbb] (rev a1)
8086:1901,10de:13c2,10de:0fbb
```

all devices in the same IOMMU group will be passed at the same time

#### Add booting entry with VFIO enabled in grub menu

copy `example-etc/grub.d/11_gpu_vfio_linux` to `etc/grub.d/11_gpu_vfio_linux`, `chmod +x etc/grub.d/11_gpu_vfio_linux`, and modify it if needed:

```
# Configurations:
VFIO_WANTED_PCI_KEYWORDS = ['NVIDIA']
```

this script will find the group with `NVIDIA` from `ls_iommu.sh` output and add an entry on grub menu booting with the group passthrough

then `grub-mkconfig -o /boot/grub/grub.cfg` and reboot, choose "VFIO '...' Linux" to boot

> devices being passthrough will not be available for host OS

## Install required packages and start service

```shell=
yay -S qemu libvirt ovmf virt-manager ebtables dnsmasq bridge-utils
systemctl enable libvirtd
systemctl restart libvirtd

# if want to avoid entering password every time:
usermod -G libvirtd $USER # re-login is required
```

I encounter `Cannot check QEMU binary /usr/bin/qemu-kvm: No such file or directory`, according to [this solution](http://wood1978.dyndns.org/~wood/wordpress/2013/03/21/cannot-check-qemu-binary-usrbinqemu-kvm-no-such-file-or-directory/), just link executable:

```shell=
ln -s /usr/bin/qemu-{system-x86_64,kvm}
```

#### Enable virtual network

ensure `ebtables`, `dnsmasq` is installed, open virt-manager GUI, `Edit` -> `Connection Details` -> `Virtual Networks`, choose `default`, check `Autostart` and press the play button:

![enable-virt-network](https://i.imgur.com/tVAbzeT.png)

## Windows VM with GPU passthrough for gaming

> for sample virt vm xml see `example-etc/libvirt/qemu/win10.xml`

#### Configure libvirt

follow [this section](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Setting_up_an_OVMF-based_guest_VM)

```bash
yaourt -S qemu libvirt ovmf virt-manager
vim /etc/libvirt/qemu.conf
# nvram = [
#   "/usr/share/ovmf/x64/OVMF_CODE.fd:/usr/share/ovmf/x64/OVMF_VARS.fd"
#   ...
# ]
systemctl restart libvirtd
```

#### Create OVMF VM

Use the pretty GUI virt-manager to create vm, basically follow [this section](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Setting_up_the_guest_OS):

1. start the GUI, click `File` -> `Add Connection` -> Hypervisor: `QEMU/KVM`
2. add VM, click `File` -> `New Virtual Machine`
3. Connection: `QEMU/KVM`, Forward.
4. Use ISO Image, Browse, Browse Local, find windows install iso file, OS type: `Windows`
5. set RAM and CPU
6. Select or create custom storage, Manage, Browse Local, find `/dev/sdx` to give whole disk
7. check `Customize configuration before install`

#### OVMF UEFI firmware

Overview, set firmware to `UEFI`:

![ovmf uefi](https://i.imgur.com/HHaYN4Y.png?1)

> required, and the GPU have to support UEFI boot as well

#### CPU

CPUs, Configuration, check `Copy host CPU configuration`

#### Using whole disk

```shell=
# check and get path of the hard disk to give to windows
cd /dev/disk/by-id/
fdisk -l
ls -l
# for example:
lrwxrwxrwx 1 root root  9 Aug 16 23:20 ata-Crucial_CT240M500SSD1_XXX -> ../../sdb
```

`Add Hardware` > `Storage`

* `Select or create custom storage` > manually type `/dev/disk/by-id/ata-Crucial_CT240M500SSD1_XXX`
* `Bus type`: `VirtIO`

##### VirtIO driver is required for windows

Download virtio windows driver iso from [fedora](https://docs.fedoraproject.org/quick-docs/en-US/creating-windows-virtual-machines-using-virtio-drivers.html), Add hardware, Storage, Select or create custom storage, Manage, Browse Local, select the virtio windows driver iso, Device type: `CDROM device`, Bus type: `IDE`

Boot Options, Enable boot menu, IDE CDROM 1 first, VirtIO Disk 1 second

#### Attatch GPU to VM

* Add hardware, PCI Host Device, choose GPU
* Add hardware, PCI Host Device, choose GPU audio

#### Sound

choose model: `ac97`:

#### Install windows

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

#### Mouse and keyboard

> refer to https://passthroughpo.st/using-evdev-passthrough-seamless-vm-input/

```bash
usermod -G input pastleo
# re-login

# find devices want to pass
cd /dev/input/by-id
cat [input_dev_id] # check which one

vim /etc/libvirt/qemu.conf
# user = "username"
# cgroup_device_acl = [
#   "/dev/input/by-id/[input_dev_id]",
# ]

virsh edit vm_name
# modify top level <domain>:
# <domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
# add qemu parameter before </domain>:
# <qemu:commandline>
#   <qemu:arg value='-object'/>
#   <qemu:arg value='input-linux,id=mouse1,evdev=/dev/input/by-id/[input_dev_id]'/>
#   <qemu:arg value='-object'/>
#   <qemu:arg value='input-linux,id=kbd1,evdev=/dev/input/by-id/[input_dev_id],grab_all=on,repeat=on'/>
# </qemu:commandline>
```

reboot the vm,

### *press left ctrl and right ctrl* to switch between host and vm

#### CPU pinning

this can improve CPU performance inside VM, follow [instructions](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#CPU_pinning), example:

```xml
  <cputune>
    <vcpupin vcpu='0' cpuset='0'/>
    <vcpupin vcpu='1' cpuset='1'/>
    <vcpupin vcpu='2' cpuset='2'/>
    <vcpupin vcpu='3' cpuset='3'/>
  </cputune>
...
    <topology sockets='1' cores='4' threads='1'/>
```

#### fool windows and prevent GPU error 43

this is required for my PC, otherwise I will get GPU error 43

```xml
  <features>
    <acpi/>
    <apic/>
    <hyperv>
      <relaxed state='on'/>
      <vapic state='on'/>
      <spinlocks state='on' retries='8191'/>
+     <vendor_id state='on' value='123456789ab'/>
    </hyperv>
+   <kvm>
+     <hidden state='on'/>
+   </kvm>
    <vmport state='off'/>
  </features>
```

#### monitor setup for convenience

I tried to switch Graphic card ownership between vfio and host without reboot or restart X server [according to this post](https://arseniyshestakov.com/2016/03/31/how-to-pass-gpu-to-vm-and-back-without-x-restart/), but it did not work, I guess it is nvidia's driver problem, currently GTX970 is still not supported by nouveau...

To utilize second monitor when VM is not running, I connect monitors like this:

```
Intel Graphics --- Monitor 1
               \
                \
                 \
Graphic card   --- Monitor 2
```

When I want to start VM, disable Monitor 2 on the host and boot VM

#### Default grub boot with gpu passthrough

set default grub boot option, my vfio boot option is 3rd (first is 0):

```
grub-set-default 2
```

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

