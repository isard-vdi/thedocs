# Cluster Active-Pasive

This setup is based on two servers (NAS1 & NAS2), both with similar 
hardware and Centos 7, that will provide high availability nfs storage 
using software raids and drbd 8. We will make use of EnhanceIO disk
cache with Intel NVME disk.

# NETWORK

## NAS1 network device names
- escola: access interface
- nas: nfs exported storage
- drbd: storage synchronization

```
cat /etc/udev/rules.d/70-persistent-net.rules
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="40:8d:5c:1e:0e:0c", ATTR{type}=="1", KERNEL=="e*", NAME="escola"
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="a0:36:9f:6e:18:c0", ATTR{type}=="1", KERNEL=="e*", NAME="nas"
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="a0:36:9f:6e:18:c2", ATTR{type}=="1", KERNEL=="e*", NAME="drbd"
```
## NAS2 network device names
```
cat /etc/udev/rules.d/70-persistent-net.rules
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="40:8d:5c:1e:1c:54", ATTR{type}=="1", KERNEL=="e*", NAME="escola"
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="a0:36:9f:34:0a:c4", ATTR{type}=="1", KERNEL=="e*", NAME="nas"
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="a0:36:9f:34:0a:c6", ATTR{type}=="1", KERNEL=="e*", NAME="drbd"

[root@nas2 network-scripts]# cat ifcfg-escola
TYPE="Ethernet"
BOOTPROTO="none"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="no"
IPV6_AUTOCONF="no"
IPV6_DEFROUTE="no"
IPV6_FAILURE_FATAL="no"
NAME="escola"
ONBOOT="yes"
HWADDR="40:8d:5c:1e:1c:54"
PEERDNS="yes"
PEERROUTES="yes"
IPV6_PEERDNS="no"
IPV6_PEERROUTES="no"

IPADDR="10.1.1.32"
PREFIX="24"
GATEWAY="10.1.1.199"
DNS1="10.1.1.200"
DNS2="10.1.1.201"
DOMAIN="escoladeltreball.org"

[root@nas2 network-scripts]# cat ifcfg-nas
TYPE="Ethernet"
BOOTPROTO="none"
DEFROUTE="no"
IPV4_FAILURE_FATAL="no"
IPV6INIT="no"
IPV6_AUTOCONF="no"
IPV6_DEFROUTE="no"
IPV6_FAILURE_FATAL="no"
NAME="drbd"
ONBOOT="yes"
HWADDR=A0:36:9F:34:0A:C4
PEERDNS="no"
PEERROUTES="no"
IPV6_PEERDNS="no"
IPV6_PEERROUTES="no"

IPADDR="10.1.2.32"
PREFIX="24"
MTU="9000"

[root@nas2 network-scripts]# cat ifcfg-drbd
TYPE="Ethernet"
BOOTPROTO="none"
DEFROUTE="no"
IPV4_FAILURE_FATAL="no"
IPV6INIT="no"
IPV6_AUTOCONF="no"
IPV6_DEFROUTE="no"
IPV6_FAILURE_FATAL="no"
NAME="drbd"
ONBOOT="yes"
HWADDR=A0:36:9F:34:0A:C6
PEERDNS="no"
PEERROUTES="no"
IPV6_PEERDNS="no"
IPV6_PEERROUTES="no"

IPADDR="10.1.3.32"
PREFIX="24"
MTU="9000"
```

# OS base installation
```
dnf update -y
vi /etc/hostname
dnf install ethtool pciutils fio smartmontools wget unzip tar mdadm net-tools

dnf install tuned
systemctl start tuned
systemctl enable tuned
tuned-adm profile throughput-performance
```


## Command line for intel NVMe & s3700
```
http://www.intel.com/support/ssdc/hpssd/sb/CS-035687.htm
wget https://downloadcenter.intel.com/downloads/eula/23931/Intel-Solid-State-Drive-Data-Center-Tool?httpDown=https%3A%2F%2Fdownloadmirror.intel.com%2F23931%2Feng%2FDataCenterTool_2_3_0_Linux.zip
unzip DataCenterTool_2_3_0_Linux.zip
rpm -Uvh isdct-2.3.0.400-13.x86_64.rpm
```

