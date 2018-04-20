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

Tuneando fedora

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

Resources
Y creamos un fichero /etc/drbd.d/vdisks.res para el recurso. 

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

En los dos vservers creamos metadatos en el volumen lógico y iniciamos
el drbd:

    drbdadm create-md vdisks

Si todo va bien nos devuelve un mensaje como:

    [...]
    Writing meta data...
    New drbd meta data block successfully created.
    success

Ahora para verificar que los ficheros de configuración son correctos
y ver que operaciones se van a realizar utilizamos la orden adjust 
en dry-run:

    drbdadm -d adjust all
    
Si parece que las órdenes son las correctas y no da errores ejecutamos:

    drbdadm adjust all
    
Esta instrucción hace attach y connect (pone en up) el recurso.

Ahora verificamos que el drbd está corriendo y el estado es inconsistente
en ambos equipos:

    [root@vserver4 ~]# cat /proc/drbd 
    version: 8.4.5 (api:1/proto:86-101)
    srcversion: 5A4F43804B37BB28FCB1F47 
     0: cs:Connected ro:Secondary/Secondary ds:Inconsistent/Inconsistent C r-----
        ns:0 nr:0 dw:0 dr:0 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:488236452

Y ahora sólo en un equipo, por ejemplo en vserver4 forzamos a que 
sea primario, y se inicia la sincronización:

    drbdadm primary --force vdisks

Desde el otro equipo vserver5 comprobamos que el dual primary funciona:

    drbdadm primary vdisks

Y aparece UpToDate en un extremo, inconsistente en otro y sincronizando

    [root@vserver4 ~]# cat /proc/drbd 
    version: 8.4.5 (api:1/proto:86-101)
    srcversion: 5A4F43804B37BB28FCB1F47 
     0: cs:SyncSource ro:Primary/Primary ds:UpToDate/Inconsistent C r-----
        ns:17322104 nr:0 dw:0 dr:17323016 al:0 bm:0 lo:2 pe:2 ua:2 ap:0 ep:1 wo:f oos:470916516
        [>....................] sync'ed:  3.6% (459876/476792)M
        finish: 3:03:06 speed: 42,844 (36,616) K/sec


El arranque automático del drbd habrá que configurarlo en cluster 
posteriormente


# Cluster

## Instalar paquetes para que pacemaker controle drbd y gfs2

Hay que instalar las utilidades para hacer fence sobre regleta ip marca
APC:

    yum -y install fence-agents-apc fence-agents-apc-snmp
    fence_apc --help
    fence_apc_snmp --help


    dnf -y install corosync pcs pacemaker pacemaker-doc

En fedora, no como en centos, hay que instalar un recurso especial para
que sepa manejar el drbd
    
    dnf -y install drbd-pacemaker 
    
Que solamente instala:

    /usr/lib/ocf/resource.d/linbit/drbd
    
Ahora el control del sistema de ficheros gfs2 requiere de dlm y clvm:

    dnf -y install gfs2-utils lvm2-cluster dlm
    


## Crear cluster

En principio el único demonio que se necesita es el pcsd, y ese ya arranca
pacemaker y corosync

    systemctl start pcsd
    systemctl enable pcsd
    
Hay que permitir que un nodo y otro puedan entrarse remotamente, por eso
cambiamos el password del usuario que usaremos para intercomunciarnos.
    
    passwd hacluster
    


Antes de empezar hay que asegurarse que el hostname que definiremos en 
el cluster es idéntico al que tenemos como hostname en linux. Por defecto 
el hostname se define con el dominio dns completo, y en nuestro caso nos
interesa dejarlo pelado. 

También hemos de protegernos de posibles caídas del servidor dns, con lo
que damos de alta vserver4 y vserver5 en el /etc/hosts

Por eso antes de empezar, en cada servidor:

En vserver4:
    echo "vserver4" > /etc/hostname
    echo "10.1.1.24   vserver4" >> /etc/hosts
    echo "10.1.1.25   vserver5" >> /etc/hosts
    exit

En vserver5: 

    echo "10.1.1.24   vserver4" >> /etc/hosts
    echo "10.1.1.25   vserver5" >> /etc/hosts
    echo "vserver5" > /etc/hostname
    exit
    
Salimos y volvemos a entrar para que la variable de sesión del hostname
se actualice.

En uno de los dos nodos (ojo, no hacerlo en los dos a la vez):

    server1=vserver4
    server2=vserver5
    cl_name=vservers
    
    pcs cluster auth $server1 $server2
    pcs cluster setup --name $cl_name $server1 $server2
    pcs cluster start --all
    pcs status

Con esto ya tenemos el cluster en marcha y la salida de pcs status:

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

Da un par de warnings, el de los stonith lo arreglaremos al definir el 
fencing, el segundo parece que no tiene mucho sentido, después de googlear
un poco pasamos de hacerle caso ya que parece que están bien definidos 
en corosync.conf y en la base de datos de pacemaker.

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


