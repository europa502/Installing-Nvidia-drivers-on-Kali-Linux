# Installing-Nvidia-drivers-on-Kali-Linux


After spending few days on how-tos and debugging the black screen issue on boot after insalling the nvidia drivers, I was finally able to find a solution to all my problems.
The main reason I'm writing this post is to let you know that the tutorial found on [Kali's official website](https://docs.kali.org/general-use/install-nvidia-drivers-on-kali-linux) is broken as of date 11 April 2018.
According to he mentioned in their website you might be able to successfully install the packages - ocl-icd-libopencl1, nvidia-driver, nvidia-cuda-toolkit
but you might encounter issues during the reboot. You might get a black screen and you won't be able to login via the GUI.

## So lets get started-
First of all let me tell you the specifications of my system-

CPU   - Intel® Core™ i5-8250U CPU @ 1.60GHz × 8 

GPU #1- Intel® UHD Graphics 620 

GPU #2- Nvidia GeForce MX150 

```shell
root@europa:~# uname -a
Linux europa 4.14.0-kali3-amd64 #1 SMP Debian 4.14.17-1kali1 (2018-02-16) x86_64 GNU/Linux
```

```shell
root@europa:~# cat /etc/*release*
DISTRIB_ID=Kali
DISTRIB_RELEASE=kali-rolling
DISTRIB_CODENAME=kali-rolling
DISTRIB_DESCRIPTION="Kali GNU/Linux Rolling"
PRETTY_NAME="Kali GNU/Linux Rolling"
NAME="Kali GNU/Linux"
ID=kali
VERSION="2018.1"
VERSION_ID="2018.1"
ID_LIKE=debian
ANSI_COLOR="1;31"
HOME_URL="http://www.kali.org/"
SUPPORT_URL="http://forums.kali.org/"
BUG_REPORT_URL="http://bugs.kali.org/"
```

Before we begin, a couple of notes:

***USE AT YOUR OWN RISK***

**This tutorial is for official NVIDIA Driver**

**Tutorial found on official Kali website is BROKEN! It never works for optimus/hybrid Graphics enabled laptop**

**Step 1:** Verify you have hybrid graphics

```shell
root@europa:~# lspci | grep -E "VGA|3D"
00:02.0 VGA compatible controller: Intel Corporation UHD Graphics 620 (rev 07)
01:00.0 3D controller: NVIDIA Corporation GP108M [GeForce MX150] (rev a1)
```

**Step 2:** Disable nouveau

```shell
echo -e "blacklist nouveau\noptions nouveau modeset=0\nalias nouveau off" > /etc/modprobe.d/blacklist-nouveau.conf
update-initramfs -u && reboot
```

**Step 3:** System will reboot and nouveau should be disabled. Verify if nouveau is disabled:
 ```shell
lsmod |grep -i nouveau
```
If shows nothing,means nouveau successfully disabled.

**Step 4:** Install nvidia driver from kali repo:
```shell
apt-get install  nvidia-driver nvidia-xconfig
```
You can also download latest .run file from [Nvidia's website](http://www.nvidia.com/Download/index.aspx). Execute and procceed with installation. Whether its from Kali's repo or Nvidia's website, procedure is same.
Code to install the .run file:
```shell 
sudo sh ./Nvidia-driver-filename.run
```
**Step 5:** Now we have to find bus id of our nvidia card:

```shell 
nvidia-xconfig --query-gpu-info | grep 'BusID : ' | cut -d ' ' -f6
```
it should show something like this:
```shell
PCI:1:0:0
```
This is our Bus ID.

**Step 6:** Now we generate /etc/X11/xorg.conf file with this bus ID according to [Nvidia's guide](http://us.download.nvidia.com/XFree86/Linux-x86/375.39/README/randr14.html):

```
Section "ServerLayout"
    Identifier "layout"
    Screen 0 "nvidia"
    Inactive "intel"
EndSection

Section "Device"
    Identifier "nvidia"
    Driver "nvidia"
    BusID "**PCI:1:0:0**"
EndSection

Section "Screen"
    Identifier "nvidia"
    Device "nvidia"
    Option "AllowEmptyInitialConfiguration"
EndSection

Section "Device"
    Identifier "intel"
    Driver "modesetting"
EndSection

Section "Screen"
    Identifier "intel"
    Device "intel"
EndSection
```
Replace the string inside ** ** with your Bus ID and save it to /etc/X11/xorg.conf

**Step 7:** Now we have to create some scripts according to our [display manager](https://wiki.archlinux.org/index.php/NVIDIA_Optimus#Display_Managers).Since im using default Kali linux which is GDM,i created two files:
/usr/share/gdm/greeter/autostart/optimus.desktop
/etc/xdg/autostart/optimus.desktop
with the following content:

```
[Desktop Entry]
Type=Application
Name=Optimus
Exec=sh -c "xrandr --setprovideroutputsource modesetting NVIDIA-0; xrandr --auto"
NoDisplay=true
X-GNOME-Autostart-Phase=DisplayServer
```
**Step 8:** Now reboot and you should be using Nvidia Driver. Verify if everything is ok:
install mesa-utils if not previously installed.
```bash
apt-get install mesa-utils
```
root@europa:~# glxinfo | grep -i "direct rendering"
direct rendering: Yes

**Step 9:** Now you can install the cuda toolkits and drivers 
```bash 
apt install -y ocl-icd-libopencl1 nvidia-driver nvidia-cuda-toolkit
```
**Step 10:** Now that our system should be ready to go, we need to verify the drivers have been loaded correctly. We can quickly verify this by running the [nvidia-smi](https://developer.nvidia.com/nvidia-system-management-interface) tool.
```
root@europa:~# nvidia-smi
Wed Apr 11 11:08:55 2018       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 390.42                 Driver Version: 390.42                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce MX150       Off  | 00000000:01:00.0 Off |                  N/A |
| N/A   60C    P0    N/A /  N/A |    368MiB /  2002MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    0       763      G   /usr/lib/xorg/Xorg                            20MiB |
|    0       793      G   /usr/bin/gnome-shell                          19MiB |
|    0      1108      G   /usr/lib/xorg/Xorg                            82MiB |
|    0      1191      G   /usr/bin/gnome-shell                         242MiB |
|    0      2132      G   gnome-control-center                           1MiB |
+-----------------------------------------------------------------------------+
```

***FIXING SCREEN TEARING ISSUE:***
After you successfully boot up with Nvidia Driver, you most probably would be experiencing screen tearing issue eg: glitches while playing videos in VLC, Youtube videos on Chrome/Firefox etc. Luckily, we can fix this by enabling PRIME Sync.

**1.** Verify if PRIME is disabled
```bash
xrandr --verbose|grep PRIME
```
it should output something like this:
```bash
PRIME Synchronization: 0
PRIME Synchronization: 1
```
First one is our connected display.So PRIME sync is disabled.

**2.** Edit **/etc/default/grub** and append **nvidia-drm.modeset=1** in **GRUB_CMDLINE_LINUX_DEFAULT** after quiet.Like the following:

```bash
...
GRUB_CMDLINE_LINUX_DEFAULT="quiet nvidia-drm.modeset=1"
...
```

**3.** Save the changes and update grub using the command:
```bash 
update-grub
```

**4.** Reboot your system.

**5.** Verify if PRIME is enabled:
```bash
xrandr --verbose|grep PRIME
```
Now it should output:
```bash
PRIME Synchronization: 1
PRIME Synchronization: 1
```

If it still shows 0 for you,then there is probably something wrong with your system config/kernel. Since this is still an experimental feature from Nvidia,you are out of luck.

***IF YOU STUCK IN BOOT SCREEN***

Revert what we have done so far:

Press CTRL+ALT+F2 or CTRL+ALT+F3, login with your password.
```bash
apt-get remove --purge nvidia-*
rm -rf /etc/X11/xorg.conf
```
Remove those display manager files we created earlier (for GDM):
```bash
rm -rf /usr/share/gdm/greeter/autostart/optimus.desktop
rm -rf /etc/xdg/autostart/optimus.desktop
```
Now reboot. You should be able get back to your old system. 




If any issues exist please post it in [Kali's form](https://forums.kali.org/showthread.php?35748-TUTORIAL-Installing-official-NVIDIA-driver-in-Optimus-laptop).

**My sincere thanks to [Tiger11](https://forums.kali.org/member.php?53670-TiGER511)** who did all the hard-work.

**I'll soon update this repo on how to switch between your Intel Graphics and Nvidia Graphic card.**