### Sample commands to check NVME
```
isdct show -intelssd
isdct show -sensor -intelssd 1
```

## Initialize disks
```
dd if=/dev/zero of=/dev/sdb bs=4k count=1024
dd if=/dev/zero of=/dev/sdc bs=4k count=1024
dd if=/dev/zero of=/dev/sdd bs=4k count=1024
dd if=/dev/zero of=/dev/sde bs=4k count=1024
dd if=/dev/zero of=/dev/sdf bs=4k count=1024
dd if=/dev/zero of=/dev/sdg bs=4k count=1024
dd if=/dev/zero of=/dev/nvme0n1 bs=4k count=1024
partprobe

We could have used also: wipefs -a /dev/xxx
```

## NTP Network time server

```
yum install ntp -y
systemctl start ntpd
systemctl status ntpd
systemctl enable ntpd
```

## HARD DISKS: 

This is lsblk:
```
NAME            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda               8:0    0  37,3G  0 disk
├─sda1            8:1    0   200M  0 part /boot/efi
├─sda2            8:2    0   500M  0 part /boot
└─sda3            8:3    0  36,6G  0 part
  ├─fedora-root 253:0    0  32,9G  0 lvm  /
  └─fedora-swap 253:1    0   3,7G  0 lvm  [SWAP]
sdb               8:16   0   2,7T  0 disk
sdc               8:32   0   2,7T  0 disk
sdd               8:48   0   1,8T  0 disk
sde               8:64   0   1,8T  0 disk
sdf               8:80   0 465,8G  0 disk
sdg               8:96   0  93,2G  0 disk
nvme0n1         259:0    0 372,6G  0 disk
```

### HAD DISKS Partitions
- sdd & sde: 7200rpm, 1,8TB
- sdb & sdc: 5400rpm, 2,7TB

Scheme that we will create with raids (md) lvms and drbd.
NOTE: We have lvs under and over drbd. That allow us to resize underlaying
scheme and also redistribute storage over drbd. We also split storage to
allow moving resources on both nodes.

```
sdd1---\
		md1--\		     /---lvgroups (md1)-------drbd30------vg_groups-------lv_groups
sde1---/	  \		    /
			   vgdata--x-----lvtemplates----------drbd20------vg_templates----lv_templates
sdb1---\	  /		    \
		md2--/		     \---lvbases--------------drbd10------vg_bases--------lv_bases
sdc1---/				  \
					       \-lvoldgroups----------drbd35------vg_oldgroups----lv_oldgroups


nvme0n1p1-\					/--lv_cachegroups-----drbd31------vgcache_groups-----lvcache_groups
           \			   /
sdf1--------x----vgcache--x----lv_cachetemplates--drbd21------vgcache_templates--lvcache_templates
		   /			   \
sdg1------/					\--lv_cachebases------drbd11------vgcache_bases------lvcache_bases

And this is the cache over the previous layout:

lv_groups----------\
					EnhanceIO:groups--------->/vimet/groups
lvcache_groups-----/

lv_templates-------\
					EnhanceIO:templates------>/vimet/templates
lvcache_templates--/

lv_bases-----------\
					EnhanceIO:bases---------->/vimet/bases
lvcache_bases------/


```

## Let's create partitions
We let an small partition on the beginning of the disks to do io tests.

With parted we can check partition alignment:
```
print free
align-check opt 1
```

