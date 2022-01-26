- [Introduction](#introduction)
- [Installing OCS Using Local Storage Devices](#installing-ocs-using-local-storage-devices)
- [Uninstalling OCS Using Local Storage Devices](#uninstalling-ocs-using-local-storage-devices)

## Introduction
Installing OpenShift Container Storage (OCS) on OCP 4.x.

In this document, there are steps applied to install OCS on OCP 4.6.  

References:
<br>
https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.6/html/deploying_openshift_container_storage_using_bare_metal_infrastructure/index
<br>
https://www.openshift.com/blog/blog-using-local-disks-on-vmware-with-openshift-container-storage
<br>
https://sizer.ocs.ninja/

Topology:

* Installation server with following components
  - DNS server for OCP 4.x
  - Local Mirror for offline installation
* 1 load balancer for the control control plane (master nodes)
* 1 load balancer for the compute nodes (worker nodes)
* 3 control plane nodes (master nodes)
* 5 compute nodes (worker nodes)
* 3 compute nodes for OCS
* 18 500 GB VM Disks (Datastore)

<br>


## Installing OCS Using Local Storage Devices

1. Find out which worker nodes you will use as OCS node. There should be at least 3 nodes available for OCS deployment

    ```
    $ oc get nodes
        NAME                                         STATUS   ROLES    AGE   VERSION
        <master 1>.<OCPDomainPrefix>.<BaseDomain>   Ready    master   21d   v1.16.2
        <master 2>.<OCPDomainPrefix>.<BaseDomain>   Ready    master   21d   v1.16.2
        <master 3>.<OCPDomainPrefix>.<BaseDomain>   Ready    master   21d   v1.16.2
        <worker 1>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
        <worker 2>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
        <worker 3>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
        <worker 4>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
        <worker 5>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
        <worker 6>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
        <worker 7>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
        <worker 8>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2        
    ```

2. Add following labels and taint to the worker nodes you will add local devices. In this example worker nodes 6,7,8 are dedicated nodes for OCS.

    * Label will be added during OCP deployment at 4.6 version. But it is suggested to add manually.
  
    ```
    
    $ oc label nodes <node>  cluster.ocs.openshift.io/openshift-storage=''
    node/<node>  labeled
    $ oc label nodes <node>  node-role.kubernetes.io/infra=''
    node/<node> labeled

    $ oc adm taint node <node> node.ocs.openshift.io/storage="true":NoSchedule
    node/<node>  tainted    

    $ oc get nodes -l cluster.ocs.openshift.io/openshift-storage=
    NAME                                         STATUS   ROLES    AGE   VERSION
    <worker 3>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
    <worker 4>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
    <worker 5>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
    ```

3. Install OCS operator using Operator HUB

    - Click Operators → OperatorHub in the left pane of the OpenShift Web Console.
    - Find and click install for OpenShift Container Storage Operator page.
    - Notice that it will create namespace **openshift-storage**
    - Enable cluster monitoring
    - Pick Automatic Approval strategy
    - Click install

4. Install Local Storage operator using Operator HUB

    - Click Operators → OperatorHub in the left pane of the OpenShift Web Console.
    - Find and click install for Local Storage Operator page.
    - Notice that it will create namespace **openshift-local-storage**
    - Pick Automatic Approval strategy
    - Click install

5. Navigate OCS Operator from web console. Click Create Instance under Storage Cluster Tab. Pick **Internal - Attached Devices** and select disk attached nodes.
   * Important Note: Be sure disks are attached with ID's On VmWare platforms hosts may not have proper setting to attach disks to operating system with UUID's. Following configuration can be added with "Advanced Configuration" on each worker node.
      ```
      disk.EnableUUID TRUE
      ```

6. Pick nodes where block devices were attached.

7. Name storage class for local block devices (for example: localblock). Set a volume set name (for example: ocs-deviceset). OCS service capacity should be disk size. (in this example it is 500Gb )

8. Pick storage class defined previous step and wait some time to see nodes with disks. Click Create to install OCS.
   
9. Deployment will take around 10 minutes.
   
10. When deployment ends if you have only one deviceset deployed. You can edit and change it to the device set you need. In this documentation, Environment has 6 devices with 500GB each per worker node. Device set count should be 6.
  ```
  # oc get storagecluster/ocs-storagecluster -o yaml | grep -A 3 storageDeviceSets
        f:storageDeviceSets: {}
    manager: Mozilla
    operation: Update
    time: "2021-08-26T08:42:21Z"
    ....
  storageDeviceSets:
  - config: {}
    count: 1
    dataPVCTemplate:

  # oc edit storagecluster/ocs-storagecluster
  storagecluster.ocs.openshift.io/ocs-storagecluster edited  
  ```

11. Wait to see Storage is green on Web interface. Status of storage deployment can be checked through CLI also.
    ```

    ```

## Uninstalling OCS Using Local Storage Devices

1) Follow the steps mentioned at redhat official documentation

    https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.6/html-single/deploying_openshift_container_storage_using_bare_metal_infrastructure/index#assembly_uninstalling-openshift-container-storage_rhocs

2) Delete hung CRDs if exists.
    - List hung CRD's
    ```
    $ oc get crd | grep -i ceph
    cephclusters.ceph.rook.io                                   2020-06-04T13:11:50Z
    $ oc get crd | grep -i localvolumes
    localvolumes.local.storage.openshift.io                     2020-06-03T09:35:37Z
    ```

    - Set finalizer to none

    ```
    $ oc patch crd/cephclusters.ceph.rook.io -p '{"metadata":{"finalizers":[]}}' --type=merge
    $ oc patch crd/localvolumes.local.storage.openshift.io  -p '{"metadata":{"finalizers":[]}}' --type=merge
    ```

3) If exists if you have following error, login to each worker node with OCS label and delete remaining links:

    ```error creating symlink /mnt/local-storage/localblock/sdc: <nil>```

    ```
    # cd /mnt/local-storage/localblock/
    # ls
    sdb  sdc  sdd  sde  sdf  sdg  sdh
    # rm -Rf *
    ```
