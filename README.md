## What this is.

This is a simple readme of how I was able to configure the nvidia drivers so a nvidia gpu is suspended except when it is needed by an application, such as cuda.

I only tested this on an Asus Zephyrus GU502gv using ubuntu 19.10 using the nvidia drivers 435.21

This assumes that the nvidia drivers are already installed. (check this by running the command `nvidia-smi`.
If you have an output in the terminal then you are good to procede.


### Steps needed

1. Configure nvidia-prime to on-demand mode
1. Configure Xorg to use the intel integrated graphics
1. Configure power management from the nvidia drivers


### Configuring nvidia-prime

First you need to set the gpu to be on-demand.

Run these commands

```
$ sudo prime-select on-demand
$ sudo reboot
```

### Configure Xorg to use the intel integrated graphics

First check to see if your computer is using the nvidia graphics or not by running
```
$ nvidia-smi
```

you should get an output similar to below

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 435.21       Driver Version: 435.21       CUDA Version: 10.1     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce RTX 2060    Off  | 00000000:01:00.0 Off |                  N/A |
| N/A   37C    P0    N/A /  N/A |      0MiB /  5934MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

under Processes it should say `No running processes found`.

If you do have xorg running (like I did) then you need to configure xorg explicitely

edit the file `/etc/X11/xorg.conf` to have the following contents

```
Section "ServerLayout"
  Identifier "layout"
  Screen 0 "iGPU"
  Option "AllowNVIDIAGPUScreens"
EndSection

Section "Device"
  Identifier "iGPU"
  Driver "modesetting"
EndSection

Section "Screen"
  Identifier "iGPU"
  Device "iGPU"
EndSection

#Section "Device"
#  Identifier "dGPU"
#  Driver "nvidia"
#EndSection
```
> source: http://download.nvidia.com/XFree86/Linux-x86_64/440.31/README/primerenderoffload.html

Then you will need to restart your computer. or restart gdm or lightdm `sudo systemctl restart gdm`

Make sure you know what you are doing when you edit xorg.conf files. It can mess with your computer. So be sure to make a backup of your config files before changing anything. I had to secure boot a few times and revert configuration files back to the previous state before I found the configuration nvidia suggests.

### Setting up nvidia Dynamic Power Management

nvidia power management lets the driver suspend and wake the GPU as needed, but there are some hardware requirements

##### Required Hardware

1. This feature is supported only on notebooks.
1. This feature requires system hardware as well as ACPI support (ACPI "_PR0" and "_PR3" methods are needed to control PCIe power). The necessary hardware and ACPI support was first added in Intel Coffeelake chipset series. Hence, this feature is supported from Intel Coffeelake chipset series.
1. This feature requires a Turing or newer GPU.
1. This feature is supported with Linux kernel versions 4.18 and newer. With older kernel versions, it may not work as intended.
1. This feature is supported when Linux kernel defines CONFIG_PM (CONFIG_PM=y). Typically, if the system supports S3 (suspend-to-RAM), then CONFIG_PM would be defined.
> Source: http://download.nvidia.com/XFree86/Linux-x86_64/440.31/README/dynamicpowermanagement.html


If your computer fits the hardware requirements (they may change in the future, so check the docs if you are unsure)

you can edit the driver to change power management.

From the nvidia docs
```
Option "NVreg_DynamicPowerManagement=0x00"
	With this setting, the NVIDIA driver will only use the GPU's built-in power management so it
	always is powered on and functional. This is the default option, since this feature is a new
	and highly experimental feature. Actual power usage will vary with the GPU's workload.

Option "NVreg_DynamicPowerManagement=0x01"
	With this setting, the NVIDIA GPU driver will allow the GPU to go into its lowest power state
	when no applications are running that use the nvidia driver stack. Whenever an application
	requiring NVIDIA GPU access is started, the GPU is put into an active state. When the
	application exits, the GPU is put into a low power state.

Option "NVreg_DynamicPowerManagement=0x02"
	With this setting, the NVIDIA GPU driver will allow the GPU to go into its lowest power state
	when no applications are running that use the nvidia driver stack. Whenever an application
	requiring NVIDIA GPU access is started, the GPU is put into an active state. When the
	application exits, the GPU is put into a low power state.

	Additionally, the NVIDIA driver will actively monitor GPU usage while applications using the
	GPU are running. When the applications have not used the GPU for a short period, the driver
	will allow the GPU to be powered down. As soon as the application starts using the GPU, the
	GPU is reactivated.

	It is important to note that the NVIDIA GPU will remain in an active state if it is driving a
	display. In this case, the NVIDIA GPU will go to a low power state only when the X
	configuration option HardDPMS is enabled and the display is turned off by some means - either
	automatically due to an OS setting or manually using commands like xset.

	Similarly, the NVIDIA GPU will remain in an active state if a CUDA application is running.
```

To enable Dynamic Power Management edit the file `/etc/modprobe.d/nvidia.conf`
and add the setting you want. 
My configuration is
```
modprobe nvidia "NVreg_DynamicPowerManagement=0x02"
```
