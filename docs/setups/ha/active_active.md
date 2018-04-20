# Active - Active cluster

In this setup we will make use of two servers (vserver4 & vserver5), both 
with similar hardware and software. Software raids, drbd 8 dual primary 
and pacemaker cluster control with GFS2 cluster filesystem.
The OS used was Fedora 22.


# Networking
Rename interfaces
```
new_name=escola; 
old_name=enp3s0; 
echo SUBSYSTEM==\"net\", ACTION==\"add\", DRIVERS==\"?*\", \
ATTR{address}==\"$(cat /sys/class/net/$old_name/address)\", \
ATTR{type}==\"1\", KERNEL==\"e*\", \
NAME=\"$new_name\" >> /etc/udev/rules.d/70-persistent-net.rules

new_name=drbd; 
old_name=enp1s0; 
echo SUBSYSTEM==\"net\", ACTION==\"add\", DRIVERS==\"?*\", \
ATTR{address}==\"$(cat /sys/class/net/$old_name/address)\", \
ATTR{type}==\"1\", KERNEL==\"e*\", \
NAME=\"$new_name\" >> /etc/udev/rules.d/70-persistent-net.rules

new_name=dual0; 
old_name=enp2s0f0; 
echo SUBSYSTEM==\"net\", ACTION==\"add\", DRIVERS==\"?*\", \
ATTR{address}==\"$(cat /sys/class/net/$old_name/address)\", \
ATTR{type}==\"1\", KERNEL==\"e*\", \
NAME=\"$new_name\" >> /etc/udev/rules.d/70-persistent-net.rules

new_name=dual1; 
old_name=enp2s0f1; 
echo SUBSYSTEM==\"net\", ACTION==\"add\", DRIVERS==\"?*\", \
ATTR{address}==\"$(cat /sys/class/net/$old_name/address)\", \
ATTR{type}==\"1\", KERNEL==\"e*\", \
NAME=\"$new_name\" >> /etc/udev/rules.d/70-persistent-net.rules
```

Network interface configurations

```

[root@vserver4 ~]# cat /etc/sysconfig/network-scripts/ifcfg-escola 
TYPE="Ethernet"
BOOTPROTO="none"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="no"
IPV6_AUTOCONF="no"
IPV6_DEFROUTE="no"
IPV6_FAILURE_FATAL="no"
ONBOOT="yes"
PEERDNS="yes"
PEERROUTES="yes"
IPV6_PEERDNS="no"
IPV6_PEERROUTES="no"

NAME="escola"
IPADDR="10.1.1.24"
PREFIX="24"
GATEWAY="10.1.1.199"
DNS1="10.1.1.200"
DNS2="10.1.1.201"
DOMAIN="escoladeltreball.org"

[root@vserver5 ~]# cat /etc/sysconfig/network-scripts/ifcfg-escola 
TYPE="Ethernet"
BOOTPROTO="none"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="no"
IPV6_AUTOCONF="no"
IPV6_DEFROUTE="no"
IPV6_FAILURE_FATAL="no"
ONBOOT="yes"
PEERDNS="yes"
PEERROUTES="yes"
IPV6_PEERDNS="no"
IPV6_PEERROUTES="no"

NAME="escola"
IPADDR="10.1.1.25"
PREFIX="24"
GATEWAY="10.1.1.199"
DNS1="10.1.1.200"
DNS2="10.1.1.201"
DOMAIN="escoladeltreball.org"

[root@vserver4 ~]# cat /etc/sysconfig/network-scripts/ifcfg-drbd 
TYPE="Ethernet"
BOOTPROTO="none"
DEFROUTE="no"
IPV4_FAILURE_FATAL="no"
IPV6INIT="no"
IPV6_AUTOCONF="no"
IPV6_DEFROUTE="no"
IPV6_FAILURE_FATAL="no"
ONBOOT="yes"
PEERDNS="no"
PEERROUTES="no"
IPV6_PEERDNS="no"
IPV6_PEERROUTES="no"

NAME="drbd"
IPADDR="10.1.3.24"
PREFIX="24"
MTU="9000"

[root@vserver5 ~]# cat /etc/sysconfig/network-scripts/ifcfg-drbd 
TYPE="Ethernet"
BOOTPROTO="none"
DEFROUTE="no"
IPV4_FAILURE_FATAL="no"
IPV6INIT="no"
IPV6_AUTOCONF="no"
IPV6_DEFROUTE="no"
IPV6_FAILURE_FATAL="no"
ONBOOT="yes"
PEERDNS="no"
PEERROUTES="no"
IPV6_PEERDNS="no"Tuneando fedora
IPV6_PEERROUTES="no"

NAME="drbd"
IPADDR="10.1.3.25"
PREFIX="24"
MTU="9000"

```

