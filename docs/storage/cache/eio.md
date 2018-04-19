# EnhanceIO disk cache

First disable kernel upgrades as it will break cache (and lost of data!)

```shell
nano /etc/yum.conf
    exclude=kernel*
```

After compiling EiO cache install modules:

```shell
modprobe enhanceio
modprobe enhanceio_lru
```
## Installation on CentOS 7.2 (kernel 3.9)

Kernel source repository:
```shell
Ref.: https://wiki.centos.org/HowTos/I_need_the_Kernel_Source
```

```shell
cd /home/user
git clone https://github.com/stec-inc/EnhanceIO.git
yum install kernel-devel gcc
```
```shell
mkdir -p ~/rpmbuild/{BUILD,BUILDROOT,RPMS,SOURCES,SPECS,SRPMS}
echo '%_topdir %(echo $HOME)/rpmbuild' > ~/.rpmmacros
yum install rpm-build redhat-rpm-config asciidoc hmaccalc perl-ExtUtils-Embed pesign xmlto 
yum install audit-libs-devel binutils-devel elfutils-devel elfutils-libelf-devel
yum install ncurses-devel newt-devel numactl-devel pciutils-devel python-devel zlib-devel
yum install bison
yum install net-tools bc
```

Download and install kernel source:
```shell
Buscar la versió de centos: rpm --query centos-release
rpm -i http://vault.centos.org/7.2.1511/updates/Source/SPackages/kernel-3.10.0-327.10.1.el7.src.rpm 2>&1 | grep -v exist
cd ~/rpmbuild/SPECS
rpmbuild -bp --target=$(uname -m) kernel.spec
```

Kernel source is in:
```shell
rpmbuild/BUILD/kernel-3.10.0-327.10.1.el7/linux-3.10.0-327.10.1.el7.x86_64/
```

```
Ref.: https://wiki.centos.org/HowTos/BuildingKernelModules
```

```shell
cd rpmbuild/BUILD/kernel-3.10.0-327.10.1.el7/linux-3.10.0-327.10.1.el7.x86_64/
make oldconfig
make menuconfig  (no canviem res en realitat)
make prepare
make modules_prepare
```

We can't build it on current folder. So, we could do an ln or build it on /usr/src/kernels:

```shell
cd /usr/src/kernels/3.10.0-327.10.1.el7.x86_64/fs
 (or ln rpmbuild/BUILD/ker... to /usr/src/ker...)
mkdir enhanceio
cd enhanceio
cp /home/user/EnhanceIO/Driver/enhanceio/* .
cd /usr/src/kernels/3.10.0-327.10.1.el7.x86_64/
```

```shell
make -C /lib/modules/`uname -r`/build M=fs/enhanceio
strip --strip-debug fs/enhanceio/*.ko
cp fs/enhanceio/*.ko /lib/modules/`uname -r`/extra
depmod -a
```

# Fedora (if LINUX_VERSION_CODE < KERNEL_VERSION(4,3,0))

```shell
yum install kernel-devel gcc
git clone https://github.com/stec-inc/EnhanceIO.git
uname -a
cd EnhanceIO/Driver/enhanceio/
make
make install
```

# Fedora (if LINUX_VERSION_CODE >= KERNEL_VERSION(4,3,0))

**NOT TESTED**
With this fork it does compile on kernels 4.3.
```
https://github.com/elmystico/EnhanceIO
```
**NOT TESTED**

