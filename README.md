# Arty Z7-10 Petalinux BSP Project

## Built for Petalinux 2017.2

#### Warning: You should only use this repo when it is checked out on a release tag

## BSP Features

This petalinux project targets the Vivado block diagram project found here: https://github.com/Digilent/Arty-Z7-10-base-linux.

The project includes the following features by default:

* Ethernet with Unique MAC address and DHCP support (see known issues)
* USB Host support
* UIO drivers for onboard switches, buttons and LEDs
* SSH server
* Build essentials package group for on-board compilation using gcc, etc. 
* HDMI output with kernel mode setting (KMS)
* HDMI input via UIO drivers

### Digilent Petalinux Apps (Coming Soon)

The next releases of this project will contain demo apps for:

* Lightweight library and demo for changing the HDMI output resolution without an X server
* HDMI input library for controlling the input video stream and a demo that passes it through
  to HDMI output with zero-copy
* UIO library and command line utility for controlling GPIO devices like onboard buttons, switches and LEDs
* UIO library and demo for controlling the PWM IP core attached to the RGB LEDs.

## Known Issues

* The console on the attached monitor will shutdown and not resume if left inactive. This can be prevented by running the following at 
  a terminal after boot.
```
echo -e '\033[9;0]' > /dev/tty1
```
* In order to meet timing, the input and output pipelines are clocked at a rate that will only support resolutions of around
  118 MHz (potentially slightly more based on horizontal blanking intervals). The output pipeline will automatically reject
  resolutions this high (this is accomplished with a device tree property), however the input pipeline cannot do the same. If a
  resolution with a pixel clock greater than 118 MHz is provided on the input pipeline, then it will overflow and likely stop
  working. 
* The MAC address is not currently being read out of the OTP region of Quad SPI flash. The MAC address should be manually
  set by reading it from the sticker on the Arty Z7 and then running petalinux-config. The option to set the MAC address is:
```
  -> Subsystem AUTO Hardware Settings -> Ethernet Settings -> Ethernet MAC Address
```
* MACHINE_NAME is currently still set to "template". Not sure the ramifications of changing this, but I don't think our boards
  our supported. For now just leave this as is until we have time to explore the effects of changing this value.
* We have experienced issues with petalinux when it is not installed to /opt/pkg/petalinux/. Digilent highly recommends installing petalinux
  to that location on your system.
* Netboot address and u-boot text address may need to be modified when using initramfs and rootfs is too large. The ramifications of this
  need to be explored and notes should be added to this guide. If this is causing a problem, then u-boot will likely crash or not successfully
  load the kernel. The workaround for now is to use SD rootfs.
* Ethernet PHY reset is not indicated correctly in the device tree. This means it will not be used by the linux and u-boot drivers. This does not
  seem to be causing any known functionality issues. 

## Quick-Start Guide

This guide will walk you through some basic steps to get you booted into Linux and rebuild the Petalinux project. After completing it, you should refer
to the Petalinux Reference Guide (UG1144) from Xilinx to learn how to do more useful things with the Petalinux toolset. Also, refer to the Known Issues 
section above for a list of problems you may encounter and work arounds.

This guide assumes you are using Ubuntu 16.04.3 LTS. Digilent highly recommends using Ubuntu 16.04.x LTS, as this is what we are most familiar with, and 
cannot guarantee that we will be able to replicate problems you encounter on other Linux distributions.

### Run the pre-built image from SD

1. Obtain a microSD card that has its first partition formatted as a FAT filesystem.
2. Copy _pre-built/linux/images/BOOT.BIN_ and _pre-built/linux/images/image.ub_ to the first partition of your SD card.
3. Eject the SD card from your computer and insert it into the Arty Z7
4. Attach a power source and select it with JP5 (note that using USB for power may not provide sufficient current)
5. If not already done to provide power, attach a microUSB cable between the computer and the Arty Z7
6. Open a terminal program (such as minicom) and connect to the Arty Z7 with 115200/8/N/1 settings (and no Hardware flow control). The Arty Z7 UART typically shows up as /dev/ttyUSB1
7. Optionally attach the Arty Z7 to a network using ethernet or an HDMI monitor.
8. Press the PORB button to restart the Arty Z7. You should see the boot process at the terminal and eventually a root prompt.

### Install the Petalinux tools

Digilent has put together this quick installation guide to make the petalinux installation process more convenient. Note it is only tested on Ubuntu 16.04.3 LTS. 

First install the needed dependencies by opening a terminal and running the following:

```
sudo -s
apt-get install tofrodos gawk xvfb git libncurses5-dev tftpd zlib1g-dev zlib1g-dev:i386  \
                libssl-dev flex bison chrpath socat autoconf libtool texinfo gcc-multilib \
                libsdl1.2-dev libglib2.0-dev screen pax 
reboot
```

Next, install and configure the tftp server (this can be skipped if you are not interested in booting via TFTP):

```
sudo -s
apt-get install tftpd-hpa
chmod a+w /var/lib/tftpboot/
reboot
```

Create the petalinux installation directory next:

```
sudo -s
mkdir -p /opt/pkg/petalinux
chown <your_user_name> /opt/pkg/
chgrp <your_user_name> /opt/pkg/
chgrp <your_user_name> /opt/pkg/petalinux/
chown <your_user_name> /opt/pkg/petalinux/
exit
```

Finally, download the petalinux installer from Xilinx and run the following (do not run as root):

```
cd ~/Downloads
./petalinux-v2017.2-final-installer.run /opt/pkg/petalinux
```

Follow the onscreen instructions to complete the installation.

### Source the petalinux tools

Whenever you want to run any petalinux commands, you will need to first start by opening a new terminal and "sourcing" the Petalinux environment settings:

