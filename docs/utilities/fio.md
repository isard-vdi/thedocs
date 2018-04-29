# fio
Utility to test io performance on hard disks.
http://fio.readthedocs.io/en/latest/fio_doc.html

## Sample configuration
```
[global]
directory=/isard/2/groups/test
direct=1
ioengine=libaio
iodepth=32
numjobs=4
size=10G
runtime=20
group_reporting=1
exec_prerun=echo 3 > /proc/sys/vm/drop_caches

[maxreadbw]
# specs: 2.200MB/s real: 2.180MB/s
bs=128k
rw=read
wait_for_previous=1

[maxreadio]
# 430.000 iops
bs=4k
rw=randread
wait_for_previous=1

[maxwritebw]
# specs: 900MB/s real: 971MB/s
bs=128k
rw=write
wait_for_previous=1

[randwriteio]
# 230.000 iops
bs=4k
rw=randwrite
wait_for_previous=1

[qcow2]
bs=64k
rw=randrw
wait_for_previous=1
```
