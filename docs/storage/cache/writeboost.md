# dm-writeboost

+ https://github.com/akiradeveloper/dm-writeboost

This sample configuration does not use DRBD, it's just a simple dm-writeboost setup.

## Installation on Centos 7

### DKMS
```
yum install dkms -y
```

### Writeboost DKMS module
Clone repository:

```
git clone https://github.com/akiradeveloper/dm-writeboost
cd dm-writeboost && make && make install
```

Check that Writeboost DKMS is installed:
```
dkms status
lsmod | grep dm_writeboost
```

### Writeboost tools

```
git clone https://github.com/akiradeveloper/dm-writeboost-tools
yum install cargo
cargo install
cp /root/.cargo/bin/* /usr/sbin/
```

Reboot and check module install


### Create new Writeboost Cache

```
wbcreate --reformat --read_cache_threshold=127  --writeback_threshold=80 storage /dev/md0 /dev/nvme0n1
```

- storage: new cache device name
- md0: slow disks
- nvme0n1: fast disks (cache device)

### Crete writeboosttab and service

vi /etc/writeboosttab

```
## dm-writeboost "tab" (mappings) file, see writeboosttab(5).
##{DM target name}    {cached block device e.g. HDD}    {caching block device e.g. SSD}    [options]
##
## wb_hdd     /dev/disk/by-uuid/2e8260bc-024c-4252-a695-a73898c974c7     /dev/disk/by-partuuid/43372b68-3407-45fa-9b2f-61afe9c26a68    writeback_threshold=70,sync_data_interval=3600
##
storage     /dev/md0     /dev/nvme0n1    writeback_threshold=80,read_cache_threshold=127

```

This is a sample writeboost systemd unit file:

```
https://gitlab.com/onlyjob/writeboost/blob/master/writeboost.service
```
systemctl daemon-reload
systemctl enable writeboost
 
And this is the modified writeboost service file that we use when drbd9 is used over writeboost:

```
[Unit]
Description=(dm-)writeboost mapper
Documentation=man:writeboost
#DefaultDependencies=false
#Conflicts=shutdown.target

## "Before=local-fs-pre" is significant as it influences correct order
## of stopping (after unmount).
#Before=shutdown.target drbd.service cryptsetup.target local-fs-pre.target
#Before=shutdown.target cryptsetup.target local-fs-pre.target

Before=shutdown.target drbdmanaged.service

[Service]
Type=oneshot

## Must remain after exit to prevent stopping right after start
## and to stop on shutdown.
RemainAfterExit=yes

## Scannong caching devices may take long time after unclean shutdown.
TimeoutStartSec=3600

ExecStart=/usr/bin/bash -c '/usr/sbin/writeboost; lvscan; lvchange -ay /dev/drbdpool/data_00'
ExecStop=/usr/sbin/writeboost -u

## Long "TimeoutStop" is essential as deadlock may happen if writeboost
## is killed during flushing of caches on shutdown, etc.
TimeoutStopSec=3600

StandardOutput=syslog+console

[Install]
#WantedBy=cryptsetup.target
#WantedBy=local-fs.target
WantedBy=drbdmanaged.service
#Alias=dm-writeboost.service


```
