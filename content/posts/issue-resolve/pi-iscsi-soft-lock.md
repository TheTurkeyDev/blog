---
title: "I Soft Locked My Netbooted Raspberry Pi"
date: 2022-10-04T22:42:16-04:00
description: "I went to upgrade my network booted Raspberry Pi 4 from Buster to Bullseye and ended up making it fail to boot"
toc: true
tags: ["Raspberry Pi", "iSCSI", "TrueNAS"]
---

Edit: Nearly 1 year after I wrote this I did the exact same thing. Good news is that I eventually remembered this post and did successfully use it to resolve me issue. I have also edited this post from my second round at doing this.

## The Problem
If you are looking to use this to help you, please read this section first. Make sure that you both have the same, or similar issue, and that you understand which path you should go down to solve this.

I decided to upgrade my Raspberry Pi 4 that I have network booted off a TrueNAS Server using iSCSI. iSCSI is the key part here as it's needed so that the Pi can run Docker Containers off the network booted file system. Somewhere along the lines of upgrading my Pi 4 from Buster to Bullseye (Seems even some apt-upgrades can trigger this issue too) I did not correctly update the boot files and after rebooting the Pi I was greeted with errors. Mainly the 2 shown below:

```
libkmod: ERROR ../libkmod/libkmod-module.c:838 kmod_module_insert_module: could not find module by name='iscsi_tcp'
...
iscsiadm: iSCSI driver tcp is not loaded. Load the module then retry the command.
```

These ultimately lead to the Pi not being able to access the TrueNAS server to locate the disk specified in the fstab.

At this point, I had now soft-locked my Raspberry Pi with no real way to boot into it to fix it and install the missing driver.

Luckily for me, I have a second, identical, Raspberry Pi 4 (Which I haven't switched to network boot yet...) that I was able to use to help me fix this soft lock. Because of that my solution below will be using that Pi, but technically I think you should be able to use the "soft locked" Pi and just throw an SD card in it to act as a second Pi, but that will require some boot loading configuration.

## My Solution?

I knew I had to do something as it related to the initramfs image that got generated. I did a few things over the few hours I tore my hair out with this, but ultimately I think this is the chain of events that fixed it.

[This article](https://jacobrsnyder.com/2021/01/20/network-booting-a-raspberry-pi-with-docker-support/) was a huge help for both fixing this issue as well as network booting the Pi in the first place.

Again these actions are run on my second Pi unless otherwise said.

1. Update the system to match the soft-locked Pi
    ```
    $ sudo apt update
    $ sudo apt dist-upgrade #Only needed if upgrading underlying versions of the OS i.e. Buster to Bullseye
    ```
2. Install open-iscsi
    ```
    $ sudo apt install open-iscsi
    $ sudo systemctl enable open-iscsi
    $ sudo systemctl start open-iscsi
    ```
3. IMPORTANT (or at least I think it is? Please correct me if it's not). Update the "initiator name" to match the soft-locked Pi
    ```
    sudo nano /etc/iscsi/initiatorname.iscsi
    ```
    Comment out the existing `InitiatorName=....` and add a new `InitiatorName=....` entry below to be the name that the soft-locked Pi has. Because the files are on TrueNAS you can still access them via the built-in shell. I think the boot sequence also shows the name as well.
4. Reboot the Pi (not needed, but might as well I guess)
5. Setup the iSCSI "initiator"
    ```
    $ sudo touch /etc/iscsi/iscsi.initramfs
    $ sudo update-initramfs -v -k `uname -r` -c
    $ sudo vi /lib/modules-load.d/open-iscsi.conf
        Change ib_user -> #ib_user
    ```
6. Reboot again
7. Confirm that the modules have loaded successfully
    ```
    $ systemctl status systemd-modules-load
    Make sure the returned data is running and not errored
    ```
8. Assuming all has gone well you can even test to make sure the iSCSI targets have been discovered
    ```
    $ sudo iscsiadm -m discovery -t sendtargets -p <ISCSI_SERVER_IP>
    ```
9. Now that that has all been verified to work let's take this and attempt to repair our Pi. Copy the entire boot folder off this Pi to a location that can be accessed by TrueNAS. I happen to have a mount on my Desktop to an NFS share on TrueNAS, so I simply used FileZilla to copy the boot folder from the Pi to the TrueNAS NFS share.
10. Copy everything from copied boot folder to the soft-locked Pi's boot folder EXCEPT, DO NOT replace the `config.txt` and `cmdline.txt`! These files need to be left as is because they have needed information on how to boot the Pi!
11. Edit the `config.txt` file of the soft-locked Pi and update the referenced `initrd.img` to be the latest version, i.e. the one we just generated
12. Attempt to boot the Pi! If all goes well, it should hopefully now work!? if not... I'm not sure what to say...
13. Clean up step! don't forget to reset the `/etc/iscsi/initiatorname.iscsi` on the non-soft-locked Pi to the original value so that in the future it's not trying to mimic the hopefully now working Pi!


## Rambling Comments
I spent about 5 hours in total trying to figure out exactly what had happened and then stressing that I have basically bricked this Pi's OS. Because of that, I can't guarantee the steps above will solve the issue, but hopefully, this at least helps. If you did stumble upon this and it either helped or didn't, feel free to drop me a message on one of my socials (You can find a link to them on [my site](https://theturkey.dev/) under "Other Links") and let me know how this worked out for you! Maybe with your help, I can tweak these to help others! End of the day though this is pretty niche and the odds of someone else having this issue are probably slim, but hey, this post is just as much for me as it is for all of you! Obviously, backing things up and taking snapshots probably would have saved me a headache here, but alas here we are.