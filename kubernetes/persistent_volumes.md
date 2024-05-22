# Persistent volumes

## AWS EFS

With the [EFS CSI Driver](https://github.com/kubernetes-sigs/aws-efs-csi-driver) installed in your cluster, you can mount EFS File Systems into your Pods as PersistentVolumes. The CSI driver is available as an optional component in our reference solution, reach out to your Customer Lead in case you'd like to enable it in any of your clusters.

There are two ways for mounting an existing EFS File System in Kubernetes: the static provisioning and the dynamic provisioning.

> [!NOTE]
> You'll need to allow the EKS nodes access to your EFS file system security group on the NFS tcp port. Normally that would be done through IaC, as all the required IDs and information is readily available in the remote state, but in any case you can check the EKS nodes security group ID by running the following `aws` command:

```console
aws --profile <your-profile> --region <your-region> ec2 describe-security-groups --filters Name=group-name,Values=<cluster-name>-workers --query 'SecurityGroups[0].GroupId' --output text
```

*Replacing `<your-profile>`, `<your-region>` and `<cluster-name>` for the correct values.*

### Static provisioning

With static provisioning, you must create a `PersistentVolume` linked to an existing EFS File System, and a matching `PersistentVolumeClaim`. You can then use the `PersistentVolumeClaim` within your Pod(s). Check the following example:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-e8a95a42
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: efs-app
spec:
  containers:
  - name: app
    image: centos
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date -u) >> /data/out.txt; sleep 5; done"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: efs-claim
```

Note that:

- you must set the EFS File System id in the `volumeHandle` attribute of the `PersistentVolume`
- the `storageClassName` attribute on both the PV and PVC is empty as we're using static provisioning
- the storage requests in the PVC (5Gi) is not used, the driver will ignore that, as EFS doesn't enforce any file system capacity. However, since the storage capacity is a required field by Kubernetes, you must specify the value and you can use any valid value for the capacity.

You can read more about static provisioning in the [driver's official documentation](https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/examples/kubernetes/static_provisioning/README.md).

### Dynamic provisioning

With dynamic provisioning you'll create a `PersistentVolumeClaim` (PVC) to dynamically provision a `PersistentVolume` (PV). On Creating a PVC, Kuberenetes requests EFS to create an Access Point in an EFS file system which will be used to mount the PV.

With this method, you need to create a `StorageClass` with the details to the EFS File system. Check the following example:

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-92107410
  directoryPerms: "700"
  gidRangeStart: "1000" # optional
  gidRangeEnd: "2000" # optional
  basePath: "/dynamic_provisioning" # optional
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: efs-app
spec:
  containers:
    - name: app
      image: centos
      command: ["/bin/sh"]
      args: ["-c", "while true; do echo $(date -u) >> /data/out; sleep 5; done"]
      volumeMounts:
        - name: persistent-storage
          mountPath: /data
  volumes:
    - name: persistent-storage
      persistentVolumeClaim:
        claimName: efs-claim
```

Note that:

- you must set the EFS File System id in the `fileSystemId` parameter of the `StorageClass`
- as with the static provisioning, the requested storage amount is not used

You can read more about dynamic provisioning in the [driver's official documentation](https://github.com/kubernetes-sigs/aws-efs-csi-driver/tree/master/examples/kubernetes/dynamic_provisioning).

## Resizing persistent volumes

We have enabled the option to allow volume expansion by default for the `gp2` and `gp2-encrypted` storageclasses.

Growing the volume happens in 2 steps:

1. [Modify the volume in your PersistentVolumeClaim](https://kubernetes.io/blog/2018/07/12/resizing-persistent-volumes-using-kubernetes/)
2. K8s handles the resizing for you
