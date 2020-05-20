# Persistent volumes

## Resizing persistent volumes

We have enabled the option to allow volume expansion by default for the `gp2` and `gp2-encrypted` storageclasses.

Growing the volume happens in 2 steps:

1. [Modify the volume in your PersistentVolumeClaim](https://kubernetes.io/blog/2018/07/12/resizing-persistent-volumes-using-kubernetes/)
2. manually extend the filesystem in your pod
