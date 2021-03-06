## `kops` hook for NVIDIA GPU Driver and DevicePlugin Installation

This kops hook container may be used to enable nodes with GPUs to work with Kubernetes.

It installs the following from web sources.

1. [Nvidia Device Drivers](http://www.nvidia.com/Download/index.aspx)
2. [nvidia-docker](https://github.com/NVIDIA/nvidia-docker)

### How it works

* This kops hook container runs on a kubernetes node upon every boot.
* It installs onto the host system a systemd oneshot service unit `nvidia-device-plugin.service` along with setup scripts.
* The systemd unit `nvidia-device-plugin.service` runs and executes the setup scripts in the host directory `/nvidia-device-plugin`.
* The scripts install the Nvidia device drivers and Nvidia docker.

### Using this DevicePlugin

#### Create a Cluster with GPU Nodes

```bash

kops version
Version 1.16.0 (git-4b0e62b82)

export KOPS_STATE_STORE=s3://some-s3-backet-name

kops create cluster \
    --cloud aws \
    --zones eu-west-1a,eu-west-1b,eu-west-1c \
    --master-zones eu-west-1a \
    --networking calico \
    --master-size m5.large \
    --node-size g4dn.xlarge \
    --node-count 1 \
    gpu.k8s.local

```

#### Enable the Kops Installation Hook and DevicePlugins

This should be safe to do for all machines, because the hook auto-detects if the machine has NVIDIA GPU installed and will NO-OP otherwise.

```yaml
kops edit ig --name=gpu.k8s.local nodes

spec:
  hooks:
  - execContainer:
      image: pure/nvidia-device-plugin:tesla
```


#### Update the cluster

```bash
kops update cluster --name gpu.k8s.local --yes
while ! kops validate cluster --name gpu.k8s.local; do sleep 10; done

```

#### Deploy the Daemonset for the Nvidia DevicePlugin

```bash
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/1.0.0-beta4/nvidia-device-plugin.yml
```

#### Check Node capacity for `nvidia.com/gpu`

```json
kubectl get no -l beta.kubernetes.io/instance-type=g4dn.xlarge -ojson | jq '.items[].status.capacity'
{
  "attachable-volumes-aws-ebs": "39",
  "cpu": "4",
  "ephemeral-storage": "125753328Ki",
  "hugepages-1Gi": "0",
  "hugepages-2Mi": "0",
  "memory": "16133788Ki",
  "nvidia.com/gpu": "1",
  "pods": "110"
}

```

### Validate that GPUs are Working

#### Deploy a Test Pod

```bash
cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: tf-gpu
spec:
  terminationGracePeriodSeconds: 3
  containers:
  - name: gpu
    image: tensorflow/tensorflow:latest-gpu
    imagePullPolicy: IfNotPresent
    args: ["sleep", "1d"]
    env:
    - name: TF_CPP_MIN_LOG_LEVEL
      value: "3"
    resources:
      limits:
        memory: 1024Mi
        nvidia.com/gpu: 1 # requesting 1 GPUs
EOF
```

#### Show GPU info from within the pod

```
kubectl exec -it tf-gpu -- nvidia-smi

+-----------------------------------------------------------------------------+
| NVIDIA-SMI 440.64.00    Driver Version: 440.64.00    CUDA Version: 10.2     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla T4            On   | 00000000:00:1E.0 Off |                    0 |
| N/A   37C    P8     9W /  70W |      0MiB / 15109MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+

```

####  Show Tensorflow detects GPUs from within the pod

```
kubectl exec -it tf-gpu -- python -c 'from tensorflow.python.client import device_lib; print(device_lib.list_local_devices())'

[name: "/device:CPU:0"
device_type: "CPU"
memory_limit: 268435456
locality {
}
incarnation: 3121764385910360567
, name: "/device:XLA_CPU:0"
device_type: "XLA_CPU"
memory_limit: 17179869184
locality {
}
incarnation: 18396904841851242797
physical_device_desc: "device: XLA_CPU device"
, name: "/device:XLA_GPU:0"
device_type: "XLA_GPU"
memory_limit: 17179869184
locality {
}
incarnation: 4503292422335862858
physical_device_desc: "device: XLA_GPU device"
, name: "/device:GPU:0"
device_type: "GPU"
memory_limit: 14941647668
locality {
  bus_id: 1
  links {
  }
}
incarnation: 6949372932394118638
physical_device_desc: "device: 0, name: Tesla T4, pci bus id: 0000:00:1e.0, compute capability: 7.5"
]

```

#### Detele Test Pod

```bash
kubectl delete pod tf-gpu

```

### Delete the test cluster

Running a Kubernetes cluster within AWS obviously costs money, and so you may want to delete your cluster if you are finished running experiments.

```bash
kops delete cluster --name gpu.k8s.local --yes
```