We do create the partitions. We check sectors to be sure they start on
the beginning of a disk sector. This will maximize io. You should check
your disks with previous commands.
```
parted /dev/sdb mklabel gpt
parted /dev/sdc mklabel gpt
parted /dev/sdd mklabel gpt
parted /dev/sde mklabel gpt
parted /dev/sdf mklabel gpt
parted /dev/sdg mklabel gpt
parted /dev/nvme0n1 mklabel gpt
parted -a optimal /dev/sdb mkpart primary ext4 4096s 2980G
parted -a optimal /dev/sdb mkpart primary ext4 2980G 100%
parted -a optimal /dev/sdc mkpart primary ext4 4096s 2980GB
parted -a optimal /dev/sdc mkpart primary ext4 2980GB 100%
parted -a optimal /dev/sdd mkpart primary ext4 4096s 1980GB
parted -a optimal /dev/sdd mkpart primary ext4 1980GB 100%
parted -a optimal /dev/sde mkpart primary ext4 4096s 1980GB
parted -a optimal /dev/sde mkpart primary ext4 1980GB 100%
parted -a optimal /dev/sdf mkpart primary ext4 2048s 495G
parted -a optimal /dev/sdf mkpart primary ext4 495GB 100%
parted -a optimal /dev/sdg mkpart primary ext4 512 94GB
parted -a optimal /dev/sdg mkpart primary ext4 94GB 99GB
parted -a optimal /dev/sdg mkpart primary ext4 99GB 100%
parted -a optimal /dev/nvme0n1 mkpart primary ext4 512 395GB
parted -a optimal /dev/nvme0n1 mkpart primary ext4 395GB 100%

reboot recommended
```

## RAIDS
Raids will allow the firsh redundancy on system. We create level 1 raids
that will provide 2x read io speed and data duplication.
```
mdadm --create /dev/md1 --level=1 --raid-devices=2 /dev/sdd1 /dev/sde1
mdadm --create /dev/md2 --level=1 --raid-devices=2 /dev/sdb1 /dev/sdc1
mdadm --detail --scan >> /etc/mdadm.conf
```

## LVMs non-clustered (local)
### Device filtering
We need to set the devices where lvm will look for lvm signatures.
```
filter = ["a|sd.*|", "a|nvme.*|", "a|md.*|", "a|drbd.*|", "r|.*|"]
```

### Scheme pv/vg/lv (under drbd)
```
pvcreate /dev/md1
pvcreate /dev/md2

vgcreate -cn vgdata /dev/md1 /dev/md2
lvcreate -l 100%FREE -n lvgroups vgdata /dev/md1
lvcreate -L 1700G -n lvoldgroups vgdata /dev/md2
lvcreate -l 100%FREE -n lvtemplates vgdata /dev/md2

pvcreate /dev/nvme0n1p1
pvcreate /dev/sdf1
pvcreate /dev/sdg1
vgcreate vgcache /dev/nvme0n1p1 /dev/sdf1 /dev/sdg1
lvcreate -l 100%FREE -n lvcachebases vgcache /dev/sdg1
lvcreate -l 100%FREE -n lvcachetemplates vgcache /dev/sdg1
lvcreate -l 100%FREE -n lvcachegroups vgcache /dev/nvme0n1p1
```

## DRBD
Install drbd 8
```
yum install drbd drbd-utils drbd-udev drbd-pacemaker -y
modprobe drbd
systemctl enable drbd
```
Create drbd.conf and resource files.

### DRBD config files

[root@nas1 ~]# cat /etc/drbd.conf 
```
# You can find an example in  /usr/share/doc/drbd.../drbd.conf.example

include "drbd.d/global_common.conf";
include "drbd.d/*.res";
```

[root@nas1 ~]# cat /etc/drbd.d/global_common.conf 
```
# DRBD is the result of over a decade of development by LINBIT.
# In case you need professional services for DRBD or have
# feature requests visit http://www.linbit.com

global {
}

common {
        handlers {
                fence-peer "/usr/lib/drbd/crm-fence-peer.sh";
                after-resync-target "/usr/lib/drbd/crm-unfence-peer.sh";
        }

        startup {
        }

        options {
        }

        disk {
                #al-extents 3389;
                #disk-barrier no;
                #disk-flushes no;
                #fencing resource-and-stonith;
                fencing resource-only;
        }

        net {
                max-buffers 16000;
                max-epoch-size 16000;
                #unplug-watermark 16;
                #sndbuf-size 2048;
                allow-two-primaries;
                after-sb-0pri discard-zero-changes;
                after-sb-1pri discard-secondary;
                after-sb-2pri disconnect;
        }

        syncer {
         }
}
```