# Partitioning and raids

- 1 SSD for OS
- 1 disco intel gama alta ssd de 100GB para hacer de cache
- 3 500MB hard disks (raid 1)

Format disks and create partitions
```
    parted -a optimal -s /dev/sda mklabel msdos
    parted -a optimal -s /dev/sdc mklabel msdos
    parted -a optimal -s /dev/sdd mklabel msdos
    parted -a optimal -s /dev/sde mklabel msdos

    [root@vserver4 ~]# lsblk 
    NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
    sda      8:0    0  93,2G  0 disk 
    sdb      8:16   0  55,9G  0 disk 
    ├─sdb1   8:17   0   500M  0 part /boot
    ├─sdb2   8:18   0   5,6G  0 part [SWAP]
    ├─sdb3   8:19   0  33,5G  0 part /
    ├─sdb4   8:20   0     1K  0 part 
    └─sdb5   8:21   0  16,4G  0 part /home
    sdc      8:32   0 931,5G  0 disk 
    sdd      8:48   0 931,5G  0 disk 
    sde      8:64   0 931,5G  0 disk 


    [root@vserver4 ~]# parted -s /dev/sdc print
    Model: ATA TOSHIBA DT01ACA0 (scsi)
    Disk /dev/sdc: 500GB
    Sector size (logical/physical): 512B/4096B
    Partition Table: msdos
    Disk Flags: 

    Number  Start   End    Size   Type     File system  Flags
     1      1049kB  500GB  500GB  primary               raid
```

## Adjust SSD disks parameters

Disable swap and avoid writes on read (noatime):

```
    echo "vm.swappiness=1" >> /etc/sysctl.d/99-sysctl.conf
```

And in fstab:
```    
    noatime,nodiratime,discard
```    

## Create raid 1
```
    mdadm --create /dev/md0 --level=mirror --raid-devices=2 /dev/sdc1 /dev/sdd1 --spare-devices=1 /dev/sde1 
    cat /proc/mdstat 
```

## Create LVM cache (over raid)

### Volumes
```
    pvcreate /dev/md0
    vgcreate vg_data /dev/md0
    pvcreate /dev/sdb1
```

### Cache
```
    vgextend vg_data /dev/sdb1
    lvcreate -L 2G -n lv_cache_meta vg_data /dev/sdb1
    lvcreate -L 88G -n lv_cache_data vg_data /dev/sdb1
    lvcreate -l 100%FREE -n lv_data vg_data /dev/md0
    lvconvert --yes --type cache-pool --cachemode writeback --poolmetadata vg_data/lv_cache_meta vg_data/lv_cache_data
    lvconvert --type cache --cachepool vg_data/lv_cache_data vg_data/lv_data

    lsblk 
    lvdisplay 
    lvdisplay -a
```

# Fedora tuning

## Firewall
```
    systemctl stop firewalld
    systemctl disable firewalld
```
## Selinux
```
    setenforce 0
    sed -i s/SELINUX=enforcing/SELINUX=permissive/ /etc/sysconfig/selinux
    sed -i s/SELINUX=enforcing/SELINUX=permissive/ /etc/selinux/config
    sestatus 
```
## Package utilities
```
    yum -y install vim git tmux
    yum -y update
```

## Network Time Protocol
```
yum -y install ntp
systemctl start ntpd
systemctl status ntpd
systemctl enable ntpd
date
```