## Fencing 

Para asegurarnos que están bien instaladas las herramientas de fencing
para la regleta ip de marca apc verificamos:

    [root@vserver4 ~]# pcs stonith list 
    fence_apc - Fence agent for APC over telnet/ssh
    fence_apc_snmp - Fence agent for APC, Tripplite PDU over SNMP

Para ver el nombre de las opciones:

    pcs stonith describe fence_apc_snmp

Y ahora configuramos guardando primero en un fichero de configuración, 
modificándolo y volviéndolo a subir. Este procedimiento permite aplicar
cambios de golpe, no paso a paso, evitando estados intermedios que en un
cluster pueden suponer la activación de un proceso de fencing o 
movimientos de recursos de un host a otro por ejemplo.


Como el fence_apc daba problemas cuando más de un nodo hacía consultas
después de mirar por internet y de hacer pruebas hemos descartado usar
el fence_apc y usar fence_apc_snmp que no bloquea el stonith mientras
hace operaciones como status, list o entramos por la web. En resumen,
que con snmp va mejor. 

Entonces hemos creado un nuevo community en snmp v1 en el stonith y 
limitado la ip origen a la familia 10.1.1.255 (que es como si dijeras
10.1.1.0/24) y hemos probado con la orden fence_apc_snmp desde el bash
que funciona. Después hemos creado el stonith en pcs. Hemos añadido un
delay de 5 segundos entre que lo apaga y lo enciende, ya que algún pc
se quedaba frito o reseteaba sin leer bien los dispositivos si el corte
de corriente era tan breve.

Probar que hace bien la monitorización, si no da error es que va bien:

	fence_apc_snmp --ip=stonith1 --action=monitor
	
Probar con la community de acceso de escritura:
	
	fence_apc_snmp --ip=stonith1 --action=monitor --community=escola2015

Probar que hace un reboot con el delay en un enchufe vacío!!

	fence_apc_snmp --ip=stonith1 --action=reboot --plug=6  --community=escola2015 --power-wait=5
	


ANTIGUA CONFIGURACIÓN QUE NO SE USA SIN SNMP

Leemos los passwords de unos ficheros:

    #pwd1=$(cat /root/pwd1)
    #pwd2=$(cat /root/pwd2)

	pcs stonith delete stonith1

    pcs cluster cib stonith_cfg

    
    pcs -f stonith_cfg stonith create stonith1 fence_apc ipaddr=10.1.1.3 login=vservers passwd=$pwd1 pcmk_host_list="vserver4 vserver5" pcmk_host_map="vserver4:4;vserver5:5"
    #pcs -f stonith_cfg stonith create  stonith2 fence_apc ipaddr=10.1.1.3 login=vserver5 passwd=$pwd2 pcmk_host_list="vserver5" pcmk_host_map="vserver5:3"
    pcs -f stonith_cfg property set stonith-enabled=false

    pcs cluster cib-push stonith_cfg

CONFIGURACIÓN QUE SÍ QUE USAMOS CON SNMP:

	pcs stonith delete stonith1

    pcs cluster cib stonith_cfg
    
    pcs -f stonith_cfg stonith create stonith1 fence_apc_snmp params ipaddr=10.1.1.3 pcmk_host_list="vserver4,vserver5" pcmk_host_map="vserver4:4;verver5:5" pcmk_host_check=static-list power_wait=5

    pcs cluster cib-push stonith_cfg

Desaparece el warning del stonith y aparece el recurso.

Comprobamos que el fencing reinicia los equipos

    pcs property set stonith-enabled=true
    
Desde vserver4:

    pcs cluster stop vserver5
    stonith_admin --reboot vserver5
    
Una vez vuelve a arrancar desde vserver5:

    pcs cluster start --all    
    pcs cluster stop vserver4
    stonith_admin --reboot vserver4

Si queremos desactivar temporalmente el fencing:

    pcs property set stonith-enabled=false

Si algo va mal repasar la configuración de los stonith con la orden:

	pcs stonith show --full

Logs en:

	tail -f /var/log/pacemaker.log 

## Controlar drbd desde pacemaker

El drbd para que se arranque automáticamente cuando inicie la máquina
y esté monitorizado por el cluster ha de pasar bajo el control de pacemaker
con lo que no utilizaremos el servicio de drbd que viene con los 
paquetes de drbd. 

ME HE QUEDADO AQUI

    echo drbd > /etc/modules-load.d/drbd.conf
    pcs resource create drbd-vdisks ocf:linbit:drbd drbd_resource=vdisks op monitor interval=60s
    pcs resource master drbd-vdisks-clone drbd-opt master-max=2 master-node-max=1 clone-max=2 clone-node-max=1 notify=true

## dlm

