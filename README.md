# NVIDIA container runtime for Wind River Linux

## Introduction
Training and using AI models are tasks that demand significant computational power. Current trends are pointing more to deep neural networks, which include thousands, if not millions of operations per iteration.
In the past year, more and more researchers have sounded the alarm on the exploding costs of deep learning.
The computing power needed to do AI is now rising seven times faster than ever before **[1]**.
These new needs are making hardware companies create hardware accelerators like Neural processing units, CPUs, and GPUS.


Embedded systems are not an exception to this transformation. We see every day intelligent traffic lights, autonomous vehicles, intelligent IoT devices, and more.
The current direction is to have accelerators inside these embedded devices, Systems On-Chip mainly.
Hardware developers have embedded small accelerators like GPUs, FPGAs, and more into SOCs, SOMs, and other systems.
We call these modern systems: heterogeneous computing architectures.

The use of GPUs on Linux is not something new; we have been able to do so for many years. However, it would be great to accelerate the development and deployment of  HPC applications. Containers enable portability, stability, and many other characteristics when deploying an application. For this reason, companies are investing so much in these technologies. For instance, NVIDIA recently started a project that enables CUDA on Docker **[2]**. 

One concern when dealing with containers is the loss of performance. However, when comparing the performance of the GPU with and without the containers environment, researchers found that no additional overhead is caused **[3]**.
The consistency in the performance is one of the principal benefits of containers over virtual machines; accessing the GPU is done seamlessly as the kernel stays the constant.

## NVIDIA-Docker on Yocto


Together with Matt Madison  (Maintainer of meta-tegra layer), we created the required recipes to build and deploy NVIDIA-docker on Wind River Linux LTS 19 (Yocto 3.0 Zeus).**[4]**

In this tutorial, you will find how to enable NVIDIA-containers on a custom distribution of Linux and run a small test application that leverages the use of GPUs inside a container.




## Description

To enable NVIDIA containers, Docker needs to have the nvidia-containers-runtime which is a modified version of runc that adds a custom pre-start hook to all containers. The nvidia-containers-runtime communicates docker using the library libnvidia-container, which automatically configures GNU/Linux containers leveraging NVIDIA hardware. This library relies on kernel primitives and is designed to be agnostic of the container runtime. All the effort to port these libraries and tools to Yocto was submitted to the community and now is part of the meta-tegra layer which is maintained by Matt Madison.

Note: this setup is based on Linux for Tegra and not the original Yocto Linux Kernel

## Benefits, and Limitations
The main benefit of GPUs inside containers is the portability and stability in the environment at the time of deployment. Of course, the development also sees benefits in having this portable environment as developers can collaborate more efficiently.
However, there are limitations due to the nature of the NVIDIA environment. Containers are heavy-weight because they are based in Linux4Tegra image that contains libraries required on runtime. On the other hand, because of redistribution limitations, some libraries are not included in the container. This requires runc to mount some property code libraries, losing portability in the process.

## Prerequisites
You are required to download NVIDIA property code from their website. To do so, you will need to create an NVIDIA Developer Network aacount.

Go into https://developer.nvidia.com/embedded/downloads , download the NVIDIA SDK Manager, install it and download all the files for the Jetson board you own. All the effort to port these libraries and tools to Yocto was submited to the community and now is part of the meta-tegra layer which is maintained by Matt Madison.
The required Jetpack version is 4.3
```bash
/opt/nvidia/sdkmanager/sdkmanager
```




![SDK Manager](dowload_jetpack_4.3.PNG)
 **Image 1. SDK Manager installation**  
 
 
If you need to include TensorRT in your builds, you must create the subdirectory and move all of the TensorRT packages downloaded by the SDK Manager there.

 ```bash
 $ mkdir /home/$USER/Downloads/nvidia/sdkm_downloads/NoDLA
 $ cp /home/$USER/Downloads/nvidia/sdkm_downloads/libnv* /home/$USER/Downloads/nvidia/sdkm_downloads/NoDLA
 ```
 
 
 
 
## Creating the project

