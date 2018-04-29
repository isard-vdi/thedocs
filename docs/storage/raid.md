# RAID
Raid allows for data to be sparsed over multiple disks and give security to your
data as one (or more) disks can fail and data integrity will still remain.

## Initialize disks
```
mdadm --zero-superblock /dev/sd[b-e]
```
or
```
wipefs -a /dev/sd[b-e]
```

## Create raid
You can set level to get more performance or more redundancy

This command will create a raid level 10.
```
mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/sd[b-e] 
```

Check raid types and benefits here: http://www.raid-calculator.com/

Save your newly created raid configuration for next reboots:
```
mdadm --detail --scan > /etc/mdadm/mdadm.conf
```

## Utils

- cat /proc/mdstat: shows raid status and progress

## Considerations

mdadm software raid will trigger a complete check every sunday at 1am.
Be aware of that as it will lower diks io speed a lot.
