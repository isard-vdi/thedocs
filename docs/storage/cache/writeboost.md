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
 