## Bash history tuning
```
    cat >> .bashrc << "EOF"

    # bash_history infinite
    export HISTFILESIZE=
    export HISTSIZE=
    export HISTTIMEFORMAT="[%F %T] "

    # Avoid duplicates
    export HISTCONTROL=ignoredups:erasedups  
    # When the shell exits, append to the history file instead of overwriting it
    shopt -s histappend

    # After each command, append to the history file and reread it
    export PROMPT_COMMAND="${PROMPT_COMMAND:+$PROMPT_COMMAND$'\n'}history -a; history -c; history -r"

    alias history_cleaned="cat .bash_history |grep -a -v ^'#'"

    export TMOUT=3600

    EOF
```
# DRBD 8

## Install packages

```
    yum -y install drbd drbd-bash-completion drbd-utils 
```

## Configuration files

Get the samples from installed packages:
```
    cp -a /etc/drbd.conf /root/drbd.conf.dist.f22
    cp -a /etc/drbd.d/global_common.conf /root/drbd_global_common.conf.dist.f22
```

drbd.conf:
```
    global {
            usage-count yes;
    }

    common {
            handlers {
            }

            startup {
            }

            options {
            }

            disk {
            }

            net {
                    protocol C;

                    allow-two-primaries;
                    after-sb-0pri discard-zero-changes;
                    after-sb-1pri discard-secondary;
                    after-sb-2pri disconnect;
            }
    }
```

Resources: /etc/drbd.d/vdisks.res
```
	resource vdisks {
		 device    /dev/drbd0;
		 disk      /dev/vg_data/lv_data;
		 meta-disk internal;
		on vserver4 {
		 address   10.1.3.24:7789;
		}
		on vserver5 {
		 address   10.1.3.25:7789;
		}
	}
```
We create drbdmetadata

```
    drbdadm create-md vdisks

    [...]
    Writing meta data...
    New drbd meta data block successfully created.
    success
```

We can 'dry-run' adjust to check config files:
```
    drbdadm -d adjust all
```
The we can execute:
```    
    drbdadm adjust all
```

Verify on both servers:
```
    [root@vserver4 ~]# cat /proc/drbd 
    version: 8.4.5 (api:1/proto:86-101)
    srcversion: 5A4F43804B37BB28FCB1F47 
     0: cs:Connected ro:Secondary/Secondary ds:Inconsistent/Inconsistent C r-----
        ns:0 nr:0 dw:0 dr:0 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:488236452
```

We can force primary on one server (vserver4):
```
    drbdadm primary --force vdisks
```

And we do it again on the other server (vserver5) as we want a dual
primary configuration:
```
    drbdadm primary vdisks

    [root@vserver4 ~]# cat /proc/drbd 
    version: 8.4.5 (api:1/proto:86-101)
    srcversion: 5A4F43804B37BB28FCB1F47 
     0: cs:SyncSource ro:Primary/Primary ds:UpToDate/Inconsistent C r-----
        ns:17322104 nr:0 dw:0 dr:17323016 al:0 bm:0 lo:2 pe:2 ua:2 ap:0 ep:1 wo:f oos:470916516
        [>....................] sync'ed:  3.6% (459876/476792)M
        finish: 3:03:06 speed: 42,844 (36,616) K/sec
```

# Cluster

## Install pacemaker packages

Fence agents
```
    yum -y install fence-agents-apc fence-agents-apc-snmp
    fence_apc --help
    fence_apc_snmp --help
```

Pacemaker
```
    dnf -y install corosync pcs pacemaker pacemaker-doc
```

Pacemaker drbd resource
```
    dnf -y install drbd-pacemaker 
```

Packages needed for gs2 filesystem (needs cluster lock control)    
```
    dnf -y install gfs2-utils lvm2-cluster dlm
```    

## Starting and configuring cluster
```
    systemctl start pcsd
    systemctl enable pcsd
```    
```    
    passwd hacluster
```    

