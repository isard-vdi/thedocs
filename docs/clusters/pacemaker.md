# Pacemaker

## Configuring
Pacemaker is a cluster an resource manager

To install it on CentOS
```
yum install corosync pacemaker pcs python-pycurl fence-agents-apc fence-agents-apc-snmp
´´´
Enable and start services
´´´
systemctl enable pcsd
systemctl enable corosync
systemctl enable pacemaker
systemctl start pcsd
´´´
We need a new password to join the cluster. The default user is hacluster
´´´
passwd hacluster
´´´
Authorize servers, set up cluster name and start all
´´´
pcs cluster auth cluster1 cluster2
pcs cluster setup --name backup_cluster cluster1 cluster2
pcs cluster start --all
´´´

We can disable some settings till we finish configuration
´´´
pcs property set stonith-enabled=false
pcs property set maintenance-mode=true
´´´

The first resource that should exist is a fencing device
´´´
pcs cluster cib stonith_cfg
pcs -f stonith_cfg stonith create stonith1 fence_apc_snmp ipaddr=10.1.79.11 pcmk_host_list="cluster1" pcmk_host_map="cluster1:6" pcmk_host_check=static-list power_wait=3 inet4_only=1 retry_on=10
pcs -f stonith_cfg stonith create stonith2 fence_apc_snmp ipaddr=10.1.79.12 pcmk_host_list="cluster2" pcmk_host_map="cluster2:6" pcmk_host_check=static-list power_wait=5 inet4_only=1 retry_on=10
pcs cluster cib-push stonith_cfg
´´´

## Filesystem resource
´´´
pcs resource create backup_ext4 ocf:heartbeat:Filesystem device="/dev/drbd104" directory="/mnt/backup" fstype="ext4" "options=defaults,noatime" op monitor interval=10s
pcs resource create isard_ext4 ocf:heartbeat:Filesystem device="/dev/drbd102" directory="/mnt/isard" fstype="ext4" "options=defaults,noatime" op monitor interval=10s
´´´
## Floating IPs
´´´
pcs resource create backup_ip ocf:heartbeat:IPaddr2 ip=10.1.90.10 cidr_netmask=32 nic=escola:0 op monitor interval=30
pcs resource create isard_ip ocf:heartbeat:IPaddr2 ip=10.1.2.200 cidr_netmask=32 nic=isard:0 op monitor interval=30
´´´

## NFS v4 server
´´´
pcs cluster cib nfs_cfg
pcs -f nfs_cfg resource create nfs_daemon systemd:nfs-server op monitor interval=30s --group=nfs_server
pcs -f nfs_cfg resource create nfs_root ocf:heartbeat:exportfs clientspec=10.1.0.0/255.255.0.0 options=rw,crossmnt,async,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash directory=/mnt fsid=0 --group=nfs_server
pcs cluster cib-push nfs_cfg
pcs resource clone nfs_server master-max=3 master-node-max=1 clone-max=3 clone-node-max=1 on-fail=restart notify=true resource-stickiness=0
´´´

## NFS v4 exports
´´´
pcs cluster cib exports_cfg
pcs -f exports_cfg resource create backup_nfs ocf:heartbeat:exportfs clientspec=10.1.0.0/255.255.0.0 wait_for_leasetime_on_stop=true options=rw,mountpoint,async,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash directory=/mnt/backup fsid=5 op monitor interval=30s
pcs -f exports_cfg resource create isard_nfs ocf:heartbeat:exportfs clientspec=10.1.0.0/255.255.0.0 wait_for_leasetime_on_stop=true options=rw,mountpoint,async,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash directory=/mnt/isard fsid=6 op monitor interval=30s
pcs cluster cib-push exports_cfg
´´´

## Resource groups
´´´
pcs resource group add backup backup_ext4 backup_nfs backup_ip
pcs resource group add isard isard_ext4 isard_nfs isard_ip
´´´

## Activate cluster
´´´
pcs property set maintenance-mode=false
pcs property set stonith-enabled=true
´´´
