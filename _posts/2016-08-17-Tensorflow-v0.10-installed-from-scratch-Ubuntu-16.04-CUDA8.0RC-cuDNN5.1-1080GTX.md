---
layout: post
title: Tensorflow v0.10 installed from scratch on Ubuntu 16.04, CUDA 8.0RC+Patch, cuDNN v5.1 with a 1080GTX
---

###### Draft

While Tensorflow has a great documentation, you have quite a lot of details that are not obvious, especially the part about setting up Nvidia libraries. So here is a guide, explaining everything from scratch.

## 1. Installing Nvidia drivers

The first step is to get the latest Nvidia driver. While you can use `apt-get`, this causes a lot of issues with automatic updates. It is safer to do everything manually.

Go to [Nvidia's download website](http://www.nvidia.fr/Download/index.aspx) and download the latest version of the driver, here for Linux 64-bit. In my case, `NVIDIA-Linux-x86_64-367.35.run`.

As drivers for graphic devices are running at a low level, you must exit the GUI with `sudo service lightdm stop` and set the [RunLevel](https://fr.wikipedia.org/wiki/Run_level) to 3 with the program `init`.

Then, move to the directory where you downloaded the .run file and run it. You will be asked to confirm several things, the pre-install of something failure, no 32-bit libraries and more. Just continue to the end. Once it is done, reboot.

```
sudo service lightdm stop
sudo init 3
sudo sh NVIDIA-Linux-x86_64-367.35.run
sudo reboot
```

###### Login loop issue after updates

Due to the manual installation, it seems that when you do Ubuntu updates, they may install the `apt-get` version of the driver. This causes a failure when you start the computer and login, you will get a black screen and go back to the login screen.

The solution is to enter the terminal with `CTRL+ALT+F1` and reinstall the driver just like before. Note that you can get back to the GUI with `Alt+F7` when you are in the terminal.

## 2. Installing CUDA

### Install the Toolkit

It's now time for CUDA. Go to the [Nvidia CUDA website](https://developer.nvidia.com/cuda-release-candidate-download) and create an account if you don't already have one and log in (I think this is only required for RC versions of CUDA, which is the case currently for CUDA 8.0RC, an account is also required to download cdDNN).

Choose **Linux > x86_64 > Ubuntu > 16.04 > runfile (local)** and download the base installer and the patch. Ubuntu 16.04 uses GCC 5.4.0 as default C compiler, which caused an issue with CUDA 8.0RC, this is fixed with the patch.

The installer has 3 parts, a Nvidia driver, CUDA Toolkit and CUDA code samples. The Nvidia driver is usually outdated, that's why we installed it before, say no when asked if you want to install the driver (in Nvidia's install guide, they tell use to enter RunLevel 3, but this isn't necessary if we don't install the driver). Then, let everything as default, install the code samples to check your CUDA installation. To avoid an error about GCC 5.4.0, add `--override`. Then, once the installation is over, run the patch.

```
sudo sh cuda_8.0.27_linux.run --override
sudo sh cuda_8.0.27.1_linux.run
```

### Update paths in .bashrc

The next part is to update CUDA_HOME, PATH and LD_LIBRARY_PATH. Move to your home folder and update `.bashrc` then reload `.bashrc` with the command `source`. For people who are not Linux experts, `.bashrc` is a file with user parameters that is launched when you login, you must reload it or restart the session for the changes to be active.

```
cd /home/username/
gedit .bashrc
```

At the bottom of the file, add the following lines and save:

```
export CUDA_HOME=/usr/local/cuda-8.0
export PATH=/usr/local/cuda-8.0/bin${PATH:+:${PATH}}
export LD_LIBRARY_PATH=/usr/local/cuda-8.0/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}
```

You can then reload `.bashrc` and check that the paths have been properly modified.

```
source ~/.bashrc
echo $CUDA_HOME
echo $PATH
echo $LD_LIBRARY_PATH
```

### Check that CUDA is working

Then, you can check is CUDA is working by checking the version of `nvcc` the CUDA compiler and also by moving to the sample directory and compiling `bandwidthTest`.

```
nvcc --version
```

```
cd NVIDIA_CUDA-8.0_Samples/1_Utilities/bandwidthTest/
make
./bandwitdhTest
```

You should get an output that looks like this:

> [CUDA Bandwidth Test] - Starting...
> Running on...
> 
> Device 0: GeForce GTX 1080
> Quick Mode
> 
> Host to Device Bandwidth, 1 Device(s)
> PINNED Memory Transfers
>   Transfer Size (Bytes)	Bandwidth(MB/s)
>   33554432			12038.9
> 
> Device to Host Bandwidth, 1 Device(s)
> PINNED Memory Transfers
>   Transfer Size (Bytes)	Bandwidth(MB/s)
>   33554432			12832.1
> 
> Device to Device Bandwidth, 1 Device(s)
> PINNED Memory Transfers
>   Transfer Size (Bytes)	Bandwidth(MB/s)
>   33554432			231046.9
> 
>Result = PASS
> 
>NOTE: The CUDA Samples are not meant for performance measurements. Results may vary when GPU Boost is enabled.

You can now move to cuDNN!

## 3. Installing cuDNN

Go to [the Nvidia cuDNN website](https://developer.nvidia.com/cudnn), login and download **Download cuDNN v5.1 (August 10, 2016), for CUDA 8.0 RC > cuDNN v5.1 Library for Linux**. Unzip the .tgz file and copy the files to the cuda-8.0 folder. Some of the .so files are links to the real .so file, to preserve the links and avoid pure copy, use `cp -P`.

```
tar xvzf cudnn-8.0-linux-x64-v5.1.tgz
sudo cp cuda/include/cudnn.h /usr/local/cuda-8.0/include/
sudo cp -P cuda/lib64/* /usr/local/cuda-8.0/lib64/
```

That's it. As you see, it is quite easy to add or remove cuDNN and replace the it by another version of the library.

## 4. Installing Tensorflow