Host name resolution must be set in /etc/hosts
```
vserver4:
    echo "vserver4" > /etc/hostname
    echo "10.1.1.24   vserver4" >> /etc/hosts
    echo "10.1.1.25   vserver5" >> /etc/hosts
    exit
```
```
vserver5: 

    echo "10.1.1.24   vserver4" >> /etc/hosts
    echo "10.1.1.25   vserver5" >> /etc/hosts
    echo "vserver5" > /etc/hostname
    exit
```

In one node only!:
```
    server1=vserver4
    server2=vserver5
    cl_name=vservers
    
    pcs cluster auth $server1 $server2
    pcs cluster setup --name $cl_name $server1 $server2
    pcs cluster start --all
    pcs status

    [root@vserver4 ~]#     pcs status
    Cluster name: vservers
    WARNING: no stonith devices and stonith-enabled is not false
    WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
    Last updated: Sun Oct 25 23:37:34 2015		Last change: 
    Stack: unknown
    Current DC: NONE
    0 nodes and 0 resources configured


    Full list of resources:


    PCSD Status:
      vserver4 member (vserver4): Online
      vserver5 member (vserver5): Online

    Daemon Status:
      corosync: active/disabled
      pacemaker: active/disabled
      pcsd: active/enabled
```

Check that cluster config is loaded as expected
```
    [root@vserver4 ~]# pcs cluster cib |grep vserver
    <cib crm_feature_set="3.0.10" validate-with="pacemaker-2.3" epoch="5" num_updates="8" admin_epoch="0" cib-last-written="Sun Oct 25 23:52:24 2015" update-origin="vserver5" update-client="crmd" update-user="hacluster" have-quorum="1" dc-uuid="2">
            <nvpair id="cib-bootstrap-options-cluster-name" name="cluster-name" value="vservers"/>
          <node id="1" uname="vserver4"/>
          <node id="2" uname="vserver5"/>
        <node_state id="2" uname="vserver5" in_ccm="true" crmd="online" crm-debug-origin="do_state_transition" join="member" expected="member">
        <node_state id="1" uname="vserver4" in_ccm="true" crmd="online" crm-debug-origin="do_state_transition" join="member" expected="member">
    [root@vserver4 ~]# grep vserver /etc/corosync/corosync.conf 
    cluster_name: vservers
            ring0_addr: vserver4
            ring0_addr: vserver5
```

## Fencing 

```
    [root@vserver4 ~]# pcs stonith list 
    fence_apc - Fence agent for APC over telnet/ssh
    fence_apc_snmp - Fence agent for APC, Tripplite PDU over SNMP
```
```
    pcs stonith describe fence_apc_snmp
```

You can check if your stonith is reacheable and working:
```
	fence_apc_snmp --ip=stonith1 --action=monitor
	fence_apc_snmp --ip=stonith1 --action=monitor --community=escola2015
	fence_apc_snmp --ip=stonith1 --action=reboot --plug=6  --community=escola2015 --power-wait=5
```	

Configure stonith resources as ssh (discarded as it is too slow)
```
    #pwd1=$(cat /root/pwd1)
    #pwd2=$(cat /root/pwd2)

	pcs stonith delete stonith1

    pcs cluster cib stonith_cfg

    
    pcs -f stonith_cfg stonith create stonith1 fence_apc ipaddr=10.1.1.3 login=vservers passwd=$pwd1 pcmk_host_list="vserver4 vserver5" pcmk_host_map="vserver4:4;vserver5:5"
    #pcs -f stonith_cfg stonith create  stonith2 fence_apc ipaddr=10.1.1.3 login=vserver5 passwd=$pwd2 pcmk_host_list="vserver5" pcmk_host_map="vserver5:3"
    pcs -f stonith_cfg property set stonith-enabled=false

    pcs cluster cib-push stonith_cfg
```

Configure stonith resource as snmp (we use this one)
```
	pcs stonith delete stonith1
    pcs cluster cib stonith_cfg
    pcs -f stonith_cfg stonith create stonith1 fence_apc_snmp params ipaddr=10.1.1.3 pcmk_host_list="vserver4,vserver5" pcmk_host_map="vserver4:4;verver5:5" pcmk_host_check=static-list power_wait=5
    pcs cluster cib-push stonith_cfg
```