[root@nas1 ~]# cat /etc/drbd.d/bases.res 
```
resource bases {
         startup {
                #become-primary-on both;
         }
        volume 0 {
                device    /dev/drbd11;
                disk      /dev/vgcache/lvcachebases;
                meta-disk internal;
        }

        volume 1 {
                device    /dev/drbd10;
                disk      /dev/vgdata/lvbases;
                meta-disk internal;
        }
    on nas1 {
         address   10.1.3.31:7810;
    }
    on nas2 {
         address   10.1.3.32:7810;
    }
}
```

[root@nas1 ~]# cat /etc/drbd.d/templates.res 
```
resource templates {
         startup {
                #become-primary-on both;
         }
   volume 0 {
         device    /dev/drbd21;
         disk      /dev/vgcache/lvcachetemplates;
         meta-disk internal;
   }
   volume 1 {
         device    /dev/drbd20;
         disk      /dev/vgdata/lvtemplates;
         meta-disk internal;
   }
    on nas1 {
         address   10.1.3.31:7820;
    }
    on nas2 {
         address   10.1.3.32:7820;
    }
}
```

[root@nas1 ~]# cat /etc/drbd.d/groups.res 
```
resource groups {
         startup {
                #become-primary-on both;
         }
        volume 0 {
                device    /dev/drbd31;
                disk      /dev/vgcache/lvcachegroups;
                meta-disk internal;
        }

        volume 1 {
                device    /dev/drbd30;
                disk      /dev/vgdata/lvgroups;
                meta-disk /dev/sdg3;
        }

        volume 2 {
                device    /dev/drbd35;
                disk      /dev/vgdata/lvoldgroups;
                meta-disk internal;
        }
    on nas1 {
         address   10.1.3.31:7830;
    }
    on nas2 {
         address   10.1.3.32:7830;
    }
}
```

Now we can create drbd resources.

```
drbdadm create-md bases
drbdadm create-md templates
drbdadm create-md groups
drbdadm up bases
drbdadm up templates
drbdadm up grups
drbdadm primary bases --force
drbdadm primary templates --force
drbdadm primary grups --force

drbdadm disk-options --resync-rate=400M <resource>
```

# LVM Nested volumes (now over drbd)

This lvm setup has to be done in only one node:

## Groups
```
pvcreate /dev/drbd30
pvcreate /dev/drbd31
vgcreate vgcache_groups /dev/drbd31
lvcreate -l 100%FREE -n lvcache_groups vgcache_groups
lvcreate -l 100%FREE -n lvcache_templates vgcache_templates
vgcreate vg_groups /dev/drbd20
lvcreate -l 100%FREE -n lv_groups vg_groups
vgchange -ay vg_groups
```

## Templates
```
pvcreate /dev/drbd20
pvcreate /dev/drbd21
vgcreate vgcache_templates /dev/drbd21
lvcreate -l 100%FREE -n lvcache_templates vgcache_templates
vgcreate vg_templates /dev/drbd20
lvcreate -l 100%FREE -n lv_templates vg_templates
vgchange -ay vg_templates
```
## Bases
```
pvcreate /dev/drbd10
pvcreate /dev/drbd11
vgcreate vgcache_bases /dev/drbd11
lvcreate -l 100%FREE -n lvcache_bases vgcache_bases
vgcreate vg_bases /dev/drbd10
lvcreate -l 100%FREE -n lv_bases vg_bases
vgchange -ay vg_templates
```

## OldGroups
```
pvcreate /dev/drbd35
vgcreate vg_oldgroups /dev/drbd35
lvcreate -l 100%FREE -n lv_oldgroups vg_oldgroups
```

