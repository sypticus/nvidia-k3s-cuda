# nvidia-k3s-cuda
## Work in Progress!

This is a set of instructions and scripts that will allow you to access nvidia GPU resources via Cuda from your K3s pods.

These instructions assume you have a K3S cluster already deployed. For help setting one up see ......

It is also assumed that you are using a Debian-based OS on the host which has the GPU(s), and that you have sudo access


## Host Setup
Each host with GPU resources needs to have the Cuda drivers.



```console
sudo apt update 
sudo apt upgrade -y
sudo apt-get install linux-headers-$(uname -r)
```


### (optional) Remove all current NVIDIA and Cuda resource.
If you have current drivers which are not working, or get into a broken state, purge the current nvidia and cuda drivers and start over.

```console
sudo apt-get --purge remove "*cublas*" "cuda*" "nsight*" 
sudo apt-get --purge remove "*nvidia*"


Restart the host
```

### Install NVIDIA and Cuda Drivers
Installing the Cuda package from Nvidia should bring in all needed NVIDIA drivers. If for some reason it is not able to do so, you can install the drivers [directly](https://www.leadergpu.com/articles/499-install-nvidia-drivers-in-linux)

```console
distribution=$(. /etc/os-release;echo $ID$VERSION_ID | sed -e 's/\.//g')
wget https://developer.download.nvidia.com/compute/cuda/repos/$distribution/x86_64/cuda-keyring_1.1-1_all.deb

sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
sudo apt-get install cuda-drivers
```


At this point you should reboot the host.

### Confirm the drivers were successful

At this point you should have all of the NVidia and Cuda drivers installed.
```console
nvidia-smi
```
This should show info about your device.

```console
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 565.57.01              Driver Version: 565.57.01      CUDA Version: 12.7     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 3060 Ti     On  |   00000000:02:00.0 Off |                  N/A |
|  0%   55C    P8             20W /  200W |       2MiB /   8192MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
                                                                                         
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```


### Install the Nvidia Container Toolkit

#### Verify and add the source
```console
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg  &&  \
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list |   \
sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

#### Install the toolkit
```Console
sudo apt-get update
sudo apt-get install nvidia-container-toolkit
```

#### Verify install
From here, most Kubernetes installs would run `nvidia-ctk runtime configure --runtime=containerd`, however, k3s handles `config.toml` a bit differently.
Instead, K3s will automatically add the plugin and add the runtimeclass.

Restart the K3S agent.
```Console
sudo systemctl restart k3s-agent.service
```
Once it has restarted, confirm the config.toml has been updated:

```Console
sudo grep nvidia /var/lib/rancher/k3s/agent/etc/containerd/config.toml"
```

```Console
------
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes."nvidia"]
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes."nvidia".options]
BinaryName = "/usr/bin/nvidia-container-runtime"
```

### Compute Capability
Lastly, get the compute capability for your card. This can also be queried directly:
```console
nvidia-smi --query-gpu=compute_cap --format=csv
```
This can also be found on a table [nvidia's site] (https://developer.nvidia.com/cuda-gpus)
You will need to save this value for later.


### Kubernetes Config

In order to access the GPU with CUDA, you will need to make a few changes in Kubernetes

First you will need to create the runtime class. Create a new file `nvidia-runtimeclass.yaml` and add the following.
```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia
handler: nvidia
```
This file should be run either as part of a helm chart of with 
```console
kubectl apply -f nvidia-runtimeclass.yaml
```

From here, you will need to set the `spec.runtimeClassName` on any pods you want to work with Cuda

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myLLMCudaPod
spec:
  runtimeClassName: nvidia
  # ...
```
 
Also you will need to ensure that the following environmental vars are set on each container that needs access to the GPU

```console
TORCH_CUDA_ARCH_LIST: "$THE_COMPUTECAP_FROM_ABOVE_QUERY"
NVIDIA_VISIBLE_DEVICES: "all"
NVIDIA_DRIVER_CAPABILITIES: "all"
```
so for example: 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myLLMCudaPod
spec:
  runtimeClassName: nvidia
  containers:
  - name: llm-container
    image: gcr.io/google-samples/hello-app:2.0
    env:
    - name: TORCH_CUDA_ARCH_LIST
      value: "8.6"
    - name: NVIDIA_VISIBLE_DEVICES
      value: "all"
    - name: NVIDIA_DRIVER_CAPABILITIES 
    value: "all"
```


### Test it out
Download the attached `test-nvidia-gpu-pod.yaml`

```commandline
kubectl apply -f test-nvidia-gpu-pod.yaml
```

This should start up a pod, the logs of which should show a successful nvidia-smi call.

```commandline
kubectl logs nvidia-test-pod


+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 565.57.01              Driver Version: 565.57.01      CUDA Version: 12.7     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 3060 Ti     On  |   00000000:02:00.0 Off |                  N/A |
|  0%   55C    P8             19W /  200W |       2MiB /   8192MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
                                                                                         
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+

```