```bash
git clone --branch WRLINUX_10_19_BASE https://github.com/WindRiver-Labs/wrlinux-x.git
./wrlinux-x/setup.sh --all-layers --dl-layers --templates feature/docker
```

Note: --distro wrlinux-graphics can be used for some applications that require x11.

### Add meta-tegra layer
**DISCAIMER: meta-tegra is a community maintained layer not supported by WindRiver at the time of writing** 
```bash
git clone https://github.com/madisongh/meta-tegra.git layers/meta-tegra
cd layers/meta-tegra
git checkout 11a02d02a7098350638d7bf3a6c1a3946d3432fd
cd -
```
**Tested with: https://github.com/madisongh/meta-tegra/commit/11a02d02a7098350638d7bf3a6c1a3946d3432fd**
```bash
. ./environment-setup-x86_64-wrlinuxsdk-linux
. ./oe-init-build-env
```

```bash
bitbake-layers add-layer ../layers/meta-tegra/
bitbake-layers add-layer ../layers/meta-tegra/contrib
```
## Configure the project
```bash
echo "BB_NO_NETWORK = '0'" >> conf/local.conf
echo 'INHERIT_DISTRO_remove = "whitelist"' >> conf/local.conf
```
### Set the machine to your Jetson Board
```bash
echo "MACHINE='jetson-nano-qspi-sd'" >> conf/local.conf
echo "PREFERRED_PROVIDER_virtual/kernel = 'linux-tegra'" >> conf/local.conf
 ```
 
CUDA cannot be compiled with GCC versions higher than 7. Set GCC version to 7.%:
 ```bash
echo 'GCCVERSION = "7.%"' >> conf/local.conf
echo "require contrib/conf/include/gcc-compat.conf" >> conf/local.conf
```
Set the IMAGE export type to tegraflash for ease of deployment.
```bash 
echo 'IMAGE_CLASSES += "image_types_tegra"' >> conf/local.conf
echo 'IMAGE_FSTYPES = "tegraflash"' >> conf/local.conf
```
Change the docker version, add nvidia-container-runtime.
```bash
echo 'IMAGE_INSTALL_remove = "docker"' >> conf/local.conf
echo 'IMAGE_INSTALL_append = " docker-ce"' >> conf/local.conf
```
Fix tini build error 
```bash
echo 'SECURITY_CFLAGS_pn-tini_append = " ${SECURITY_NOPIE_CFLAGS}"' >> conf/local.conf
```
Set NVIDIA download location
```bash
echo "NVIDIA_DEVNET_MIRROR='file:///home/$USER/Downloads/nvidia/sdkm_downloads'" >> conf/local.conf
echo 'CUDA_BINARIES_NATIVE = "cuda-binaries-ubuntu1604-native"' >> conf/local.conf
```
Add the Nvidia containers runtime, AI libraries and the AI libraries CSV files
```bash 
echo 'IMAGE_INSTALL_append = " nvidia-docker nvidia-container-runtime cudnn tensorrt libvisionworks libvisionworks-sfm libvisionworks-tracking cuda-container-csv cudnn-container-csv tensorrt-container-csv libvisionworks-container-csv libvisionworks-sfm-container-csv libvisionworks-tracking-container-csv"' >> conf/local.conf
```
Enable ldconfig required by the nvidia-container-runtime
```bash
echo 'DISTRO_FEATURES_append = " ldconfig"' >> conf/local.conf

```
## Build the project
```bash
bitbake wrlinux-image-glibc-std
```

## Burn the image into the SD card
```bash
unzip wrlinux-image-glibc-std-sato-jetson-nano-qspi-sd-20200226004915.tegraflash.zip -d wrlinux-jetson-nano
cd wrlinux-jetson-nano

```
Connect the Jetson Board to your computer using the microusb as shown in the image:



![Jetson Nano Setup](jetson_nano_pins_setup_photo.jpg)  
**Image 2. Recovery mode setup for Jetson Nano**




![Jetson Nano Pins](jetson_nano_pins.PNG)  


**Image 3. Pins Diagram for Jetson Nano**