# Filesystem

```
mkfs.ext4 /dev/vg_bases/lv_bases
mkfs.ext4 /dev/vg_templates/lv_templates
mkfs.ext4 /dev/vg_groups/lv_groups
mkfs.ext4 /dev/vg_oldgroups/lv_oldgroups
```
** Now you can try to mount lvs to check everything was fine.

## EnhanceIO Disk cache
First we install it:
```
dnf install kernel-devel gcc
git clone https://github.com/stec-inc/EnhanceIO.git
cd EnhanceIO/Driver/enhanceio
make && make install

cd /root/EnhanceIO/CLI
```
Apply patch for kernels >3:
```
diff --git a/CLI/eio_cli b/CLI/eio_cli
index 3453b35..a0b60c5 100755
--- a/CLI/eio_cli
+++ b/CLI/eio_cli
@@ -1,4 +1,4 @@
-#!/usr/bin/python
+#!/usr/bin/python2
 #
 # Copyright (C) 2012 STEC, Inc. All rights not specifically granted
 # under a license included herein are reserved
@@ -305,10 +305,12 @@ class Cache_rec(Structure):

        def do_eio_ioctl(self,IOC_TYPE):
                #send ioctl to driver
-               fd = open(EIODEV, "r")
+               libc = CDLL('libc.so.6')
+               fd = os.open (EIODEV, os.O_RDWR, 0400)
                fmt = ''
+               selfaddr = c_uint64(addressof(self))
                try:
-                       if ioctl(fd, IOC_TYPE, addressof(self)) == SUCCESS:
+                       if libc.ioctl(fd, IOC_TYPE, selfaddr) == SUCCESS:
                                return SUCCESS
                except Exception as e:
                        print e
```

Copy binary to path folder and install modules in kernel:
```
cp EnhanceIO/CLI/eio_cli /sbin

modprobe enhanceio
modprobe enhanceio_fifo  x
modprobe enhanceio_lru
modprobe enhanceio_rand  x
```

Persistent modules install:
```
vi /etc/modules-load.d/enhanceio.conf
	enhanceio
	enhanceio_lru
```
To stop cache device we have to put it in read only and wait till all
dirty sectors from nvme are layered down to rotational hard disks.
```
eio_cli edit -c data -m ro
```
Avoid kernel updates as it will break enhanceio and you could lose all
your data!.
```
vi /etc/dnf/dnf.conf
	exclude=kernel*
```
We should create caches ONLY on primary drbd node and copy udev rules
created to the other nas. (/etc/udev/rules.d/94...):
```
eio_cli create -d /dev/vg_groups/lv_groups -s /dev/vgcache_groups/lvcache_groups -p lru -m wb -c groups
eio_cli create -d /dev/vg_templates/lv_templates -s /dev/vgcache_templates/lvcache_templates -p lru -m wb -c templates
eio_cli create -d /dev/vg_bases/lv_bases -s /dev/vgcache_bases/lvcache_bases -p lru -m wt -c bases
```
Check cache status:
```
eio_cli info
cat /proc/enhanceio/<cache_name>/stats
```


# PACEMAKER Cluster
Pacemaker will keep our resources under monitoring and will restart it on
the other node in case of failure.

```
dnf install corosync pacemaker pcs -y
systemctl enable pcsd; systemctl enable corosync; systemctl enable pacemaker;
systemctl start pcsd;
#passwd hacluster  (habitual)
pcs cluster auth nas1 nas2  (hacluster/habitual)
pcs cluster setup --name vimet_cluster nas1 nas2
pcs cluster start --all
pcs status

pcs property set default-resource-stickiness=200
pcs property set no-quorum-policy=ignore
```

### To remove an existing cluster:
```
pcs cluster stop --all
pcs cluster destroy 
rm -rf /var/lib/pcsd/*
reboot
```

