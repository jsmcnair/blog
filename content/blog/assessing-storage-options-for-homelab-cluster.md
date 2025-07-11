Within the specific context of deploying on a cluster at home and the following ideals/requirements:

* Simple deployment - not too many components to deploy/manage
* Lightweight - does not consume a lot of resources
* Low configuration complexity - not having to many underlying pre-requisites (eg. extra kernel modules needed)

OpenEBS Mayastor - has many components (etcd/NATS,etc.). Also requires to take full control of a disk/partition but Talos does not support this (User volumes can only be formatted XFS or EXT4).
Longhorn - Requires iSCSI extensions for Talos and uses NFS server for RWX
Rook/Ceph - Resource hungry unless tuned below recommendations

Useful reference: https://kubedo.com/kubernetes-storage-comparison/
