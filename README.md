# NVIDIA container runtime for Wind River Linux
This demo is part of the WindRiver Labs project available at: http://labs.windriver.com

To enable NVIDIA containers, Docker needs to have the nvidia-containers-runtime which is a modified version of runc that adds a custom pre-start hook to all containers. The nvidia-containers-runtime communicates docker using the library libnvidia-container, which automatically configures GNU/Linux containers leveraging NVIDIA hardware. This library relies on kernel primitives and is designed to be agnostic of the container runtime. All the effort to port these libraries and tools to Yocto was submitted to the community and now is part of the meta-tegra layer which is maintained by Matt Madison.

Note: this setup is based on Linux for Tegra and not the original Yocto Linux Kernel


## Prerequisites
You are required to download NVIDIA property code from their website. To do so, you will need to create an NVIDIA Developer Network aacount.

Go into https://developer.nvidia.com/embedded/downloads , download the NVIDIA SDK Manager, install it and download all the files for the Jetson board you own.All the effort to port these libraries and tools to Yocto was submited to the community and now is part of the meta-tegra layer which is maintained by Matt Madison
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
git clone --branch WRLINUX_10_19_LTS https://windshare.windriver.com/remote.php/gitsmart/WRLinux-lts-19-Core/wrlinux-x
./wrlinux-x/setup.sh --all-layers --dl-layers --templates feature/docker --distro wrlinux-graphics
```

Note: wrlinux-graphics is not required but it can be used for some demos that require x11.

### Add meta-tegra layer
```bash
git clone --single-branch --branch wip-container-32.3.1 https://github.com/madisongh/meta-tegra.git layers/meta-tegra
```
**Tested with: https://github.com/madisongh/meta-tegra/commit/15c0a4979348d76737af7d5a184dadeb52b055c4**
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
bitbake wrlinux-image-glibc-std-sato
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
This command will create the file `wrlinux-image-glibc-std-sato.sdcard` that contains the SD card image required to boot.

Burn the Image to the SD Card:
```bash
sudo dd if=wrlinux-image-glibc-std-sato.sdcard of=/dev/***** bs=8k
```
**Warning: substitute the of= device to the one that points to your sdcard**
**Failure to do so can lead to unexpected erase of hard disks**


## Deploy the target

Boot up the board and find the ip address with the command `ifconfig`.

Then, ssh into the machine and run docker:

```bash
$ ssh root@<ip_address>
# docker run --runtime nvidia -it paroque28/l4t-tensorflow
```

Inside the container run:
```bash
python3 ./tensorflow_demo.py
```