## Fence agents
Install fence agents. A pacemaker setup CAN'T work without a fencing
device. We used APC stonith device:
 
```
yum install python-pycurl fence-agents-apc fence-agents-apc-snmp -y

```

Fence agent (stonith) pacemaker resource definition:
```
pcs cluster cib fence_cfg
pcs -f fence_cfg stonith create stonith fence_apc_snmp params ipaddr=10.1.1.4 pcmk_host_list="nas1,nas2" pcmk_host_map="nas1:3;nas2:4" pcmk_host_check=static-list power_wait=5 inet4only debug=/var/log/stonith.log retry_on=10
pcs cluster cib-push fence_cfg

pcs property set stonith-enabled=true
pcs property set start-failure-is-fatal=false

```

## DRBD resources

```
pcs cluster cib drbd_bases_cfg
pcs -f drbd_bases_cfg resource create drbd_bases ocf:linbit:drbd drbd_resource=bases op monitor interval=15s role=Master op monitor interval=30s role=Slave
pcs -f drbd_bases_cfg resource master drbd_bases-clone drbd_bases master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 notify=true
pcs cluster cib-push drbd_bases_cfg

pcs cluster cib drbd_templates_cfg
pcs -f drbd_templates_cfg resource create drbd_templates ocf:linbit:drbd drbd_resource=templates op monitor interval=15s role=Master op monitor interval=30s role=Slave
pcs -f drbd_templates_cfg resource master drbd_templates-clone drbd_templates master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 notify=true
pcs cluster cib-push drbd_templates_cfg

pcs cluster cib drbd_groups_cfg
pcs -f drbd_groups_cfg resource create drbd_groups ocf:linbit:drbd drbd_resource=groups op monitor interval=15s role=Master op monitor interval=30s role=Slave
pcs -f drbd_groups_cfg resource master drbd_groups-clone drbd_groups master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 notify=true
pcs cluster cib-push drbd_groups_cfg
```

## LVM resources
```
pcs resource create lv_bases ocf:heartbeat:LVM volgrpname=vg_bases op monitor interval=30s
pcs resource create lv_templates ocf:heartbeat:LVM volgrpname=vg_templates op monitor interval=30s
pcs resource create lv_groups ocf:heartbeat:LVM volgrpname=vg_groups op monitor interval=30s
pcs resource create lv_oldgroups ocf:heartbeat:LVM volgrpname=vg_oldgroups op monitor interval=30s
pcs resource create lvcache_bases ocf:heartbeat:LVM volgrpname=vgcache_bases op monitor interval=30s
pcs resource create lvcache_templates ocf:heartbeat:LVM volgrpname=vgcache_templates op monitor interval=30s
pcs resource create lvcache_groups ocf:heartbeat:LVM volgrpname=vgcache_groups op monitor interval=30s
```


## Filesystem ext4 and mountpoints
```
pcs resource create ext4_bases Filesystem device="/dev/vg_bases/lv_bases" directory="/vimet/bases" fstype="ext4" "options=defaults,noatime,nodiratime,noquota" op monitor interval=10s
pcs resource create ext4_templates Filesystem device="/dev/vg_templates/lv_templates" directory="/vimet/templates" fstype="ext4" "options=defaults,noatime,nodiratime,noquota" op monitor interval=10s
pcs resource create ext4_groups Filesystem device="/dev/vg_groups/lv_groups" directory="/vimet/groups" fstype="ext4" "options=defaults,noatime,nodiratime,noquota" op monitor interval=10s
pcs resource create ext4_oldgroups Filesystem device="/dev/vg_oldgroups/lv_oldgroups" directory="/vimet/oldgroups" fstype="ext4" "options=defaults,noatime,nodiratime,noquota" op monitor interval=10s
```

## NFS server

