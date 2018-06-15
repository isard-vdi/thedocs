# Live Migration

In this page, you are going to see how VM live migration works. We have  an hypervisor cluster and a NAS cluster. The NAS cluster is sharing the VM virtual disk through an NFS server while the hypervisor cluster is running the VM. The connection that you can see in the video is done via RDP (directly to the guest). The guest is playing a youtube video to demonstrate how the migration is done and how much is the client experience affected.



## 1- Hypervisor migration

```
          HYPERVISORS                                      NAS

+--------+          +--------+               +--------+           +--------+
|        |          |        |               |        |           |        |
| vnode2 |          | vnode3 |               |  nas1  |           |  nas2  |
|        |          |        |               |        |           |        |
|        |          |        |               |        |           |        |
|  (vm)  |          |        |               |        |           |  (vm)  |
|        |          |        |               |        |           |        |
+---+----+          +----+---+               +--------+           +--------+
    |                    ^
    |                    |
    +--------------------+
```

<iframe width="560" height="315" src="https://www.youtube.com/embed/mbgg8kDrT0s" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>



## 2- NAS migration

```
          HYPERVISORS                                      NAS

+--------+          +--------+               +--------+           +--------+
|        |          |        |               |        |           |        |
| vnode2 |          | vnode3 |               |  nas1  |           |  nas2  |
|        |          |        |               |        |           |        |
|        |          |        |               |        |           |        |
|        |          |  (vm)  |               |        |           |  (vm)  |
|        |          |        |               |        |           |        |
+---+----+          +----+---+               +--------+           +--------+
                                                 ^                    |
                                                 |                    |
                                                 +--------------------+
```

<iframe width="560" height="315" src="https://www.youtube.com/embed/HfD5VG1mgCM" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>



## 3- Hypervisor and NAS migration

```
          HYPERVISORS                                      NAS

+--------+          +--------+               +--------+           +--------+
|        |          |        |               |        |           |        |
| vnode2 |          | vnode3 |               |  nas1  |           |  nas2  |
|        |          |        |               |        |           |        |
|        |          |        |               |        |           |        |
|        |          |  (vm)  |               |  (vm)  |           |        |
|        |          |        |               |        |           |        |
+---+----+          +----+---+               +--------+           +--------+
    ^                    |                       |                    ^
    |                    |                       |                    |
    +--------------------+                       +--------------------+
```

<iframe width="560" height="315" src="https://www.youtube.com/embed/nfYFtSYO2DI" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>