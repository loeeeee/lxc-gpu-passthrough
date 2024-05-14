# LXC with nVidia GPU Passthrough

A guide for [nVidia](https://www.nvidia.com/) GPU passthrough on a [Ubuntu](https://ubuntu.com/) LXC ([Linux Containers](https://linuxcontainers.org/)) with a [Proxmox VE](https://www.proxmox.com/en/proxmox-virtual-environment/overview) 8 host.

This guide should be widely applicable to other LXC guest system that is supported by nVidia driver, because we only need to do minimal modification inside the guest system.

## Host setup

I am using `Proxmox VE 8` as the LXC host. It is based on `Debian` with some modification, and I have a NVIDIA GPU installed in the server.

## Prepare the LXC

First, install a Ubuntu 22.04 LTS LXC. No special configuration is needed here, besides making sure that enough disk space is given to the container so that the huge driver (and other nVidia components) will not overflow the entire disk.

## Actual installation

In this part of the guide, I will cover how to pass through nVidia GPU to the LXC container.

If we oversimplify things a little bit in the world of Linux, everything is a file in the system. Thus, the principle of this part is quite similar to mounting host file into the guest file system. We mount the nVidia "device file" into the LXC and install the driver.

### Install GPU driver on the host

First, we need to get the nVidia "device file" in the host. By default, Linux use [nouveau](https://nouveau.freedesktop.org/) driver for nVidia. It is a great project, but it does not support CUDA program right now. (Maybe [NVK](https://docs.mesa3d.org/drivers/nvk.html) driver will in one day, hopefully.) Consequently, the "device files" that are available to us are from nouveau driver, and are not what we want. And we need to install nVidia proprietary driver (Sad face).

Because we eventually will have two nVidia driver running in both host and container, we do not want the package manager of the host or the guest OS in the container update the driver by themselves or automatically. Thus, we are going to use the universal `.run` file from nVidia download center. Even though it is also possible to use package pinning in Ubuntu or similar functionality in other system, `.run` driver installer also provides a wider selection of driver, and is more portable.

You can copy the link by right click the download.

Inside the host machine,

```bash
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/550.78/NVIDIA-Linux-x86_64-550.78.run
chmod +x NVIDIA-Linux-x86_64-550.78.run
./NVIDIA-Linux-x86_64-550.78.run
```

Note:

- If your download failed and you start download again, the file will be named og name.1 and so on.
- Please check compatible driver version with your installed GPU. The link above is only an example.
- This method requires a driver reinstallation for every kernel update.

After the installation, proceed with a reboot of the host.

After the reboot, when running `nvidia-smi` there should be a lovely TUI output.

### Get permission of "device files" on the host

Now, let's set up the permission and mount the GPU.

First, let's look at the permission of the usage of GPU in the host machine.

```bash
ls -l /dev/nvidia*
ls -l /dev/dri/
```

The output will be something like this.

```bash
crw-rw-rw- 1 root root 195,   0 May  3 22:34 /dev/nvidia0
crw-rw-rw- 1 root root 195, 255 May  3 22:34 /dev/nvidiactl
crw-rw-rw- 1 root root 195, 254 May  3 22:34 /dev/nvidia-modeset
crw-rw-rw- 1 root root 508,   0 May  3 22:34 /dev/nvidia-uvm
crw-rw-rw- 1 root root 508,   1 May  3 22:34 /dev/nvidia-uvm-tools

/dev/nvidia-caps:
total 0
cr-------- 1 root root 511, 1 May  3 22:34 nvidia-cap1
cr--r--r-- 1 root root 511, 2 May  3 22:34 nvidia-cap2

total 0
drwxr-xr-x 2 root root      60 May  3 22:34 by-path
crw-rw---- 1 root video 226, 0 May  1 23:34 card0
```


Take a note of numbers, `60, 195, 226, 255, 254, 508, 511`. Note that these number may be different on different system, even different between kernel updates.

### Mount the "device files" in the LXC

Open the `/etc/pve/<lxc-id>.conf`, add the following lines, change the numbers accordingly.

```config
# ... Existing config
lxc.cgroup2.devices.allow: c 60:* rwm
lxc.cgroup2.devices.allow: c 195:* rwm
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.cgroup2.devices.allow: c 254:* rwm
lxc.cgroup2.devices.allow: c 255:* rwm
lxc.cgroup2.devices.allow: c 508:* rwm
lxc.cgroup2.devices.allow: c 511:* rwm
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
```

After saving the config, boot the LXC container.

### Install driver in the LXC guest OS

Inside the LXC container, install the nVidia driver but with a catch.

```bash
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/550.78/NVIDIA-Linux-x86_64-550.78.run
chmod +x NVIDIA-Linux-x86_64-550.78.run
./NVIDIA-Linux-x86_64-550.78.run --no-kernel-modules
```

We install the driver without kernel modules because LXC containers shares kernel with the host machine. Because we already install the driver on the host and its kernel, and we share the kernel, we do not need the kernel modules.

Note:

- One must use the **same version** of nVidia driver.
- **NO KERNEL MODULES**

After all these, we should be able to run `nvidia-smi` inside the LXC without error.

Zu easy, innit?
