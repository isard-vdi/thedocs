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

And here you have a sample ouput and you can see that the results approximate to what the disk company says.
```
maxreadbw: (g=0): rw=read, bs=128K-128K/128K-128K/128K-128K, ioengine=libaio, iodepth=32
...
maxreadio: (g=1): rw=randread, bs=4K-4K/4K-4K/4K-4K, ioengine=libaio, iodepth=32
...
maxwritebw: (g=2): rw=write, bs=128K-128K/128K-128K/128K-128K, ioengine=libaio, iodepth=32
...
randwriteio: (g=3): rw=randwrite, bs=4K-4K/4K-4K/4K-4K, ioengine=libaio, iodepth=32
...
qcow2: (g=4): rw=randrw, bs=64K-64K/64K-64K/64K-64K, ioengine=libaio, iodepth=32
...
fio-2.2.8
Starting 20 processes
maxreadbw: Laying out IO file(s) (1 file(s) / 10240MB)
maxreadbw: Laying out IO file(s) (1 file(s) / 10240MB)
maxreadbw: Laying out IO file(s) (1 file(s) / 10240MB)
maxreadbw: Laying out IO file(s) (1 file(s) / 10240MB)
maxreadio: Laying out IO file(s) (1 file(s) / 10240MB)
maxreadio: Laying out IO file(s) (1 file(s) / 10240MB)
maxreadio: Laying out IO file(s) (1 file(s) / 10240MB)
maxreadio: Laying out IO file(s) (1 file(s) / 10240MB)
maxwritebw: Laying out IO file(s) (1 file(s) / 10240MB)
maxwritebw: Laying out IO file(s) (1 file(s) / 10240MB)
maxwritebw: Laying out IO file(s) (1 file(s) / 10240MB)
maxwritebw: Laying out IO file(s) (1 file(s) / 10240MB)
randwriteio: Laying out IO file(s) (1 file(s) / 10240MB)
randwriteio: Laying out IO file(s) (1 file(s) / 10240MB)
randwriteio: Laying out IO file(s) (1 file(s) / 10240MB)
randwriteio: Laying out IO file(s) (1 file(s) / 10240MB)
qcow2: Laying out IO file(s) (1 file(s) / 10240MB)
qcow2: Laying out IO file(s) (1 file(s) / 10240MB)
qcow2: Laying out IO file(s) (1 file(s) / 10240MB)
qcow2: Laying out IO file(s) (1 file(s) / 10240MB)
maxreadbw : Saving output of prerun in maxreadbw.prerun.txt
maxreadbw : Saving output of prerun in maxreadbw.prerun.txt
maxreadbw : Saving output of prerun in maxreadbw.prerun.txt
maxreadbw : Saving output of prerun in maxreadbw.prerun.txt
maxreadio : Saving output of prerun in maxreadio.prerun.txt] [17.4K/0/0 iops] [eta 00m:20s]
maxreadio : Saving output of prerun in maxreadio.prerun.txt
maxreadio : Saving output of prerun in maxreadio.prerun.txt
maxreadio : Saving output of prerun in maxreadio.prerun.txt
maxwritebw : Saving output of prerun in maxwritebw.prerun.txt /s] [424K/0/0 iops] [eta 00m:20s]
maxwritebw : Saving output of prerun in maxwritebw.prerun.txt
maxwritebw : Saving output of prerun in maxwritebw.prerun.txt
maxwritebw : Saving output of prerun in maxwritebw.prerun.txt
randwriteio : Saving output of prerun in randwriteio.prerun.txts] [0/7717/0 iops] [eta 00m:22s]        
randwriteio : Saving output of prerun in randwriteio.prerun.txt
randwriteio : Saving output of prerun in randwriteio.prerun.txt
randwriteio : Saving output of prerun in randwriteio.prerun.txt
qcow2 : Saving output of prerun in qcow2.prerun.txt338.4MB/0KB /s] [0/86.7K/0 iops] [eta 01m:12s]
qcow2 : Saving output of prerun in qcow2.prerun.txt
qcow2 : Saving output of prerun in qcow2.prerun.txt
qcow2 : Saving output of prerun in qcow2.prerun.txt
Jobs: 4 (f=4): [_(16),m(4)] [90.2% done] [681.6MB/667.2MB/0KB /s] [10.1K/10.7K/0 iops] [eta 00m:11s]
maxreadbw: (groupid=0, jobs=4): err= 0: pid=11498: Sat Nov 14 17:41:50 2015
  read : io=40960MB, bw=2177.2MB/s, iops=17416, runt= 18814msec
    slat (usec): min=5, max=1158, avg=16.98, stdev= 5.34
    clat (usec): min=313, max=18243, avg=7329.29, stdev=3387.42
     lat (usec): min=328, max=18259, avg=7346.41, stdev=3387.40
    clat percentiles (usec):
     |  1.00th=[  700],  5.00th=[ 1320], 10.00th=[ 2352], 20.00th=[ 4576],
     | 30.00th=[ 5664], 40.00th=[ 6624], 50.00th=[ 7328], 60.00th=[ 7968],
     | 70.00th=[ 8896], 80.00th=[10176], 90.00th=[12352], 95.00th=[13376],
     | 99.00th=[14144], 99.50th=[14400], 99.90th=[14912], 99.95th=[15040],
     | 99.99th=[15552]
    bw (KB  /s): min=548864, max=570112, per=25.00%, avg=557278.53, stdev=3994.50
    lat (usec) : 500=0.18%, 750=1.12%, 1000=1.60%
    lat (msec) : 2=5.68%, 4=8.45%, 10=62.19%, 20=20.78%
  cpu          : usr=1.03%, sys=10.54%, ctx=294128, majf=0, minf=4171
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=100.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued    : total=r=327680/w=0/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=32
maxreadio: (groupid=1, jobs=4): err= 0: pid=11523: Sat Nov 14 17:41:50 2015
  read : io=29406MB, bw=1470.3MB/s, iops=376374, runt= 20001msec
    slat (usec): min=1, max=1136, avg= 2.61, stdev= 2.80
    clat (usec): min=39, max=3137, avg=336.87, stdev=208.64
     lat (usec): min=42, max=3139, avg=339.54, stdev=208.70
    clat percentiles (usec):
     |  1.00th=[   78],  5.00th=[   92], 10.00th=[  102], 20.00th=[  126],
     | 30.00th=[  171], 40.00th=[  274], 50.00th=[  342], 60.00th=[  378],
     | 70.00th=[  418], 80.00th=[  474], 90.00th=[  588], 95.00th=[  708],
     | 99.00th=[ 1012], 99.50th=[ 1144], 99.90th=[ 1432], 99.95th=[ 1560],
     | 99.99th=[ 2064]
    bw (KB  /s): min=324544, max=497128, per=25.00%, avg=376387.60, stdev=38308.47
    lat (usec) : 50=0.01%, 100=8.77%, 250=29.14%, 500=45.15%, 750=12.98%
    lat (usec) : 1000=2.90%
    lat (msec) : 2=1.04%, 4=0.01%
  cpu          : usr=10.42%, sys=36.06%, ctx=2972500, majf=0, minf=527
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=100.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued    : total=r=7527872/w=0/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=32
maxwritebw: (groupid=2, jobs=4): err= 0: pid=11531: Sat Nov 14 17:41:50 2015
  write: io=19572MB, bw=976.99MB/s, iops=7815, runt= 20033msec
    slat (usec): min=9, max=1220, avg=38.16, stdev=11.93
    clat (usec): min=34, max=64684, avg=16334.02, stdev=12762.63
     lat (usec): min=68, max=64734, avg=16372.51, stdev=12762.33
    clat percentiles (usec):
     |  1.00th=[   66],  5.00th=[   96], 10.00th=[  310], 20.00th=[ 2160],
     | 30.00th=[ 4832], 40.00th=[ 7392], 50.00th=[15680], 60.00th=[24960],
     | 70.00th=[27776], 80.00th=[30336], 90.00th=[32384], 95.00th=[33024],
     | 99.00th=[34560], 99.50th=[35072], 99.90th=[37632], 99.95th=[45824],
     | 99.99th=[59136]
    bw (KB  /s): min=240629, max=259321, per=25.00%, avg=250112.90, stdev=4088.96
    lat (usec) : 50=0.02%, 100=5.62%, 250=4.00%, 500=2.33%, 750=1.79%
    lat (usec) : 1000=1.47%
    lat (msec) : 2=4.08%, 4=7.25%, 10=17.73%, 20=8.86%, 50=46.82%
    lat (msec) : 100=0.03%
  cpu          : usr=3.13%, sys=8.95%, ctx=152485, majf=0, minf=4170
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=99.9%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=156572/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=32
randwriteio: (groupid=3, jobs=4): err= 0: pid=11617: Sat Nov 14 17:41:50 2015
  write: io=9133.5MB, bw=467610KB/s, iops=116902, runt= 20001msec
    slat (usec): min=3, max=55410, avg=12.25, stdev=415.43
    clat (usec): min=7, max=69546, avg=1081.96, stdev=3366.40
     lat (usec): min=13, max=69556, avg=1094.26, stdev=3392.60
    clat percentiles (usec):
     |  1.00th=[   22],  5.00th=[  564], 10.00th=[  604], 20.00th=[  684],
     | 30.00th=[  740], 40.00th=[  788], 50.00th=[  844], 60.00th=[  892],
     | 70.00th=[  932], 80.00th=[  964], 90.00th=[ 1004], 95.00th=[ 1032],
     | 99.00th=[ 1256], 99.50th=[40192], 99.90th=[46848], 99.95th=[48384],
     | 99.99th=[58112]
    bw (KB  /s): min=76648, max=228728, per=25.14%, avg=117577.33, stdev=42200.75
    lat (usec) : 10=0.01%, 20=0.84%, 50=0.75%, 100=0.07%, 250=0.13%
    lat (usec) : 500=0.60%, 750=29.72%, 1000=57.44%
    lat (msec) : 2=9.59%, 4=0.12%, 10=0.13%, 20=0.01%, 50=0.55%
    lat (msec) : 100=0.04%
  cpu          : usr=3.77%, sys=29.50%, ctx=1820945, majf=0, minf=531
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=100.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=2338168/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=32
qcow2: (groupid=4, jobs=4): err= 0: pid=11724: Sat Nov 14 17:41:50 2015
  read : io=13225MB, bw=676766KB/s, iops=10574, runt= 20010msec
    slat (usec): min=3, max=10750, avg=12.63, stdev=23.91
    clat (usec): min=200, max=52176, avg=6612.80, stdev=4262.64
     lat (usec): min=207, max=52180, avg=6625.59, stdev=4262.60
    clat percentiles (usec):
     |  1.00th=[  338],  5.00th=[  692], 10.00th=[ 1128], 20.00th=[ 2024],
     | 30.00th=[ 2896], 40.00th=[ 4448], 50.00th=[ 6688], 60.00th=[ 8896],
     | 70.00th=[10176], 80.00th=[10944], 90.00th=[11840], 95.00th=[12480],
     | 99.00th=[13632], 99.50th=[14528], 99.90th=[17280], 99.95th=[26496],
     | 99.99th=[47360]
    bw (KB  /s): min=125952, max=185216, per=25.00%, avg=169196.04, stdev=7218.28
  write: io=13185MB, bw=674735KB/s, iops=10542, runt= 20010msec
    slat (usec): min=4, max=11507, avg=15.31, stdev=42.32
    clat (usec): min=18, max=49475, avg=5476.05, stdev=4190.25
     lat (usec): min=34, max=56137, avg=5491.52, stdev=4191.28
    clat percentiles (usec):
     |  1.00th=[   35],  5.00th=[   46], 10.00th=[  143], 20.00th=[  700],
     | 30.00th=[ 1592], 40.00th=[ 3248], 50.00th=[ 5600], 60.00th=[ 7840],
     | 70.00th=[ 9152], 80.00th=[ 9920], 90.00th=[10560], 95.00th=[10944],
     | 99.00th=[12096], 99.50th=[12864], 99.90th=[15808], 99.95th=[22144],
     | 99.99th=[42752]
    bw (KB  /s): min=126208, max=179072, per=25.00%, avg=168713.49, stdev=5997.84
    lat (usec) : 20=0.01%, 50=2.82%, 100=1.68%, 250=1.74%, 500=3.59%
    lat (usec) : 750=3.32%, 1000=3.05%
    lat (msec) : 2=10.18%, 4=14.19%, 10=33.94%, 20=25.43%, 50=0.06%
    lat (msec) : 100=0.01%
  cpu          : usr=2.70%, sys=9.85%, ctx=368799, majf=0, minf=2143
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=100.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued    : total=r=211595/w=210960/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=32

Run status group 0 (all jobs):
   READ: io=40960MB, aggrb=2177.2MB/s, minb=2177.2MB/s, maxb=2177.2MB/s, mint=18814msec, maxt=18814msec

Run status group 1 (all jobs):
   READ: io=29406MB, aggrb=1470.3MB/s, minb=1470.3MB/s, maxb=1470.3MB/s, mint=20001msec, maxt=20001msec

Run status group 2 (all jobs):
  WRITE: io=19572MB, aggrb=976.99MB/s, minb=976.99MB/s, maxb=976.99MB/s, mint=20033msec, maxt=20033msec

Run status group 3 (all jobs):
  WRITE: io=9133.5MB, aggrb=467610KB/s, minb=467610KB/s, maxb=467610KB/s, mint=20001msec, maxt=20001msec

Run status group 4 (all jobs):
   READ: io=13225MB, aggrb=676765KB/s, minb=676765KB/s, maxb=676765KB/s, mint=20010msec, maxt=20010msec
  WRITE: io=13185MB, aggrb=674734KB/s, minb=674734KB/s, maxb=674734KB/s, mint=20010msec, maxt=20010msec

Disk stats (read/write):
  nvme0n1: ios=8065889/3774940, merge=0/0, ticks=6274124/10488887, in_queue=16780920, util=95.98%
```
