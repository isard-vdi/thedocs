# Disk utilities

## Partition alignment

Unaligned partitions and disk sectors will result in two disk writes for an OS write. 
That will dramatically reduce disk io throughput. To avoid that you can use parted to
create and align partitions.

First we should get the optimal sector where to start and aligned partition with disk
sectors.
```
initial sector=(optimal_io_size+alignment_offset)/physical_block_size
```
And to get the parameters for sdd disk:
```
cat /sys/block/sdd/queue/optimal_io_size
```
```
cat /sys/block/sdd/alignment_offset
```
```
cat /sys/block/sdd/queue/physical_block_size
```
