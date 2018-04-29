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

## Utils

- drbdmanage nodes: Show nodes in cluster with available space
- drbdmanage resources: show resources
- drbdmanage volumes: Show volumes
- drbdmanage uninit <node-name>: Remove node from cluster. All resources on that node will be lost.
- drbdmanage peer-disk-options --common <options>: Modify disk options.
- drbdmanage net-options --common <options>: Modify net options
- drbdsetup status: Show resource status and synchronization progress. Use --verbose for details.
