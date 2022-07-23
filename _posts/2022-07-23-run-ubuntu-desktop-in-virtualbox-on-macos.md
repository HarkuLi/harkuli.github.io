---
layout: post
title: "Run Ubuntu Desktop in VirtualBox on macOS"
date: 2022-07-23 21:15:00 +0800
categories: virtualization
---

I have installed Ubuntu Desktop in VirtualBox on Windows laptops, and it runs
well without any special setting. Although I have also used Ubuntu in VirtualBox
on macOS, I always interact with the virtual machine through ssh without using
GUI, and it just requires some simple configuration for ssh key. Until I needed
to use Ubuntu Desktop on my macOS laptop yesterday, I encountered some problems
during setup. In this article, I will describe those problems I met and how I
resolve them.

## Environment

* macOS Big Sur
* VirtualBox 6.1
* Ubuntu 22.04 Desktop

## TL;DR

1.  Right-click on
    "Finder > Applications > VirtualBox > Contents > Resources > VirtualBoxVM",
    select "Get Info", and check the **"Open in Low Resolution"** option.
2.  Run your guest OS.
3.  Run the following commands in the guest OS.

    ```bash
    sudo apt update
    sudo apt install build-essential dkms
    ```

4.  Select the "Devices > Insert Guest Additions CD image..." in the VirtualBox
    VM menu and run the `autorun.sh` script contained in the loaded disk in the
    guest OS.
5.  Restart the guest OS.
6.  Increase the "Settings > Display > Screen > Video Memory" configuration of
    your VM in VirtualBox Manager if the display becomes whole black when you
    try to use high resolution setting.

## Run VirtualBox VM in Low Resolution

If you start your VM with freshly installed VirtualBox on macOS, you may find
that the display of your VM doesn't go well. The text and UI are too small to
read and manipulate. To resolve it, you should run your VirtualBox VM in low
resolution:

1.  Find out the "VirtualBox" application in "Finder > Applications".
2.  Right-click and select "Show Package Contents".

    ![show_package_contents](/images/2022-07-23-run-ubuntu-desktop-in-virtualbox-on-macos/show_package_contents.png)

3.  Find out the "VirtualBoxVM" application in "Contents > Resources".
4.  Right-click, select "Get Info", and check the **"Open in Low Resolution"**
    option.

    ![open_in_low_resolution_option](/images/2022-07-23-run-ubuntu-desktop-in-virtualbox-on-macos/open_in_low_resolution_option.png)

5.  Run your guest OS.

Checking the option will also reduce lag when interacting with the GUI of guest
OS.

## Auto-resize Guest Display

By default, the resolution of VM doesn't change along with the window size in
host. The "Auto-resize Guest Display" functionality is useful when you need to
adjust the window size in host frequently. To enable it, you need to install
**VirtualBox guest additions** in the guest OS. Although it is almost an
essential step for one who is going to run a VM by VirtualBox, I still decide to
roughly describe the steps here to make this article complete:

1.  Update apt repository.

    ```bash
    sudo apt update
    ```

2.  Install the
    [build-essential](https://packages.ubuntu.com/en/focal/build-essential)
    packages.

    ```bash
    sudo apt install build-essential
    ```

3.  Optionally, you may install the
    [dkms](https://manpages.ubuntu.com/manpages/jammy/man8/dkms.8.html) package
    to keep the guest additions installed after a guest kernel update.

    ```bash
    sudo apt install dkms
    ```

4.  Select "Devices > Insert Guest Additions CD image..." in your VirtualBox VM
    menu, and the "VBoxGuestAdditions.iso" disk will be loaded in your guest OS.

    ![insert_guest_additions_cd_image](/images/2022-07-23-run-ubuntu-desktop-in-virtualbox-on-macos/insert_guest_additions_cd_image.png)

5.  Go to the directory of guest additions disk and run the `autorun.sh` script
    to start the installation.

    ![execute_guest_addtions_autorun](/images/2022-07-23-run-ubuntu-desktop-in-virtualbox-on-macos/execute_guest_addtions_autorun.png)

6.  Restart your guest OS.

## Increase Video Memory

If the display of your guest OS becomes whole black when increasing the display
resolution, it might be caused by **not enough video memory** for your VM. You
can find out the configuration of your VM at
"Settings > Display > Screen > Video Memory" in VirtualBox Manager.

![video_memory_config](/images/2022-07-23-run-ubuntu-desktop-in-virtualbox-on-macos/video_memory_config.png)

## References

* [VirtualBox running slow and lag on macOS, MacBook Pro](https://mkyong.com/mac/virtualbox-running-slow-and-lag-on-macos-macbook-pro)
* [How do I install Guest Additions in a VirtualBox VM?](https://askubuntu.com/questions/22743/how-do-i-install-guest-additions-in-a-virtualbox-vm)
* [VirtualBox Manual / Guest Additions for Linux](https://www.virtualbox.org/manual/ch04.html#additions-linux)