After connecting the board, run:

```bash
sudo ./dosdcard.sh
```
This command will create the file `wrlinux-image-glibc-std.sdcard` that contains the SD card image required to boot.

Burn the Image to the SD Card:
```bash
sudo dd if=wrlinux-image-glibc-std.sdcard of=/dev/***** bs=8k
```
**Warning: substitute the of= device to the one that points to your sdcard**
**Failure to do so can lead to unexpected erase of hard disks**


## Deploy the target

Boot up the board and find the ip address with the command `ifconfig`.

Then, ssh into the machine and run docker:

```bash
$ ssh root@<ip_address>
```

Run the container:
```bash
# docker run --runtime nvidia -it paroque28/l4t-tensorflow python3 ./tensorflow_demo.py
```
## Results
```bash
# docker run --runtime nvidia -it paroque28/l4t-tensorflow python3 ./tensorflow_demo.py
2020-04-22 21:13:47.003164: I tensorflow/stream_executor/platform/default/dso_loader.cc:42] Successfully opened dynamic library libcudart.so.10.0
Instructions for updating:
Call initializer instance with the dtype argument instead of passing it to the constructor
# Fit model on training data
Train on 50000 samples, validate on 10000 samples
2020-04-22 21:13:56.632375: W tensorflow/core/platform/profile_utils/cpu_utils.cc:98] Failed to find bogomips in /proc/cpuinfo; cannot determine CPU frequency
2020-04-22 21:13:56.633182: I tensorflow/compiler/xla/service/service.cc:168] XLA service 0x2e6e7d10 executing computations on platform Host. Devices:
2020-04-22 21:13:56.633240: I tensorflow/compiler/xla/service/service.cc:175]   StreamExecutor device (0): <undefined>, <undefined>
2020-04-22 21:13:56.641913: I tensorflow/stream_executor/platform/default/dso_loader.cc:42] Successfully opened dynamic library libcuda.so.1
2020-04-22 21:13:56.774096: I tensorflow/stream_executor/cuda/cuda_gpu_executor.cc:972] ARM64 does not support NUMA - returning NUMA node zero
2020-04-22 21:13:56.774427: I tensorflow/compiler/xla/service/service.cc:168] XLA service 0x2fcb32d0 executing computations on platform CUDA. Devices:
2020-04-22 21:13:56.774484: I tensorflow/compiler/xla/service/service.cc:175]   StreamExecutor device (0): NVIDIA Tegra X1, Compute Capability 5.3
2020-04-22 21:13:56.775698: I tensorflow/stream_executor/cuda/cuda_gpu_executor.cc:972] ARM64 does not support NUMA - returning NUMA node zero
2020-04-22 21:13:56.776901: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1640] Found device 0 with properties: 
name: NVIDIA Tegra X1 major: 5 minor: 3 memoryClockRate(GHz): 0.9216
pciBusID: 0000:00:00.0
2020-04-22 21:13:56.777011: I tensorflow/stream_executor/platform/default/dso_loader.cc:42] Successfully opened dynamic library libcudart.so.10.0
2020-04-22 21:13:56.801153: I tensorflow/stream_executor/platform/default/dso_loader.cc:42] Successfully opened dynamic library libcublas.so.10.0
2020-04-22 21:13:56.830989: I tensorflow/stream_executor/platform/default/dso_loader.cc:42] Successfully opened dynamic library libcufft.so.10.0
2020-04-22 21:13:56.843623: I tensorflow/stream_executor/platform/default/dso_loader.cc:42] Successfully opened dynamic library libcurand.so.10.0
2020-04-22 21:13:56.876305: I tensorflow/stream_executor/platform/default/dso_loader.cc:42] Successfully opened dynamic library libcusolver.so.10.0
2020-04-22 21:13:56.895317: I tensorflow/stream_executor/platform/default/dso_loader.cc:42] Successfully opened dynamic library libcusparse.so.10.0
2020-04-22 21:13:56.968629: I tensorflow/stream_executor/platform/default/dso_loader.cc:42] Successfully opened dynamic library libcudnn.so.7
2020-04-22 21:13:56.968924: I tensorflow/stream_executor/cuda/cuda_gpu_executor.cc:972] ARM64 does not support NUMA - returning NUMA node zero
2020-04-22 21:13:56.969203: I tensorflow/stream_executor/cuda/cuda_gpu_executor.cc:972] ARM64 does not support NUMA - returning NUMA node zero
2020-04-22 21:13:56.969319: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1763] Adding visible gpu devices: 0
2020-04-22 21:13:56.969455: I tensorflow/stream_executor/platform/default/dso_loader.cc:42] Successfully opened dynamic library libcudart.so.10.0
2020-04-22 21:13:58.209433: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1181] Device interconnect StreamExecutor with strength 1 edge matrix:
2020-04-22 21:13:58.209532: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1187]      0 
2020-04-22 21:13:58.209560: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1200] 0:   N 
2020-04-22 21:13:58.209928: I tensorflow/stream_executor/cuda/cuda_gpu_executor.cc:972] ARM64 does not support NUMA - returning NUMA node zero
2020-04-22 21:13:58.210412: I tensorflow/stream_executor/cuda/cuda_gpu_executor.cc:972] ARM64 does not support NUMA - returning NUMA node zero
2020-04-22 21:13:58.210600: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1326] Created TensorFlow device (/job:localhost/replica:0/task:0/device:GPU:0 with 268 MB memory) -> physical GPU (device: 0, name: NVIDIA Tegra X1, pci bus id: 0000:00:00.0, compute capability: 5.3)
Epoch 1/3
2020-04-22 21:13:59.253255: I tensorflow/stream_executor/platform/default/dso_loader.cc:42] Successfully opened dynamic library libcublas.so.10.0
50000/50000 [==============================] - 13s 269us/sample - loss: 0.3386 - sparse_categorical_accuracy: 0.9043 - val_loss: 0.1752 - val_sparse_categorical_accuracy: 0.9488
Epoch 2/3
50000/50000 [==============================] - 11s 226us/sample - loss: 0.1648 - sparse_categorical_accuracy: 0.9516 - val_loss: 0.1322 - val_sparse_categorical_accuracy: 0.9619
Epoch 3/3
50000/50000 [==============================] - 11s 226us/sample - loss: 0.1211 - sparse_categorical_accuracy: 0.9630 - val_loss: 0.1418 - val_sparse_categorical_accuracy: 0.9596

history dict: {'loss': [0.33861770302057265, 0.1648293803167343, 0.12105341740369797], 'sparse_categorical_accuracy': [0.90434, 0.95156, 0.96302], 'val_loss': [0.17516357889175416, 0.1322396732479334, 0.14180631960630416], 'val_sparse_categorical_accuracy': [0.9488, 0.9619, 0.9596]}

# Evaluate on test data
10000/10000 [==============================] - 0s 48us/sample - loss: 0.1421 - sparse_categorical_accuracy: 0.9585
test loss, test acc: [0.14207501376457513, 0.9585]

# Generate predictions for 3 samples
predictions shape: (3, 10)
root@jetson-nano-qspi-sd:~# 
```
## Conclusions

## References

- [1] K. Hao, “The computing power needed to train AI is now rising seven times faster than ever before”, MIT Technology Review, Nov. 2019. [Online]. Available: https://www.technologyreview.com/s/614700/the- computing- power- needed- to-
train-ai-is-now-rising-seven-times-faster-than-ever-before.
- [2] Nvidia, nvidia-docker, [Online; accessed 15. Mar. 2020], Feb. 2020. [Online]. Available:https://github.com/NVIDIA/nvidia-docker.
- [3] L. Benedicic and M. Gila, “Accessing gpus from containers in hpc”, 2016. [Online]. Available: http://sc16.supercomputing.org/sc-archive/tech_poster/poster_files/post187s2-file3.pdf.
- [4] M. Madison, Container runtime for master, [Online; accessed 30. Mar. 2020], Mar. 2020. [Online]. Available:https://github.com/madisongh/meta-tegra/pull/266
