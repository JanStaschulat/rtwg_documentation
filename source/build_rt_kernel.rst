Build RT kernel
=================

The goal is to create a Ubuntu based system with RT Linux kernel to test
real-time ROS2 stack. Two boards are proposed \* Raspberry Pi 4 B+ board
(ARMv8) \* Intel UP2 board (x86_64) # Intel UP2 board RT kernel build
See
https://index.ros.org/doc/ros2/Tutorials/Building-Realtime-rt_preempt-kernel-for-ROS-2/
# Raspberry PI 4 RT kernel build Ubuntu 20.04 x86_64 docker container is
used to cross-compile a new kernel. There is a Dockerfile which can be
used for that purpose. If you want to build it using gitpod you need to
run
https://gitpod.io/#https://github.com/ros-realtime/rt-kernel-docker-builder.
It will spawn a docker container automatically for you. ## build and run
docker container For the local build:

.. code:: bash

   $ git clone https://github.com/ros-realtime/rt-kernel-docker-builder
   $ cd rt-kernel-docker-builder
   $ docker build -t rtwg-image .
   $ docker run -t -i rtwg-image bash

setup a build environment
-------------------------

Container comes with cross-compilation tools installed, and a
ready-to-build RT kernel: \* ARMv8 cross-compilation tools \* linux
source build dependencies \* linux source buildinfo, i.e. from where
config is copied \* Ubuntu RPI4 linux source installed under
~/linux_build \* RT kernel patch downloaded and applied - the nearest to
the recent RPI4 Ubuntu kernel ## Kernel configuration Additionally RT
kernel configured as

.. code:: bash

   $ ./scripts/config -d CONFIG_PREEMPT \
   $ ./scripts/config -e CONFIG_PREEMPT_RT \
   $ ./scripts/config -d CONFIG_NO_HZ_IDLE \
   $ ./scripts/config -e CONFIG_NO_HZ_FULL \
   $ ./scripts/config -d CONFIG_HZ_250 \
   $ ./scripts/config -e CONFIG_HZ_1000 \
   $ ./scripts/config -d CONFIG_AUFS_FS \

which corresponds to the following

.. code:: bash

   # Enable CONFIG_PREEMPT_RT
    -> General Setup
     -> Preemption Model (Fully Preemptible Kernel (Real-Time))
      (X) Fully Preemptible Kernel (Real-Time)

   # Enable CONFIG_HIGH_RES_TIMERS
    -> General setup
     -> Timers subsystem
      [*] High Resolution Timer Support

   # Enable CONFIG_NO_HZ_FULL
    -> General setup
     -> Timers subsystem
      -> Timer tick handling (Full dynticks system (tickless))
       (X) Full dynticks system (tickless)

   # Set CONFIG_HZ_1000
    -> Kernel Features
     -> Timer frequency (1000 HZ)
      (X) 1000 HZ

   # Set CPU_FREQ_DEFAULT_GOV_PERFORMANCE [=y]
    -> CPU Power Management
     -> CPU Frequency scaling
      -> CPU Frequency scaling (CPU_FREQ [=y])
       -> Default CPUFreq governor (<choice> [=y])
        (X) performance

   # Disable CONFIG_AUFS_FS, otherwise RT kernel build breaks
    x     -> File systems                                                                                                                          x
     x (1)   -> Miscellaneous filesystems (MISC_FILESYSTEMS [=y])

If you need to reconfigure it, run

.. code:: bash

   $ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig

kernel build
------------

.. code:: bash

   $ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j `nproc` deb-pkg

You need 32GB free disk space to build it, it takes a while, and the
results are located:

.. code:: bash

   gitpod ~/linux_build/linux-raspi-5.4.0 $ ls -la ../*.deb
   -rw-r--r-- 1 gitpod gitpod  11278252 Nov 29 14:01 ../linux-headers-5.4.86-rt48_5.4.86-rt48-1_arm64.deb
   -rw-r--r-- 1 gitpod gitpod 486149956 Nov 29 14:04 ../linux-image-5.4.86-rt48-dbg_5.4.86-rt48-1_arm64.deb
   -rw-r--r-- 1 gitpod gitpod  38504756 Nov 29 14:01 ../linux-image-5.4.86-rt48_5.4.86-rt48-1_arm64.deb
   -rw-r--r-- 1 gitpod gitpod   1054624 Nov 29 14:01 ../linux-libc-dev_5.4.86-rt48-1_arm64.deb

Deploy a new kernel on RPI4
---------------------------

download and install Ubuntu 20.04 image
---------------------------------------

Follow these links to download and install Ubuntu 20.04 on your RPI4 \*
https://ubuntu.com/download/raspberry-pi \*
https://ubuntu.com/download/raspberry-pi/thank-you?version=20.04&architecture=arm64+raspi
\*
https://ubuntu.com/tutorials/create-an-ubuntu-image-for-a-raspberry-pi-on-ubuntu#2-on-your-ubuntu-machine

.. code:: bash

   # initial username and password
   ubuntu/ubuntu

update your system
------------------

After that you need to connect to the Internet and update your system

.. code:: bash

   $ sudo apt-get update && apt-get upgrade

copy a new kernel to your system and install it
-----------------------------------------------

Assumed you have already copied all \*.deb packages to your
``$HOME/ubuntu`` directory

.. code:: bash

   $ cd $HOME/ubuntu
   $ sudo dpkg -i *.deb

adjust vmlinuz and initrd.img links
-----------------------------------

There is an extra step in compare to the x86_64 install (why is that?)

.. code:: bash

   $ cd /boot
   $ sudo ln -s -f vmlinuz-5.4.86-rt48 vmlinuz
   $ sudo ln -s -f vmlinuz-5.4.0-1029-raspi vmlinuz.old
   $ sudo ln -s -f initrd.img-5.4.86-rt48 initrd.img
   $ sudo ln -s -f initrd.img-5.4.0-1029-raspi initrd.img.old
   $ sudo cp vmlinuz firmware/vmlinuz
   $ sudo cp vmlinuz firmware/vmlinuz.bak
   $ sudo cp initrd.img firmware/initrd.img
   $ sudo cp initrd.img firmware/initrd.img.bak

   $ sudo reboot

After reboot you should see a new RT kernel installed

.. code:: bash

   ubuntu@ubuntu:~$ uname -a
   Linux ubuntu 5.4.86-rt48 #1 SMP PREEMPT_RT Sun Nov 15 22:44:33 UTC 2020 aarch64 aarch64 aarch64 GNU/Linux
