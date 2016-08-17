---
layout: post
title: Tensorflow v0.10 installed from sratch on Ubuntu 16.04, CUDA8.0RC+Patch, cuDNN5.1 with a 1080GTX
---

###### Draft

While Tensorflow has a great documentation, you have quite a lot of details that are not obvious, especially the part about setting up Nvidia libraries. So here is a guide, explaining everything from scratch.

## 1. Installing Nvidia drivers

The first step is to get the latest Nvidia drivers. While you can use `apt-get`, this causes a lot of issues with automatic updates. It is safer to do everything manually.

Go to [Nvidia's download website](http://www.nvidia.fr/Download/index.aspx) and download the latest version of the driver, here for Linux 64-bit. In my case, `NVIDIA-Linux-x86_64-367.35.run`.

```
sudo service lightdm stop
sudo init 3
sudo sh NVIDIA-Linux-x86_64-367.35.run
sudo reboot
```