Para que vaya el clvm y para que vaya el sistema de ficheros gfs2
necesitamos que funcione el dlm que gestiona los bloqueos de recursos
en un cluster. Para eso hay un demonio que ha de arrancar pacemaker,
y ese demonio ha de correr a la vez en los dos nodos del cluster. Si
no no pueden arrancar lvm ni gfs2.
	
	 
	pcs cluster cib dlm_cfg
	pcs -f dlm_cfg resource create dlm ocf:pacemaker:controld op monitor interval=60s
	pcs -f dlm_cfg resource clone dlm clone-max=2 clone-node-max=1
	pcs cluster cib-push dlm_cfg

## Cluster lvm

### Tunear systemd y lvm en fedora

Com que crearem clvm desactivem el servei de cache de metadades, 
activamos que lvm trabaje con clvm y reiniciamos para que lvm esté 
disponible con clvm desde el arranque:

    systemctl disable lvm2-lvmetad.service
	systemctl disable lvm2-lvmetad.socket
	systemctl stop lvm2-lvmetad.service

    lvmconf --enable-cluster
    reboot
    

Hay que decirle que sólo mire firmas de lvm en los discos físicos, 
raids y drbds, para eso editamos el fichero /etc/lvm/lvm.conf
y añadimos la línea donde toque:

	filter = ["a|sd.*|", "a|md.*|", "a|drbd.*|", "r|.*|"]
	
### Crear el recurso clvmd en el cluster

No existe un demonio de clvm, pero en pacemaker podemos hacer que haya 
un hearthbeat que mire si el clvm está activo. 

	pcs cluster cib clvmd_cfg
	pcs -f clvmd_cfg resource create clvmd ocf:heartbeat:clvm params daemon_options="timeout=30s" op monitor interval=60s
	pcs -f clvmd_cfg resource clone clvmd clone-max=2 clone-node-max=1
	pcs cluster cib-push clvmd_cfg
	
Verificamos que está started antes de crear el volumen de grupo en clsuter:

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

### Crear el volumen lógico en cluster

En cada nodo:

	pvcreate /dev/drbd0
	
Cuando creamos los grupos de volumen con los modificadores:

	-cn ==> no los crea en cluster
	-ci ==> sí los crea en cluster


Ahora sólo en uno de los nodos (por ejemplo vserver4) creamos los
volúmenes lógicos y los grupos de volumen en cluster.

	vgcreate -cy vgcluster /dev/drbd0

En el otro nodo (por ejemplo vserver5) comprobamos que se han creado
los volúmenes en cluster y que los lee:

	[root@vserver5 ~]# vgs
	  VG        #PV #LV #SN Attr   VSize   VFree  
	  vg_data     2   1   0 wz--n- 558,79g   1,16g
	  vgcluster   1   0   0 wz--nc 465,62g 465,62g
	  
	                             |---> esa c nos dice que es en cluster
	                             
Dejo un poco de espacio libre en el volumen lógico por si queremos hacer
alguna prueba:

	lvcreate -l 97%FREE -n lvcluster1 vgcluster /dev/drbd0
	
Y desde el otro nodo se ve:

	[root@vserver4 ~]# lvs
	  LV         VG        Attr       LSize   Pool            Origin          Data%  Meta%  Move Log Cpy%Sync Convert
	  lv_data    vg_data   Cwi-aoC--- 465,63g [lv_cache_data] [lv_data_corig] 0,00   0,82            0,00            
	  lvcluster1 vgcluster -wi-a----- 451,65g    


# Fencing desde drbd

Hay que tocar el /etc/drbd.d/global_common.conf y añadir:

En el apartado handlers:

	fence-peer "/usr/lib/drbd/crm-fence-peer.sh";
	after-resync-target "/usr/lib/drbd/crm-unfence-peer.sh";

En el apartado disk:

	fencing resource-and-stonith;

Y el fichero queda:

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

Verificamos que en pacemaker está el stonith activado:

	[root@vserver4 ~]# pcs property list
	Cluster Properties:
	 cluster-infrastructure: corosync
	 cluster-name: vservers
	 dc-version: 1.1.13-3.fc22-44eb2dd
	 have-watchdog: false
	 stonith-enabled: true

Si no:

    pcs property set stonith-enabled=true
    

# Antes del gfs2

## Constraints

dlm y clvmd han de ir juntos y arrancarse en orden



Podemos montar un order set para que ejecute todo en orden hasta el
montaje de gfs2:

pcs cluster cib cons/traints_cfg
pcs constraint order set drbd-vdisks-clone action=promote \
set dlm-clone clvmd-clone action=start \
sequential=true
pcs cluster cib-push constraints_cfg	
# ideas en el tintero

Igual molaría un script que pusiera para los ssds:

    echo deadline > /sys/block/sdX/queue/scheduler 
    
drbd también recomienda sobre ssds y caches:

    echo 0    > /sys/block/<device>/queue/iosched/front_merges
    echo 150  > /sys/block/<device>/queue/iosched/read_expire
    echo 1500 > /sys/block/<device>/queue/iosched/write_expire


Revisar mtu en drbd
