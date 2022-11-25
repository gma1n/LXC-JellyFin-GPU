# Nvidia GPU Passthrough to LXC Container in Proxmox.
This is a step-by-step guide that will walk you through getting your GPU passed through from the host to a LXC container. My goal is to try and make your life easier so just about everything will be copy and paste to make your life a little easier (but I'll warn you when you cant).

### *System overview / Prerequisite*
- System running Proxmox
- Supported NVENC GPU - which can be found here: [Nvidia GPU Matrix](https://developer.nvidia.com/video-encode-and-decode-gpu-support-matrix-new)
  - I am using a GTX 970 but this will work for just about any supported nvidia card.
- In this scenario, I'll be using using ubuntu 22.04 CT Container (don't worry about downloading it now - we will grab it in proxmox later).
- Enable IOMMU support in your motherboards BIOS
- This will work for AMD and Intel CPU's but this guide is specifically for **Nvidia GPU's**.

## Before we take the plunge
We need to make sure that IOMMU is enabled - if it isn't the rest of this process **Will. Not. Work**. To check for this run
```
dmesg | grep IOMMU
```
It should return something similar to
```
root@pve:~# dmesg | grep IOMMU  
[    0.097544] AGP: Please enable the IOMMU option in the BIOS setup
[    0.589184] pci 0000:00:00.2: AMD-Vi: Found IOMMU cap 0x40
```
All we care about is seeing that "Found IOMMU" for the PCI group.

## Getting started
Before we can do anything.. the first thing we need to do is update **Everything**. To start this:
1. Select your node in the left pane
2. Click `Repositories` under Updates
3. Click `Add`
4. Add the No-subscription Repo:

![image](https://i.imgur.com/FY8uo5S.png)

> **Be sure to disable the `pve-enterprise` repo after adding the No-subscription repo.**

Once added, go into the updates menu and click `Refresh`. This will load any/or all system updates to the kernel. When it's done loading click on the `>_Upgrade`. This will open a new window and start the updates. Be paitent as this can take a few minutes.

Reboot the node after updates are completed.

## Checking Headers and Blocking Nouveau
Once the node has booted back up we want to double check the headers are up to date and that it was propperly installed. To do this:
```
uname -r
```
This should return (or something similar - systems will vary):
```
root@pve:~# uname -r
5.15.74-1-pve
```
To make sure we got what we need - lets run:
```
apt-cache search pve-header
```
This will return a big list of all the available headers. Remember the updates we did in the previous step? That updated us to the most recent kernel for our system. Sometimes, not *everything* gets downloaded and/or installed properly. 
The trick here is matching what was returned when doing `uname -r` and **NOT** going with the latest number you see. In this scenario, I want `pve-headers-5.15.74-1-pve` so I'm going to run:
```
apt install pve-headers-5.15.74-1-pve
```
![image](https://i.imgur.com/t5MndtW.png)

Once that's done it's now time to block the Nouveau driver..
 > Side note - The nouveau driver is the generic opensource graphics driver for linux systems. These two drivers can not coexist with each other. I promise we will be getting the official Nvidia drivers soon.

To block this:
```
nano /etc/modprobe.d/blacklist.conf
```
Once open paste in
```
blacklist nouveau
```
When done press `Ctrl + x` - press `y` - then `Enter` to close.

Then:
```
update-initramfs -u
```
This will take a minute to update. Once done - reboot the node.

## Installing Dependencies and Installing Nvidia drivers
Once the node has booted back up - we need some dependencies to build the drivers. Run:
```
apt install build-essential
```
When that's done.. Now it's time to get your **specific drivers**. Head on over to [Nvidia drivers](https://www.nvidia.com/Download/index.aspx).
For my GTX 970 I'm going to select:
```
Product Type: Geforce
Product Series: Geforce 900 Series
Product: Geforce GTX 970
Operating System: Linux 64-bit
Download Type: Production Branch
Language: English (US)
```
Then click `Search` then `Download`. Once on the final landing download page, right click the download button and `Copy Link Address`.
Should look like this:
![image](https://i.imgur.com/Wost8fv.png)

Go back to your node console and `wget <link to drivers>`. For me, it looks like this:
```
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/515.86.01/NVIDIA-Linux-x86_64-515.86.01.run
```
You can check it was downloaded by typing `ls` - the installer will be listed. 
```
root@pve:~# ls
NVIDIA-Linux-x86_64-515.86.01.run
```
Now that we have the driver downloaded we have to make the installer executable. To do this:
```
chmod +x <file> -should look like- chmod +x NVIDIA-Linux-x86_64-515.86.01.run
```
> ### Pro tip - Type out chmod +x then type out capital `N` then press `Tab`. It'll autofill the file name for you.
Next - put a `./` in front of your file to run it. Should Look like this:
```
./NVIDIA-Linux-x86_64-515.86.01.run
```
### Driver Install
After a few seconds, you should be prompted with a Nvidia install screen. The first screen to pop up should say:
```
WARNING: nvidia-installer was forced to guess the X library path.... blah blah blah.
```
Hit `Enter` for `OK`. Next message:
```
Install NVIDIA's 32-bit compatibility libraries?
```
Hit `Yes`. Next screen will be some loading bars. In a few seconds you'll see a message stating:
```
Would you like to run the nvidia-xconfig utility...
```
This is asking if you would like the GUI. We have no way of using it as we're going to monitor everything in terminal so hit `No`. Next message:
```
Installation of the NVIDIA Accelerated Graphics Driver is now complete..
```
Hit `Enter` to select `Ok` to close/finish the installer - This will bring you back to the terminal.



**IF ALL WENT ACCORDING TO PLAN**.. Let's test to see if everything is working properly (so far).
Type in:
```
nvidia-smi
```
![image](https://i.imgur.com/Za2Hhel.png)

Your node can now fully recognize and utilize your GPU!!! Go ahead and reboot the node.

## WAIT!! I got an error when trying to run the installer! Said something about modprobe, nouveau being blocked, or nouveau was still in use!
Thats okay, we will fix it with this:
```
nano /etc/modprobe.d/blacklist-nouveau.conf
```
Add in:
```
blacklist nouveau
options nouveau modeset=0
```
When done press `Ctrl + x` - press `y` - then `Enter` to close.

Regenerate the kernel initramfs:
```
update-initramfs -u
```
Afer thats done, reboot the node. Repeat the driver install.

## Config files 
After the reboot, make sure the drivers are still loaded by
```
nvidia-smi
```
Next, go to:
```
nano /etc/modules-load.d/modules.conf
```
Now add in:
```
nvidia
nvidia-modeset
nvidia_uvm
```
![image](https://i.imgur.com/AuHsExV.png)

Once saved we need to regenerate the kernel initramfs- 
```
update-initramfs -u
```
Next - we are going to create udev rules in the node.
```
nano /etc/udev/rules.d/70-nvidia.rules
```
Add these lines
```
KERNEL=="nvidia", RUN+="/bin/bash -c '/usr/bin/nvidia-smi -L && /bin/chmod 666 /dev/nvidia*'"
KERNEL=="nvidia_modeset", RUN+="/bin/bash -c '/usr/bin/nvidia-modprobe -c0 -m && /bin/chmod 666 /dev/nvidia-modeset*'"
KERNEL=="nvidia_uvm", RUN+="/bin/bash -c '/usr/bin/nvidia-modprobe -c0 -u && /bin/chmod 666 /dev/nvidia-uvm*'
```
![image](https://i.imgur.com/8UAEmJI.png)
> These udev rules are setting the permissions and calling for the module files to be loaded into the kernel since this wasn't done by the nvidia driver installer.

When done, reboot the node. Once booted back up, double check drivers are still running by `nvidia-smi`.

## Creating a LXC Container
Now that we have done ALL that, now its time to create the container.

![image](https://i.imgur.com/09nawli.png)
![image](https://i.imgur.com/MvbaKRg.png)

After downloading (output log will show "TASK OK" when done), go to the top right of your screen and `Create CT`.
![image](https://i.imgur.com/okjrgTi.png)

This will bring up the `Create: LXC Container`
- `General` - set your host name. Since I plan on using this container as a JellyFin server, my hostname is `JellyFin`. Set a root password, uncheck 'Unpriviledged Container'.
- `Template` - select the Ubuntu-22.04 CT we just downloaded.
- `Disks` - Disk size set to 8gb.
- `CPU` - set to 2 cores.
- `Memory` - set to 2048mb (2gb) and 512mb swap.
- `Network` - set IPv4 to DHCP, set IPv6 to static but left it blank (should say none in the box)
- `DNS` - Left blank / "use host settings"

#### Before you hit finish **make sure the `start after created` box is Unchecked!!**
![image](https://i.imgur.com/pYUWq7x.png)

Hit finish!

## Editing the container for passthrough
When the container is done being created, we need to go back to your main node console. We still have to make some changes to allow the the GPU to be passed through. This is where your input is going to be needed.
## Do **NOT** copy my settings exactly! These numbers are going to be different for your system. I will show what my config looks like but be sure to change these values.

To start:
```
ls -l /dev/nv*
```
![image](https://i.imgur.com/9iPSdRT.png)

The highlighted numbers are the system specific hardware ID's that correspond with the files we are going to be calling for in the next step. Take note of these numbers.

```
nano /etc/pve/lxc/<container id>.conf
```
![image](https://i.imgur.com/wNz2d5a.png)

Once in the containers config file, we want add these lines:
```
#cgroup access
lxc.cgroup2.devices.allow: c 195:0 rw
lxc.cgroup2.devices.allow: c 195:255 rw
lxc.cgroup2.devices.allow: c 195:254 rw
lxc.cgroup2.devices.allow: c 510:0 rw
lxc.cgroup2.devices.allow: c 510:1 rw
lxc.cgroup2.devices.allow: c 10:144 rw

#device files   
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
lxc.mount.entry: /dev/nvram dev/nvram none bind,optional,create=file
```
Remember those numbers I highlighted with `ls -l /dev/nv*`? This is where they come into play.
![image](https://i.imgur.com/josMC55.png)

When done press `Ctrl + x` - press `y` - then `Enter` to close.


## Installing Drivers nvidia drivers in the container
Since we are done editing config files, we can start the container!
Login as `root` and the password you set. Once logged in - we need to update:
```
apt update && apt upgrade -y
```
![image](https://i.imgur.com/hVYI6l6.png)

We need to grab the drivers again:
```
wget <nvidia driver link>
```
After it downloaded:
```
chmod +x <driver file>
```
Before we run the installer again we need to add an extra flag to the installer.:
```
./<driver> --no-kernel-module
```
so for me it should read as:
```
./NVIDIA-Linux-x86_64-515.86.01.run --no-kernel-module
```
The reason for this is because this container is using your main nodes kernel. That's why we did all the legwork before to get to this point.

Once the driver install opens the only new message will be:
```
WARNING: You specified the '--no-kernel-module' command line option...
```
Just hit `OK` and proceed with the driver install like we did previously. When the installer is done, reboot the container.
After reboot log back in as `root`. Run the `nvidia-smi` command.

![image](https://i.imgur.com/BFVo8nl.png)

Congratulations! You have successfully just passed your GPU to an LXC Container!!
From here you can do whatever you like. Since I am going to be using this as a media server you can install something like Plex or JellyFin.
- Plex - [Installing Plex on Ubuntu 22.04](https://linuxhint.com/install_plex_ubuntu-2/)
- JellyFin - [Installing JellyFin on Ubuntu](https://jellyfin.org/docs/general/administration/installing#ubuntu-repository)

# Thank You!

