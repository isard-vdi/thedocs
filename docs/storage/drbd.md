# DRBD 9
## Install required packages
```
apt install drbd-dkms drbd-utils python-drbdmanage
```

Check module install
```
modprobe drbd
modinfo drbd
```

Enable and start drbd cluster manager service
```
systemctl enable drbdmanaged
systemctl start drbdmanaged
```

## Configure drbd9 cluster
Create drbdpool volume group on desired PVs
```
vgcreate drbdpool /dev/nvme0n1 /dev/mapper/disks
```

Initialize the drbdmanage in the master node with own parameters
```
drbdmanage init <node-name> <ip_drbd>
```

Add volume and resources
```
drbdmanage add-volume <volume-name> <capacity>
```
Note: It will create and associated resource with the same volume-name.

Deploy resources to nodes
```
drbdmanage deploy <resource-name> <nº nodes>
```

Just if you want to allocate volumes on desired PV
```
pvmove <resource-name> <pv>
```

Create filesystem and mount
```
mkfs.ext4 /dev/<resource-name>
mount /dev/<resource-name> /mnt
```
In drbd9 mount action will automatically trigger a Secondary -> Primary change to allow rw mount.

## Types of nodes
In all types what it is shared is a block device, also in diskless nodes.

### Control node
- Control volume (.drbdctrl): local (could be pri/sec)
- Resources: local (could be pri/sec)
```
drbdmanage add-node drbd1 192.168.0.11
```
The cluster startup is done with drbdmanage init and it will become also a control node.

### Pure controller node
- Control volume (.drbdctrl): local (could be pri/sec)
- Resources: local (could be pri/sec)
```
drbdmanage add-node --no-storage drbd2-controller 192.168.0.12
```
It is like a control node but as it won't have storage, will only act a a control node or a satellite.

### Satellite node
- Control volume (.drbdctrl): remote, via TCP. (could be pri/sec)
- Resources: local (could be pri/sec)
```
drbdmanage add-node --satellite drbd3-satellite 192.168.0.13
```

### Pure client node
- Control volume (.drbdctrl): remote, via TCP. (could be pri/sec)
- Resources: remote, via TCP. (could be pri/sec)
```
drbdmanage add-node --satellite --no-storage drbd4-client 192.168.0.14
```

## Utils

- drbdmanage nodes: Show nodes in cluster with available space
- drbdmanage resources: show resources
- drbdmanage volumes: Show volumes
- drbdmanage uninit <node-name>: Remove node from cluster. All resources on that node will be lost.
- drbdmanage peer-disk-options --common <options>: Modify disk options.
- drbdmanage net-options --common <options>: Modify net options
- drbdsetup status: Show resource status and synchronization progress. Use --verbose for details.
