# Live Migration

In this page, you are going to see how VM live migration works. We have  an hypervisor cluster and a NAS cluster. The NAS cluster is sharing the VM virtual disk through an NFS server while the hypervisor cluster is running the VM. The connection that you can see in the video is done via RDP (directly to the guest). The guest is playing a youtube video to demonstrate how the migration is done and how much is the client experience affected.



## 1- Hypervisor migration

Here we will do a virtual domain resource move within the hypervisors cluster. The hypervisor cluster has a shared storage exported from nas cluster and during the move the guest os will almos not notice it has been moved from running in one hypervisor to another one.

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

Here we will do a nfs storage cluster resource move from one nas to another while we keep a guest os running on one hypervisor node. 

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

Pretty much impressive we do a live virtual domain resource move from running in one hypervisor to another while we move the shared nfs storage from one nas to another at the same time. Some freeze seconds can be seen on user guest os but the guest continues working as if nothing has happened.

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



## Conclusions

We used two clusters, both set up with pacemaker monitoring resources.

On the storage cluster we used drbd primary/secondary configuration with nfs4 server exports and floating ips. On the hypervisors cluster we have virtual domain resources that will be able to be live migrated. This 'magic' is done by committing the ram memory that will be copied to the new hypervisor while the guest is still running and when it finishes it just pauses the virtual domain for some cents of milliseconds while the ram increment is finally copied. The domain is then started on the other hypervisor with the same memory state and it can continue.

This will allow us to get a high availability cluster where both hypervisors and storage can allow a node to fail without even noticing on clients.