Version 4
## NFS: server and root
```
pcs cluster cib nfsserver_cfg
pcs -f nfsserver_cfg resource create nfs-daemon systemd:nfs-server \
nfs_shared_infodir=/nfsshare/nfsinfo nfs_no_notify=true op monitor interval=30s \
--group nfs_server
pcs -f nfsserver_cfg resource create nfs-root exportfs \
clientspec=10.1.0.0/255.255.0.0 \
options=rw,crossmnt,async,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash \
directory=/vimet \
fsid=0 \
--group nfs_server
pcs cluster cib-push nfsserver_cfg
pcs resource clone nfs_server master-max=2 master-node-max=1 clone-max=2 clone-node-max=1 on-fail=restart notify=true resource-stickiness=0
```

## NFS: Exports

```
pcs cluster cib exports_cfg
pcs -f exports_cfg resource create nfs_bases exportfs \
clientspec=10.1.0.0/255.255.0.0 \
wait_for_leasetime_on_stop=true \
options=rw,mountpoint,async,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash directory=/vimet/bases \
fsid=11 \
op monitor interval=30s

pcs -f exports_cfg resource create nfs_templates exportfs \
clientspec=10.1.0.0/255.255.0.0 \
wait_for_leasetime_on_stop=true \
options=rw,mountpoint,async,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash directory=/vimet/templates \
fsid=20 \
op monitor interval=30s

pcs -f exports_cfg resource create nfs_groups exportfs \
clientspec=10.1.0.0/255.255.0.0 \
wait_for_leasetime_on_stop=true \
options=rw,async,mountpoint,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash directory=/vimet/groups \
fsid=30 \
op monitor interval=30s

pcs -f exports_cfg resource create nfs_oldgroups exportfs \
clientspec=10.1.0.0/255.255.0.0 \
wait_for_leasetime_on_stop=true \
options=rw,async,mountpoint,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash directory=/vimet/oldgroups \
fsid=35 \
op monitor interval=30s
pcs cluster cib-push exports_cfg
```

## Floating IPs
Those IPs will 'move' always with assigned resources.
```
pcs resource create IPbases ocf:heartbeat:IPaddr2 ip=10.1.2.210 cidr_netmask=32 nic=nas:0  op monitor interval=30 
pcs resource create IPtemplates ocf:heartbeat:IPaddr2 ip=10.1.2.211 cidr_netmask=32 nic=nas:0  op monitor interval=30 
pcs resource create IPgroups ocf:heartbeat:IPaddr2 ip=10.1.2.212 cidr_netmask=32 nic=nas:0 op monitor interval=30
pcs resource create IPvbases ocf:heartbeat:IPaddr2 ip=10.1.1.28 cidr_netmask=32 nic=escola:0 op monitor interval=30
pcs resource create IPvtemplates ocf:heartbeat:IPaddr2 ip=10.1.1.29 cidr_netmask=32 nic=escola:0 op monitor interval=30
pcs resource create IPvgroups ocf:heartbeat:IPaddr2 ip=10.1.1.30 cidr_netmask=32 nic=escola:0 op monitor interval=30
```


# Groups and constraints
Groups will allow starting and stopping resources in order.
Constraings will allow restriction definitions.

### bases
```
pcs resource group add bases \
	lvcache_bases lv_bases ext4_bases nfs_bases IPbases IPvbases 

pcs constraint order \
	promote drbd_bases-clone then bases INFINITY \
	require-all=true symmetrical=true \
	setoptions kind=Mandatory \
	id=o_drbd_bases
	
pcs constraint colocation add \
	bases with master drbd_bases-clone INFINITY \
	id=c_drbd_bases
	
pcs constraint order \
	nfs_server-clone then bases INFINITY \
	require-all=true symmetrical=true \
	setoptions kind=Mandatory \
	id=o_nfs_bases
	
pcs constraint colocation add \
	bases with started nfs_server-clone INFINITY \
	id=c_nfs_bases
```