```
source /opt/pkg/petalinux/settings.sh
```

### Generate project

If you have obtained the project source directly from github, then you should simply _cd_ into the Petalinux project directory. If you have downloaded the 
.bsp, then you must first run the following command to create a new project.

```
petalinux-create -t project -s <path to .bsp file>
```

This will create a new petalinux project in your current working directory, which you should then _cd_ into.

### Build the petalinux project

Run the following commands to build the petalinux project with the default options:

```
petalinux-config --oldconfig
petalinux-build
petalinux-package --boot --force --fsbl images/linux/zynq_fsbl.elf --fpga images/linux/Arty_Z7_10_wrapper.bit --u-boot
```

### Boot the newly built files from SD 

Follow the same steps as done with the pre-built files, except use the BOOT.BIN and image.ub files found in _images/linux_.

### Configure SD rootfs 

This project is initially configured to have the root file system (rootfs) existing in RAM. This configuration is referred to as "initramfs". A key 
aspect of this configuration is that changes made to the files (for example in your /home/root/ directory) will not persist after the board has been reset. 
This may or may not be desirable functionality.

Another side affect of initramfs is that if the root filesystem becomes too large (which is common if you add many features with "petalinux-config -c rootfs)
 then the system may experience poor performance (due to less available system memory). Also, if the uncompressed rootfs is larger than 128 MB, then booting
 with initramfs will fail unless you make modifications to u-boot (see note at the end of the "Managing Image Size" section of UG1144).

For those that want file modifications to persist through reboots, or that require a large rootfs, the petalinux system can be configured to instead use a 
filesystem that exists on the second partition of the microSD card. This will allow all 512 MiB of memory to be used as system memory, and for changes that 
are made to it to persist in non-volatile storage. To configure the system to use SD rootfs, write the generated root fs to the SD, and then boot the system, 
do the following:

Start by running petalinux-config and setting the following option to "SD":

```
 -> Image Packaging Configuration -> Root filesystem type
```

Next, open project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi in a text editor and locate the "bootargs" line. It should read as follows:

`
		bootargs = "console=ttyPS0,115200 earlyprintk uio_pdrv_genirq.of_id=generic-uio";
`

Replace that line with the following before saving and closing system-user.dtsi:

`
		bootargs = "console=ttyPS0,115200 earlyprintk uio_pdrv_genirq.of_id=generic-uio root=/dev/mmcblk0p2 rw rootwait";
`

#### Note: If you wish to change back to initramfs in the future, you will need to undo this change to the bootargs line.

Then run petalinux-build to build your system. After the build completes, your rootfs image will be at images/linux/rootfs.ext4.

Format an SD card with two partitions: The first should be at least 500 MB and be FAT formatted. The second needs to be at least 1.5 GB (3 GB is preferred) and 
formatted as ext4. The second partition will be overwritten, so don't put anything on it that you don't want to lose. If you are uncertain how to do this in 
Ubuntu, gparted is a well documented tool that can make the process easy.

Copy _images/linux/BOOT.BIN_ and _images/linux/image.ub_ to the first partition of your SD card.

Identify the /dev/ node for the second partition of your SD card using _lsblk_ at the command line. It will likely take the form of /dev/sdX2, where X is 
_a_,_b_,_c_,etc.. Then run the following command to copy the filesystem to the second partition:

#### Warning! If you use the wrong /dev/ node in the following command, you will overwrite your computer's file system. BE CAREFUL

```
sudo umount /dev/sdX2
sudo dd if=images/linux/rootfs.ext4 of=/dev/sdX2
sync
```

The following commands will also stretch the file system so that you can use the additional space of your SD card. Be sure to replace the
block device node as you did above, and also XG needs to be replaced with the size of the second partition on your SD card (so if your 
2nd partition is 3 GiB, you should use 3G):

```
sudo resize2fs /dev/sdX2 XG
sync
```

#### Note: It is possible to use a third party prebuilt rootfs (such as a Linaro Ubuntu image) instead of the petalinux generated rootfs. To do this, just copy the prebuilt image to the second partition instead of running the "dd" command above. Please direct questions on doing this to the Embedded linux section of the Digilent forum.

Eject the SD card from your computer, then do the following:

1. Insert the microSD into the Arty Z7
2. Attach a power source and select it with JP5 (note that using USB for power may not provide sufficient current)
3. If not already done to provide power, attach a microUSB cable between the computer and the Arty Z7
4. Open a terminal program (such as minicom) and connect to the Arty Z7 with 115200/8/N/1 settings (and no Hardware flow control). The Arty Z7 UART typically shows up as /dev/ttyUSB1
5. Optionally attach the Arty Z7 to a network using ethernet or an HDMI monitor.
6. Press the PORB button to restart the Arty Z7. You should see the boot process at the terminal and eventually a root prompt.

### Prepare for release

This section is only relevant for those who wish to upstream their work or version control their own project correctly on Github.
Note the project should be released configured as initramfs for consistency, unless there is very good reason to release it with SD rootfs.

#### Warning: image.ub must be less than 100 MB, or github will break and the image likely won't work

```
petalinux-package --prebuilt --clean --fpga images/linux/Arty_Z7_10_wrapper.bit -a images/linux/image.ub:images/image.ub 
petalinux-build -x distclean
petalinux-build -x mrproper
petalinux-package --bsp --force --output ../releases/Petalinux-Arty-Z7-10-20XX.X-X.bsp -p ./
```
Remove TMPDIR setting from project-spec/configs/config (this is done automatically for bsp project).
```
cd ..
git status # to double-check
git add .
git commit
git push
```
Finally, open a browser and go to github to push your .bsp as a release.


