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

.. raw:: html

<video controls>

​	<source src="live-migration/migration_hypervisor.mp4" type="video/mp4">

</video>





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

.. raw:: html

<video controls>

​	<source src="live-migration/migration_nas.mp4" type="video/mp4">

</video>



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

.. raw:: html

<video controls>

​	<source src="live-migration/migration_desfase.mp4" type="video/mp4">

</video>