### templates
```
pcs resource group add templates \
	lvcache_templates lv_templates ext4_templates nfs_templates IPtemplates IPvtemplates 

pcs constraint order \
	promote drbd_templates-clone then templates INFINITY \
	require-all=true symmetrical=true \
	setoptions kind=Mandatory \
	id=o_drbd_templates
	
pcs constraint colocation add \
	templates with master drbd_templates-clone INFINITY \
	id=c_drbd_templates
	
pcs constraint order \
	nfs_server-clone then templates INFINITY \
	require-all=true symmetrical=true \
	setoptions kind=Mandatory \
	id=o_nfs_templates
	
pcs constraint colocation add \
	templates with started nfs_server-clone INFINITY \
	id=c_nfs_templates
```

### groups
```
pcs resource group add groups \
	lvcache_groups lv_groups lv_oldgroups ext4_groups ext4_oldgroups nfs_groups nfs_oldgroups IPgroups IPvgroups

pcs constraint order \
	promote drbd_groups-clone then groups INFINITY \
	require-all=true symmetrical=true \
	setoptions kind=Mandatory \
	id=o_drbd_groups
	
pcs constraint colocation add \
	groups with master drbd_groups-clone INFINITY \
	id=c_drbd_groups
	
pcs constraint order \
	nfs_server-clone then groups INFINITY \
	require-all=true symmetrical=true \
	setoptions kind=Mandatory \
	id=o_nfs_groups
	
pcs constraint colocation add \
	groups with started nfs_server-clone INFINITY \
	id=c_nfs_groups
```

This constraints will allow for resources to be put on a preferred node.
```
pcs constraint location lv_groups prefers nas2=50
pcs constraint location lv_oldgroups prefers nas2=50
pcs constraint location lv_bases prefers nas1=50
pcs constraint location lv_templates prefers nas1=50
```



# Pacemaker utils
## Manage/Unmanage resource

Unmanage drbd resource.
```
pcs resource unmanage drbd_templates-clone
```
Let the cluster control a drbd resource again:
```
pcs resource meta drbd_templates-clone is-managed=true
``` 

## Resize volumes under drbd
Stop cluster and start drbd manually.
```
pcs cluster stop/standby --all
modprobe drbd
```
Check that drbd resources are not active (cat /proc/drbd)

# CACHE: lvcachebases (ro), lvcachetemplates (wb), lvcachegroups (wb)

1.- Cache set to read only
```
eio_cli edit -c groups -m ro
```

2.- Wait till there are no dirty sectors (till it is 0)
```
grep nr_dirty /proc/enhanceio/groups/stats
```

3.- Remove cache device
```
eio_cli delete -c groups
```

4.- For example we do reduce lvcachegroups (600G: nvme0n1p1+/dev/sdf1)
    and will grow lvcachetemplates to the new free space.
    
    Shrink nvme0n1p1:
```
		lvreduce -L 100G /dev/vgcache/lvcachegroups
```
	Growing nvme0n1p1
```
		lvresize -l 100%FREE -n /dev/vgcache/lvcachegroups /dev/nvme0n1p1
```
    Extend template cache volume to the free space we just created.
```
		lvextend -l +100%FREE /dev/vgcache/lvcachetemplates
```

Now drbd21 (resource groups volume 0) will be in 'diskless' state. This
is normal as we broked drbd filesystem inside resized volumes.

Now drbd21 (resource templates volume 0) will be synchronizing, as we
growed it, but it still needs to resize manually.

This will be applied on reduced volumes:
```
wipefs -a /dev/vgcache/lvcachegroups
drbdadm create-md groups/0
drbdadm up groups
drbdadm invalidate-remote groups/0
```
Now we can create EnhanceIO groups cache again.

This will be applied on growed volumes:
```
drbdadm resize templates/0
```
Now we can create EnhanceIO templates cache again.

For filesystems to be resized use resizefs for ext4.

### Credits:

https://github.com/stec-inc/EnhanceIO
https://www.suse.com/documentation/sle_ha/singlehtml/book_sleha_techguides/book_sleha_techguides.html
https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Power_Management_Guide/ALPM.html

       
_______________________________________-