Activate stonith resource:
```
    pcs property set stonith-enabled=true
```

Tests (warning, will reboot nodes!)    
```
    pcs cluster stop vserver5
    stonith_admin --reboot vserver5
```    
```
    pcs cluster start --all    
    pcs cluster stop vserver4
    stonith_admin --reboot vserver4
```

While configuring cluster you may disable fencing:
```
    pcs property set stonith-enabled=false
```

Check stonith resource definition
```
	pcs stonith show --full
```

You'll find logs in:
```
	tail -f /var/log/pacemaker.log 
```


## PACEMAKER DRBD
```
    echo drbd > /etc/modules-load.d/drbd.conf
    pcs resource create drbd-vdisks ocf:linbit:drbd drbd_resource=vdisks op monitor interval=60s
    pcs resource master drbd-vdisks-clone drbd-opt master-max=2 master-node-max=1 clone-max=2 clone-node-max=1 notify=true
```

## dlm
We need cluster locking for gfs2 filesystem
```	 
	pcs cluster cib dlm_cfg
	pcs -f dlm_cfg resource create dlm ocf:pacemaker:controld op monitor interval=60s
	pcs -f dlm_cfg resource clone dlm clone-max=2 clone-node-max=1
	pcs cluster cib-push dlm_cfg
```
## Cluster lvm

### Set up cluster lvms
```
    systemctl disable lvm2-lvmetad.service
	systemctl disable lvm2-lvmetad.socket
	systemctl stop lvm2-lvmetad.service

    lvmconf --enable-cluster
    reboot
```    

You should define the devices where lvm will look for lvm signatures in
file /etc/lvm/lvm.conf:
```
	filter = ["a|sd.*|", "a|md.*|", "a|drbd.*|", "r|.*|"]
```

	
### Set up cluster lock lvms
```
	pcs cluster cib clvmd_cfg
	pcs -f clvmd_cfg resource create clvmd ocf:heartbeat:clvm params daemon_options="timeout=30s" op monitor interval=60s
	pcs -f clvmd_cfg resource clone clvmd clone-max=2 clone-node-max=1
	pcs cluster cib-push clvmd_cfg
```
Verify cluster status:	
```
	[root@vserver5 ~]# pcs status
	Cluster name: vservers
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Thu Oct 29 13:14:58 2015		Last change: Thu Oct 29 13:14:42 2015 by root via cibadmin on vserver5
	Stack: corosync
	Current DC: vserver5 (version 1.1.13-3.fc22-44eb2dd) - partition with quorum
	2 nodes and 7 resources configured

	Online: [ vserver4 vserver5 ]

	Full list of resources:

	 Master/Slave Set: drbd-vdisks-clone [drbd-vdisks]
		 Masters: [ vserver4 vserver5 ]
	 stonith1	(stonith:fence_apc_snmp):	Started vserver4
	 Clone Set: dlm-clone [dlm]
		 Started: [ vserver4 vserver5 ]
	 Clone Set: clvmd-clone [clvmd]
		 Started: [ vserver4 vserver5 ]

	PCSD Status:
	  vserver4 member (vserver4): Online
	  vserver5 member (vserver5): Online

	Daemon Status:
	  corosync: active/enabled
	  pacemaker: active/enabled
	  pcsd: active/enabled
```

### Create volumes

In each server:
```
	pvcreate /dev/drbd0
```
	
If we need to do cluster actions we should use -ci:
```
	-cn ==> local action
	-cy ==> cluster wide action
```
Now we can continue with cluster wide commands:
```
	vgcreate -cy vgcluster /dev/drbd0
```

We can check on the other node if the vg was created:
```
	[root@vserver5 ~]# vgs
	  VG        #PV #LV #SN Attr   VSize   VFree  
	  vg_data     2   1   0 wz--n- 558,79g   1,16g
	  vgcluster   1   0   0 wz--nc 465,62g 465,62g
```	  
	                             
