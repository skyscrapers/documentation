# Using GPU accelerated instances

To leverage GPU accelerated instances in Kubernetes, there are some pre-requisites and steps to follow. These include both configuration of the cluster and the applications docker image.

### Infrastructure pre-requisites

1. A karpenter nodePool with ability to provision GPU accelerated instances, with an appropriate EC2NodeClass with an AMI that uses the correct container runtime. ref: [nvidia container runtime](https://developer.nvidia.com/container-runtime). Amazon EKS optimized AMIs are available with the nvidia container runtime pre-configured. You can check the available EKS optimized AMIs [here]([amazon-eks-node-al2023-x86_64-nvidia-1.31-v2024112](https://awslabs.github.io/amazon-eks-ami/CHANGELOG/)).
2. A device-plugin for the specific GPU type. eg: [NVIDIA GPU device plugin](https://github.com/NVIDIA/k8s-device-plugin) or [AWS neuron device plugin](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/containers/tutorials/k8s-setup.html). This runs as a DaemonSet on the cluster, and exposes the GPU resources to the cluster.

To test the setup, you can use a simple test pod that will run on a GPU node.

[ref: NVIDIA CUDA sample](https://github.com/NVIDIA/gpu-operator/blob/main/tests/gpu-pod.yaml)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  restartPolicy: OnFailure
  containers:
    - name: cuda-vector-add
      # https://catalog.ngc.nvidia.com/orgs/nvidia/teams/k8s/containers/cuda-sample
      image: "nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04"
      resources:
        limits:
          nvidia.com/gpu: 1
```

### Application pre-requisites

The application must be built with the correct CUDA libraries and drivers for the GPU type. This is easiest by using a base image that has the correct libraries installed and configured. For NVIDIA GPUs, the [nvidia/cuda](https://hub.docker.com/r/nvidia/cuda) images are a good starting point. To run neuron based applications, the [AWS Neuron](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/containers/docker-example/inference/Dockerfile-libmode.html#libmode-dockerfile) dockerfile reference is a good starting point.

### Testing the GPU availability in the application

To test the GPU availability in the application, for CUDA, you can run a script that checks the availability of the GPU device:

```python
import torch
print(torch.cuda.is_available())
```

For AWS Neuron, you can run a script that checks the availability of the Neuron device:

```bash
neuron-top
```
