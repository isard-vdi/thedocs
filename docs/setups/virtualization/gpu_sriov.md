# GPU SRIOV with a FirePro S7150 graphics card

In order to archieve a GPU SRIOV with a FirePro S7150 card, we'll need to have a **kernel 4.4** of an **Ubuntu 16.04 server**. During the installation process, we'll need to make sure that the virtualization option is checked on.  



**TODO**:
- Check if it can be made with other distros or kernels




## 1- Modify the BIOS 

The first step is going to be modify the BIOS. We'll need to reset it to defaults, change the boot method to **legacy**, activate all the parameters related with **virtualization** and, finally, enable **SRIOV**. If the motherboard doesn't have this last option, the whole thing is not going to work. We'll have to change the **main graphics** card to the **onboard** one, since the FirePro S7150 doesn't have a video output. (`IntelRCSetup > Miscellanious Configuration > Active Video [Onboard Device]`)



## 2- Kernel compilation

In order to use the graphics cards and all its cores, we'll have to recompile the Linux Kernel, modifying some values and applying some paches.



### 2.1- Activate the `deb-src` apt repositories to be able to download the Kernel source

```bash
vim /etc/apt/sources.list
----
# See http://help.ubuntu.com/community/UpgradeNotes for how to upgrade to
# newer versions of the distribution.
deb http://es.archive.ubuntu.com/ubuntu/ xenial main restricted
deb-src http://es.archive.ubuntu.com/ubuntu/ xenial main restricted

## Major bug fix updates produced after the final release of the
## distribution.
deb http://es.archive.ubuntu.com/ubuntu/ xenial-updates main restricted
deb-src http://es.archive.ubuntu.com/ubuntu/ xenial-updates main restricted

## N.B. software from this repository is ENTIRELY UNSUPPORTED by the Ubuntu
## team. Also, please note that software in universe WILL NOT receive any
## review or updates from the Ubuntu security team.
deb http://es.archive.ubuntu.com/ubuntu/ xenial universe
deb-src http://es.archive.ubuntu.com/ubuntu/ xenial universe
deb http://es.archive.ubuntu.com/ubuntu/ xenial-updates universe
deb-src http://es.archive.ubuntu.com/ubuntu/ xenial-updates universe

## N.B. software from this repository is ENTIRELY UNSUPPORTED by the Ubuntu 
## team, and may not be under a free licence. Please satisfy yourself as to 
## your rights to use the software. Also, please note that software in 
## multiverse WILL NOT receive any review or updates from the Ubuntu
## security team.
deb http://es.archive.ubuntu.com/ubuntu/ xenial multiverse
deb-src http://es.archive.ubuntu.com/ubuntu/ xenial multiverse
deb http://es.archive.ubuntu.com/ubuntu/ xenial-updates multiverse
deb-src http://es.archive.ubuntu.com/ubuntu/ xenial-updates multiverse

## N.B. software from this repository may not have been tested as
## extensively as that contained in the main release, although it includes
## newer versions of some applications which may provide useful features.
## Also, please note that software in backports WILL NOT receive any review
## or updates from the Ubuntu security team.
deb http://es.archive.ubuntu.com/ubuntu/ xenial-backports main restricted universe multiverse
deb-src http://es.archive.ubuntu.com/ubuntu/ xenial-backports main restricted universe multiverse

## Uncomment the following two lines to add software from Canonical's
## 'partner' repository.
## This software is not part of Ubuntu, but is offered by Canonical and the
## respective vendors as a service to Ubuntu users.
# deb http://archive.canonical.com/ubuntu xenial partner
# deb-src http://archive.canonical.com/ubuntu xenial partner

deb http://security.ubuntu.com/ubuntu xenial-security main restricted
deb-src http://security.ubuntu.com/ubuntu xenial-security main restricted
deb http://security.ubuntu.com/ubuntu xenial-security universe
deb-src http://security.ubuntu.com/ubuntu xenial-security universe
deb http://security.ubuntu.com/ubuntu xenial-security multiverse
deb-src http://security.ubuntu.com/ubuntu xenial-security multiverse
----
apt update && apt upgrade -y
reboot # !!!! THIS REBOOT IS NEEDED !!!!
```



### 2.2- Now we need to download the required packages to be able to unpack the kernel. We need to download the repo that contains the `gim` module sources and the paches to apply

```bash
apt install -y dpkg-dev
apt source linux-image-$(uname -r)
git clone https://github.com/GPUOpen-LibrariesAndSDKs/MxGPU-Virtualization
```

 

### 2.3- Now we install all the packages required to compile the Kernel

```bash
apt build-dep linux-image-$(uname -r)
apt install libncurses5 libncurses5-dev
```



### 2.4- Kernel preparation and patch installation

```bash
cd linux-4.4.0
make oldconfig
make menuconfig
> Enable loadable module support --->
>> [ ] Module signature verification # This has to be unchecked. After that, we can save and exit
patch < ../MxGPU-Virtualization/patch/0001-Added-legacy-endpoint-type-to-sriov-for-ubuntu-4.4.0-75-generic.diff
File to patch: ./drivers/pci/iov.c # We need to specify the path of the file we want to patch
patch < ../MxGPU-Virtualization/patch/0002-add-pci-io-access-cap-for-ubuntu-4.4.0-75-generic.diff
File to patch: ./drivers/vfio/pci/vfio_pci_config.c # We need to specify the path of the file we want to patch
```



### 2.5- Kernel compilation

```bash
make -j <number of threads that the CPU has> deb-pkg LOCALVERSION=-vm-firepro
```



### 2.6- New Kernel installation

```bash
cd ..
dpkg -i *.deb

```



## 3- GRUB preparation

### 3.1- Add the module `amdgpu` to the blacklist (we'll use the `gim` module)

```bash
vim /etc/modprobe.d/blacklist.conf
----
...
# AMD GPU (we'll use the gim module)
blacklist amdgpu
----
update-initramfs -u
```



### 3.2- Edit the default GRUB configuration in order to  enable  IOMMU

```bash
vim /etc/default/grub
----
...
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"
...
----
update-grub
```



## 4- `gim` module compilation

The `gim` module is the module that we're going to use in order to control the graphics card. 

```bash
cd MxGPU-Virtualization
./gim.sh
```



Once it's compiled, we need to execute the following:

```bash
reboot # To apply all the GRUB configuration
modprobe gim
modprobe -r gim
nano /etc/gim_config # It needs to be nano, the vi command isn't going to work
----
...
vf_num=16 # This is going to activate the 16 cores of the GPU
...
----
modprobe gim
dmesg # Finally, we need to check that the gim module has successfully been loaded, without errors
lspci -vvv # We can now see that the FirePro S7150 has all the 16 cores activated
```