We keep some free space just in case we want to do io tests:
```
	lvcreate -l 97%FREE -n lvcluster1 vgcluster /dev/drbd0
```

And check it fromk the other server:	
```
	[root@vserver4 ~]# lvs
	  LV         VG        Attr       LSize   Pool            Origin          Data%  Meta%  Move Log Cpy%Sync Convert
	  lv_data    vg_data   Cwi-aoC--- 465,63g [lv_cache_data] [lv_data_corig] 0,00   0,82            0,00            
	  lvcluster1 vgcluster -wi-a----- 451,65g    
```

## How to run fencing from drbd itself

In 'handlers' section in /etc/drbd.d/global_common.conf:
```
	fence-peer "/usr/lib/drbd/crm-fence-peer.sh";
	after-resync-target "/usr/lib/drbd/crm-unfence-peer.sh";
```
In 'disk' section:
```
	fencing resource-and-stonith;
```
This is the result:
```
	[root@vserver4 ~]# cat /etc/drbd.d/global_common.conf 
	global {
		usage-count yes;
	}

	common {
		handlers {
			split-brain "/usr/lib/drbd/notify-split-brain.sh root";
			fence-peer "/usr/lib/drbd/crm-fence-peer.sh";
			after-resync-target "/usr/lib/drbd/crm-unfence-peer.sh";
		}

		startup {
		}

		options {
		}

		disk {
			fencing resource-and-stonith;
		}

		net {
			protocol C;

			allow-two-primaries;
			after-sb-0pri discard-zero-changes;
			after-sb-1pri discard-secondary;
			after-sb-2pri disconnect;
		}
	}
```
Check that pacemaker and stonith are working:
```
	[root@vserver4 ~]# pcs property list
	Cluster Properties:
	 cluster-infrastructure: corosync
	 cluster-name: vservers
	 dc-version: 1.1.13-3.fc22-44eb2dd
	 have-watchdog: false
	 stonith-enabled: true
```

Check again that stonith is enabled:
```
    pcs property set stonith-enabled=true
```    

## Constraints

dlm and clvmd must be started in order:

```
pcs cluster cib cons/traints_cfg
pcs constraint order set drbd-vdisks-clone action=promote \
set dlm-clone clvmd-clone action=start \
sequential=true
pcs cluster cib-push constraints_cfg	
```


## DRBD
Sample config of drbd on gfs2 cluster.
```
yum install drbd drbd-utils drbd-udev drbd-pacemaker -y
modprobe drbd
systemctl enable drbd
```
We create drbd resources (/etc/drbd.d/...)
```
drbdadm create-md bases
drbdadm create-md templates
drbdadm create-dm grups
drbdadm up bases
drbdadm up templates
drbdadm up grups
drbdadm primary bases --force
drbdadm primary templates --force
drbdadm primary grups --force
```

## dlm and clvm2 resources

Clusteres lvms
```
pcs cluster cib locks_cfg
pcs -f locks_cfg resource create dlm ocf:pacemaker:controld op monitor interval=60s --group cluster_lock
pcs -f locks_cfg resource create clvmd ocf:heartbeat:clvm params daemon_options="timeout=30s" op monitor interval=60s  --group cluster_lock
pcs -f locks_cfg resource clone cluster_lock clone-max=2 clone-node-max=1 on-fail=restart 
pcs cluster cib-push locks_cfg
```

Without clustered lvms
```
pcs resource create dlm ocf:pacemaker:controld op monitor interval=60s 
pcs resource clone dlm clone-max=2 clone-node-max=1 on-fail=restart
```

