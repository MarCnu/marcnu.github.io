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

The next part is to update CUDA_HOME, PATH and LD_LIBRARY_PATH. Move to your home folder and update `.bashrc` then reload `.bashrc` with the command `source`. For those who are not Linux experts, `.bashrc` is a file with user parameters that is launched when you login, you must reload it or restart the session for the changes to be active.

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

### (Optional) Check that CUDA is working

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

Go to [the Nvidia cuDNN website](https://developer.nvidia.com/cudnn), login and download **Download cuDNN v5.1 (August 10, 2016), for CUDA 8.0 RC > cuDNN v5.1 Library for Linux**. Unzip the .tgz file and copy the files to the cuda-8.0 folder. Note that some of the .so files are links to the "real" .so file, by copying it, we duplicate the file, that way, when building Tensorflow from source, any cuDNN version will give libcudnn.so.5.1.5.

```
tar xvzf cudnn-8.0-linux-x64-v5.1.tgz
cd cuda
sudo cp include/cudnn.h /usr/local/cuda-8.0/include/
sudo cp lib64/* /usr/local/cuda-8.0/lib64/
```

That's it. As you see, it is quite easy to add or remove cuDNN and replace the it by another version of the library.

## 4. Installing Tensorflow

It's now time to install Tensorflow from source as the official binaries are only for CUDA 7.5. We will install it for Python2.7.

### Install dependencies
First, install some general dependancies.

```
sudo apt-get install python-pip python-dev python-numpy swig python-dev python-wheel
```

### Install Bazel

Then install [Bazel](http://bazel.io/docs/install.html), a build tool from Google.

First, you need to download and install JDK 8.

```
sudo add-apt-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install oracle-java8-installer
```

It's now time to get Bazel.

```
echo "deb [arch=amd64] http://storage.googleapis.com/bazel-apt stable jdk1.8" | sudo tee /etc/apt/sources.list.d/bazel.list
curl https://storage.googleapis.com/bazel-apt/doc/apt-key.pub.gpg | sudo apt-key add -
sudo apt-get update && sudo apt-get install bazel
sudo apt-get upgrade bazel
```

### Install Tensorflow itself

First, you must get the code from Github. You can either take the most recent master branch (lots of new commits) or the latest release branch (should be more stable, but still updated every few days). Here, we get branch r0.10.

```
git clone -b r0.10 https://github.com/tensorflow/tensorflow
cd tensorflow
```

#### Important: fix CROSSTOOL file

Edit the text file **tensorflow/third_party/gpus/crosstool/CROSSTOOL** and add `cxx_builtin_include_directory: "/usr/local/cuda-8.0"` as below.

```
  cxx_builtin_include_directory: "/usr/lib/gcc/"
  cxx_builtin_include_directory: "/usr/local/include"
  cxx_builtin_include_directory: "/usr/include"
  cxx_builtin_include_directory: "/usr/local/cuda-8.0"
  tool_path { name: "gcov" path: "/usr/bin/gcov" }
```

If you don't do this, you will get an error that looks like this:

> ERROR: /home/marceau/Documents/retest/tensorflow/tensorflow/contrib/rnn/BUILD:46:1: undeclared inclusion(s) in rule '//tensorflow/contrib/rnn:python/ops/_lstm_ops_gpu':  
> this rule is missing dependency declarations for the following files included by 'tensorflow/contrib/rnn/kernels/lstm_ops_gpu.cu.cc':  
>   '/usr/local/cuda-8.0/include/cuda_runtime.h'  
>   '/usr/local/cuda-8.0/include/host_config.h'  
>   ...  
>   '/usr/local/cuda-8.0/include/curand_discrete2.h'.  
> nvcc warning : option '--relaxed-constexpr' has been deprecated and replaced by option '--expt-relaxed-constexpr'.  
> nvcc warning : option '--relaxed-constexpr' has been deprecated and replaced by option '--expt-relaxed-constexpr'.  
> Target //tensorflow/tools/pip_package:build_pip_package failed to build  
> Use --verbose_failures to see the command lines of failed build steps.  
> INFO: Elapsed time: 203.657s, Critical Path: 162.10s


You can now run the configure script. If you have only cuda 8.0, then leaving everything as default should be fine. I just provided the compute capability of my GPU, in my case 6.1.

```
./configure
```

> Please specify the location of python. [Default is /usr/bin/python]:   
> **Do you wish to build TensorFlow with Google Cloud Platform support? [y/N] N**  
> No Google Cloud Platform support will be enabled for TensorFlow  
> Found possible Python library paths:  
>   /usr/local/lib/python2.7/dist-packages  
>   /usr/lib/python2.7/dist-packages  
> Please input the desired Python library path to use.  Default is [/usr/local/lib/python2.7/dist-packages]  
>   
> /usr/local/lib/python2.7/dist-packages  
> **Do you wish to build TensorFlow with GPU support? [y/N] y**  
> GPU support will be enabled for TensorFlow  
> Please specify which gcc should be used by nvcc as the host compiler. [Default is /usr/bin/gcc]:   
> Please specify the Cuda SDK version you want to use, e.g. 7.0. [Leave empty to use system default]:   
> Please specify the location where CUDA  toolkit is installed. Refer to README.md for more details. [Default is /usr/local/cuda]:   
> Please specify the Cudnn version you want to use. [Leave empty to use system default]:   
> Please specify the location where cuDNN  library is installed. Refer to README.md for more details. [Default is /usr/local/cuda]:   
> libcudnn.so resolves to libcudnn.5  
> Please specify a list of comma-separated Cuda compute capabilities you want to build with.  
> You can find the compute capability of your device at: https://developer.nvidia.com/cuda-gpus.  
> Please note that each additional compute capability significantly increases your build time and binary size.  
> **[Default is: "3.5,5.2"]: 6.1**  
> Setting up Cuda include  
> Setting up Cuda lib64  
> Setting up Cuda bin  
> Setting up Cuda nvvm  
> Setting up CUPTI include  
> Setting up CUPTI lib64  
> Configuration finished  

You can then run Bazel. The build will take quite a lot of time, 900s on my PC. Then, create the pip package and install it with pip. The name of the pip package may be different depending of Tensorflow's version.

```
bazel build -c opt --config=cuda //tensorflow/tools/pip_package:build_pip_package
bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg
pip install /tmp/tensorflow_pkg/tensorflow-0.10.0-py2-none-any.whl
```

That's it, Tensorflow is installed!

You can create a test.py file with the following code and run it to check that everything is working and that the GPU is recognised.

```python
import tensorflow as tf

hello = tf.constant('Hello, TensorFlow!')
sess = tf.Session()
print(sess.run(hello))
# Hello, TensorFlow!
a = tf.constant(10)
b = tf.constant(32)
print(sess.run(a + b))
# 42
```

```
python test.py
```

>I tensorflow/stream_executor/dso_loader.cc:108] successfully opened CUDA library libcublas.so.8.0 locally  
>I tensorflow/stream_executor/dso_loader.cc:108] successfully opened CUDA library libcudnn.so.5 locally  
>I tensorflow/stream_executor/dso_loader.cc:108] successfully opened CUDA library libcufft.so.8.0 locally  
>I tensorflow/stream_executor/dso_loader.cc:108] successfully opened CUDA library libcuda.so.1 locally  
>I tensorflow/stream_executor/dso_loader.cc:108] successfully opened CUDA library libcurand.so.8.0 locally  
>I tensorflow/stream_executor/cuda/cuda_gpu_executor.cc:925] successful NUMA node read from SysFS had negative value (-1), but there must be at least one NUMA node, so returning NUMA node zero  
>I tensorflow/core/common_runtime/gpu/gpu_init.cc:118] Found device 0 with properties:   
>name: GeForce GTX 1080  
>major: 6 minor: 1 memoryClockRate (GHz) 1.797  
>pciBusID 0000:01:00.0  
>Total memory: 7.92GiB  
>Free memory: 7.52GiB  
>I tensorflow/core/common_runtime/gpu/gpu_init.cc:138] DMA: 0   
>I tensorflow/core/common_runtime/gpu/gpu_init.cc:148] 0:   Y   
>I tensorflow/core/common_runtime/gpu/gpu_device.cc:870] Creating TensorFlow device (/gpu:0) -> (device: 0, name: GeForce GTX 1080, pci bus id: 0000:01:00.0)  
>Hello, TensorFlow!  
>42

You can now start having fun.
