# Openshift 4.x Restart

- [Openshift 4.x Restart](#openshift-4x-restart)
  - [Introduction](#introduction)
  - [Shut Down Cluster](#shut-down-cluster)
  - [Restart Cluster](#restart-cluster)
  
## Introduction

In this document, there are steps applied to shut down Openshift cluster gracefully and restarting up properly.

References:<br>
https://docs.openshift.com/container-platform/4.6/backup_and_restore/graceful-cluster-shutdown.html<br>
https://docs.openshift.com/container-platform/4.6/backup_and_restore/graceful-cluster-restart.html#graceful-restart-cluster<br>
https://docs.openshift.com/container-platform/4.6/backup_and_restore/backing-up-etcd.html<br>

Topology:

* Installation server with following components
  - DNS server for OCP 4.x
  - Squid to use as proxy to OCP
* 1 load balancer for the control control plane (master nodes)
* 1 load balancer for the compute nodes (worker nodes)
* 1 Bootstrap node
* 3 control plane nodes (master nodes)
* 5 compute nodes (worker nodes)
* 3 compute nodes for OCS

<br>

## Shut Down Cluster

1. Take etcd backup from one of the master nodes.
   ```
    $ oc get nodes
    NAME                                         STATUS   ROLES          AGE    VERSION
    master1   Ready    master         3d6h   v1.19.0+d856161
    master2   Ready    master         3d6h   v1.19.0+d856161
    master3   Ready    master         3d6h   v1.19.0+d856161
    worker1  Ready    worker         3d5h   v1.19.0+d856161
    ......

    $ oc debug node/master1
      # chroot /host
      # /usr/local/bin/cluster-backup.sh /home/core/assets/backup
      # cp /home/core/assets/backup/* /tmp/
   ```

2. Get etcd backups to installation server where ssh keys were deployed to RHCOS nodes.
   ```
   $ scp core@master1:/tmp/snapshot_2021-08-27_114834.db .
   $ scp core@master1:/tmp/static_kuberesources_2021-08-27_114834.tar.gz .
   ```

3. Collect must-gather for OCP and OCS in case Red Hat support will be needed. 

  ```
  $ oc adm must-gather
  $ oc adm must-gather --image=registry.redhat.io/ocs4/ocs-must-gather-rhel8:v4.6
  ```

4. Take snapshots of virtual machines RedHat CoreOS servers are running. Can be done from Vcenter or ask your admin.

5. [Optional] It is always good to have application backups as well.

6. Shutdown cluster nodes.

  ```
  $ nodes=$(oc get nodes -o jsonpath='{.items[*].metadata.name}')
  $ for node in ${nodes[@]}
    do
        echo "==== Shut down $node ===="
        ssh core@$node sudo shutdown -h 1
    done
  ```

7. If any server stalls more than 5 minutes, you can stop it from Vsphere directly.

## Restart Cluster

1. Be sure all depended servers are up and running. Like DNS, LoadBalancers etc

2. Start all cluster machines

3. Monitor cluster with following command.
   ```
   $ watch -n5 'oc get clusteroperators ; oc get nodes ; oc get clusterversion'
   ```
4. In case of any stucked node, please check CSR's waiting for approval and approve if needed.
    ```
    $ oc get csr
    $ oc adm certificate approve <csr_name>
    ```
5. Check deployed applications one by one to be sure. 

----