## DRBD Resources
```
pcs cluster cib drbd_bases_cfg
pcs -f drbd_bases_cfg resource create drbd_bases ocf:linbit:drbd drbd_resource=bases op monitor interval=60s
pcs -f drbd_bases_cfg resource master drbd_bases-clone drbd_bases master-max=2 master-node-max=1 clone-max=2 clone-node-max=1 notify=true
pcs cluster cib-push drbd_bases_cfg

pcs cluster cib drbd_templates_cfg
pcs -f drbd_templates_cfg resource create drbd_templates ocf:linbit:drbd drbd_resource=templates op monitor interval=60s
pcs -f drbd_templates_cfg resource master drbd_templates-clone drbd_templates master-max=2 master-node-max=1 clone-max=2 clone-node-max=1 notify=true
pcs cluster cib-push drbd_templates_cfg

pcs cluster cib drbd_grups_cfg
pcs -f drbd_grups_cfg resource create drbd_grups ocf:linbit:drbd drbd_resource=grups op monitor interval=60s
pcs -f drbd_grups_cfg resource master drbd_grups-clone drbd_grups master-max=2 master-node-max=1 clone-max=2 clone-node-max=1 notify=true
pcs cluster cib-push drbd_grups_cfg
```

## GFS2 filesystem
```
mkfs.gfs2 -p lock_dlm -t vimet_cluster:bases -j 2 /dev/drbd10
mkfs.gfs2 -p lock_dlm -t vimet_cluster:templates -j 2 /dev/drbd11
mkfs.gfs2 -p lock_dlm -t vimet_cluster:grups -j 2 /dev/drbd30
```
## GFS2 resources
```
pcs resource create gfs2_bases Filesystem device="/dev/drbd10" directory="/vimet/bases" fstype="gfs2" "options=defaults,noatime,nodiratime,noquota" op monitor interval=10s on-fail=restart clone clone-max=2 clone-node-max=1
pcs resource create gfs2_templates Filesystem device="/dev/drbd11" directory="/vimet/templates" fstype="gfs2" "options=defaults,noatime,nodiratime,noquota" op monitor interval=10s on-fail=restart clone clone-max=2 clone-node-max=1
pcs resource create gfs2_grups Filesystem device="/dev/drbd30" directory="/vimet/grups" fstype="gfs2" "options=defaults,noatime,nodiratime,noquota" op monitor interval=10s on-fail=restart clone clone-max=2 clone-node-max=1
```

## NFS 4 server
```
pcs cluster cib nfsserver_cfg
pcs -f nfsserver_cfg resource create nfs-daemon systemd:nfs-server \
nfs_shared_infodir=/nfsshare/nfsinfo nfs_no_notify=true \
--group nfs_server
pcs -f nfsserver_cfg resource create nfs-root exportfs \
clientspec=10.1.0.0/255.255.0.0 \
options=rw,async,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash \
directory=/vimet \
fsid=0 \
--group nfs_server
pcs cluster cib-push nfsserver_cfg
pcs resource clone nfs_server master-max=2 master-node-max=1 clone-max=2 clone-node-max=1 on-fail=restart notify=true resource-stickiness=0
```


## NFS 4 exports
```
pcs cluster cib exports_cfg
pcs -f exports_cfg resource create nfs_bases exportfs \
clientspec=10.1.0.0/255.255.0.0 \
options=rw,async,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash directory=/vimet/bases \
fsid=11 \
clone master-max=2 master-node-max=1 clone-max=2 clone-node-max=1 on-fail=restart notify=true resource-stickiness=0

pcs -f exports_cfg resource create nfs_templates exportfs \
clientspec=10.1.0.0/255.255.0.0 \
options=rw,async,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash directory=/vimet/templates \
fsid=21 \
clone master-max=2 master-node-max=1 clone-max=2 clone-node-max=1 on-fail=restart notify=true resource-stickiness=0

pcs -f exports_cfg resource create nfs_grups exportfs \
clientspec=10.1.0.0/255.255.0.0 \
options=rw,async,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash directory=/vimet/grups \
fsid=31 \
clone master-max=2 master-node-max=1 clone-max=2 clone-node-max=1 on-fail=restart notify=true resource-stickiness=0
pcs cluster cib-push exports_cfg
```

## Floatin IPs

```
pcs resource create ClusterIPbases ocf:heartbeat:IPaddr2 ip=10.1.2.210 cidr_netmask=32 nic=nas:10 clusterip_hash=sourceip-sourceport-destport meta resource-stickiness=0 op monitor interval=5 clone globally-unique=true clone-max=2 clone-node-max=2 on-fail=restart resource-stickiness=0
pcs resource create ClusterIPtemplates ocf:heartbeat:IPaddr2 ip=10.1.2.211 cidr_netmask=32 nic=nas:11 clusterip_hash=sourceip-sourceport-destport meta resource-stickiness=0 op monitor interval=5 clone globally-unique=true clone-max=2 clone-node-max=2 on-fail=restart resource-stickiness=0
pcs resource create ClusterIPgrups ocf:heartbeat:IPaddr2 ip=10.1.2.212 cidr_netmask=32 nic=nas:30 clusterip_hash=sourceip-sourceport-destport meta resource-stickiness=0 op monitor interval=5 clone globally-unique=true clone-max=2 clone-node-max=2 on-fail=restart resource-stickiness=0
pcs resource create ClusterIPcnasbases ocf:heartbeat:IPaddr2 ip=10.1.1.28 cidr_netmask=32 nic=nas:110 clusterip_hash=sourceip-sourceport-destport meta resource-stickiness=0 op monitor interval=5 clone globally-unique=true clone-max=2 clone-node-max=2 on-fail=restart resource-stickiness=0
pcs resource create ClusterIPcnastemplates ocf:heartbeat:IPaddr2 ip=10.1.1.29 cidr_netmask=32 nic=nas:111 clusterip_hash=sourceip-sourceport-destport meta resource-stickiness=0 op monitor interval=5 clone globally-unique=true clone-max=2 clone-node-max=2 on-fail=restart resource-stickiness=0
pcs resource create ClusterIPcnasgrups ocf:heartbeat:IPaddr2 ip=10.1.1.30 cidr_netmask=32 nic=nas:130 clusterip_hash=sourceip-sourceport-destport meta resource-stickiness=0 op monitor interval=5 clone globally-unique=true clone-max=2 clone-node-max=2 on-fail=restart resource-stickiness=0
```

## Constraints
Start and stop order and restrictions
```
pcs constraint order \
	set stonith action=start \
	set cluster_lock-clone action=start \
	set nfs_server-clone action=start \
	require-all=true sequential=true \
	setoptions kind=Mandatory id=serveis

pcs constraint order \
	set drbd_bases-clone action=promote role=Master \
	set gfs2_bases-clone \
	set nfs_bases-clone \
	set ClusterIPcnasbases-clone action=start \
	set ClusterIPbases-clone action=start \
	require-all=true sequential=true \
	setoptions kind=Mandatory id=bases

pcs constraint order \
	set drbd_templates-clone action=promote role=Master \
	set gfs2_templates-clone \
	set nfs_templates-clone \
	set ClusterIPcnastemplates-clone action=start \
	set ClusterIPtemplates-clone action=start \
	require-all=true sequential=true \
	setoptions kind=Mandatory id=templates

pcs constraint order \
	set drbd_grups-clone action=promote role=Master \
	set gfs2_grups-clone \
	set nfs_grups-clone \
	set ClusterIPcnasgrups-clone action=start \
	set ClusterIPgrups-clone action=start \
	require-all=true sequential=true \
	setoptions kind=Mandatory id=grups
```

Location constraints
```
pcs constraint colocation add \
	ClusterIPbases-clone with nfs_bases-clone INFINITY \
	id=colocate_bases

pcs constraint colocation add \
	ClusterIPtemplates-clone with nfs_templates-clone INFINITY \
	id=colocate_templates

pcs constraint colocation add \
	ClusterIPgrups-clone with nfs_grups-clone INFINITY \
	id=colocate_grups

pcs constraint colocation add \
	ClusterIPcnasbases-clone with nfs_bases-clone INFINITY \
	id=colocate_cnasbases

pcs constraint colocation add \
	ClusterIPcnastemplates-clone with nfs_templates-clone INFINITY \
	id=colocate_cnastemplates

pcs constraint colocation add \
	ClusterIPcnasgrups-clone with nfs_grups-clone INFINITY \
	id=colocate_cnasgrups
	

```
