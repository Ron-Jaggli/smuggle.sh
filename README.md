# Magisk Delta on Waydroid - build prerooted images!

![installedmagisk](https://magiskwaydroid.fra1.digitaloceanspaces.com/installedmagisk.png)
![magiskmodules](https://magiskwaydroid.fra1.digitaloceanspaces.com/magiskmodules.png)

**Follow [installation](#installation-guide) to init modified images.**

Note: This is forked https://github.com/pagkly/MagiskOnWaydroid, but It has been and abandoned and building doesn't work anymore. So I fixed it and we can build prerooted images again! Please use Magisk Delta, normal Magisk is very buggy, but the option is still there, If you really want to.

Forked [MagiskOnWSA](https://github.com/LSPosed/MagiskOnWSA) and modified to install Magisk Delta and OpenGapps (pico) replacing stock Google Apps.

Most OpenGApps variants works except for aroma (aroma does not support x86_64, please use super instead)

Tested in ArchLinux, Fedora. Archlinux users, please use linux-xanmod-anbox kernel, or It will not work at all!

More in [Waydroid Telegram](https://t.me/WayDroid)

## Bugs
1. **Zygisk not yet working (No ETA)**. See [docs/zygisk-integration-plan.md](docs/zygisk-integration-plan.md) for the design/proposal to make it work.

    Currently, this script is using MagiskonWSA method in patching initrc so that it would load magisk su binaries close to the end of initrc, before ui started. However zygote is usually loaded at the start of initrc, meaning it would be too late to replace zygote with zygisk zygote by the time su from magisk is loaded. 
    
    In [LSPosed's MagiskonWSA](https://github.com/LSPosed/MagiskonWSA) implementation, there is a [patched WSA kernel with su binaries](https://github.com/LSPosed/WSA-Kernel-SU) just like in Android devices where Magisk patch recovery/boot image to have su binaries. Waydroid, on the other hand, uses lxc containers utilizing linux host kernel without su binaries needed to patch zygote.
    
    There might be a way to load su binaries as [kernel module when lxc session is starting](https://askubuntu.com/questions/314817/how-do-i-install-a-kernel-module-in-an-lxc-guest-machine) with one caveat that this solution might introduce security issues to linux host. Anybody who is well-versed in lxc can contact me/create issue to explain to me how to make it work.
    
    Thus, modules requiring zygote/zygisk like [Riru](https://github.com/RikkaApps/Riru), [LSPosed](https://github.com/LSPosed/LSPosed) (pre-zygisk) or Shamiko (module to hide magisk root utilizing zygisk) unfortunately wont work for now unless someone figures out a workaround.
    
    In the meantime, here is some examples of modules that work: 
    - [Busybox NDK](https://github.com/Magisk-Modules-Repo/busybox-ndk)
    - [Magisk Hide Prop](https://github.com/Magisk-Modules-Repo/MagiskHidePropsConf)
    - [Detach](https://github.com/Magisk-Modules-Repo/Detach) (detach app from play store)
    
    
**Available Magisk versions**.
    
    Magisk 26.4
    Magisk 26.4 - canary (Normal Magisk just for testing purposes)
    Magisk Delta latest (Highly recommended)

    
## Cleaning Previous Image
Run this command below to clean previous images, before installing new Waydroid image or when updating to new Magisk.

**Warning: You will lose all your data, apps, settings of previous Android installation.**

**Kindly backup everything in your Android before you proceed**
```shell
waydroid session stop
sudo waydroid container stop
sudo systemctl stop waydroid-container.service
sudo rm -rf /var/lib/waydroid /home/.waydroid ~/waydroid ~/.share/waydroid ~/.local/share/applications/*aydroid* ~/.local/share/waydroid
sudo rm -rf /etc/waydroid-extra/images/*
```
  
## Installation Guide

1. Fork this repo and go to the **Action** tab.
2. In the left sidebar, click the **Modify Waydroid** workflow.
3. Above the list of workflow runs, select **Run workflow**
4. Select Magisk version and gapps version and the select **Run workflow**
5. Wait for the action to complete and download the artifact
6. Unzip the artifact
    - The size shown in the webpage is uncompressed size and the zip you download will be compressed. So the size of the zip will be much less than the size shown in the webpage.
7. Copy system.img and vendor.img to /etc/waydroid-extra/images/ and init the new img.
    ```shell    
    # Remove original img and copy magisk img
    sudo rm -rf /var/lib/waydroid/images/system.img
    sudo rm -rf /var/lib/waydroid/images/vendor.img
    sudo cp system.img /etc/waydroid-extra/images/system.img
    sudo cp vendor.img /etc/waydroid-extra/image/vendor.img

    # Initialize new images
    sudo waydroid init -f
    # Restart waydroid lxc container
    sudo systemctl restart waydroid-container.service
    waydroid session start or waydroid show-full-ui
    ```
8. After additional setup in magisk delta, please go in install section and select direct install to system partition and install it!
  ![directinstall](https://magiskwaydroid.fra1.digitaloceanspaces.com/directinstallsystem.png)
      
## Additional FAQ

- Why the size of the zip does not match the one shown?

   The zip you downloaded is compressed and Github is showing the uncompressed size.
- How can I update System.img to new version?

    Rerun the Github action, download the new artifact, replace the content of your previous installation.
- How can I update Magisk to new version?

    Do the same as updating System.img
- How to pass safetynet?

    Like all the other emulators, no way.
- Magisk online module list is empty?

    Install manually by 
   
    ```shell
    adb push module.zip /data/local/tmp
    adb shell su -c magisk --install-module /data/local/tmp/module.zip
    ```

- Github Action script is updated, how can I synchronize it?

    1. In your fork repository, click `fetch upstream`
        ![fetch](https://docs.github.com/assets/cb-33284/images/help/repository/fetch-upstream-drop-down.png)
    1. Then and click `fetch and merge`
        ![merge](https://docs.github.com/assets/cb-128489/images/help/repository/fetch-and-merge-button.png)

## Credits
- [Waydroid](https://github.com/waydroid/waydroid)
- [Magisk](https://github.com/topjohnwu/Magisk)
- [The Open GApps Project](https://opengapps.org)
- [WSA-Kernel-SU](https://github.com/LSPosed/WSA-Kernel-SU) and [kernel-assisted-superuser](https://git.zx2c4.com/kernel-assisted-superuser/)
- [WSAGAScript](https://github.com/ADeltaX/WSAGAScript)
- [MagiskonWSA](https://github.com/LSPosed/MagiskonWSA)
- [MagiskOnEmulator](https://github.com/shakalaca/MagiskOnEmulator)
- [MagiskOnEmu](https://github.com/HuskyDG/MagiskOnEmu)
- [Project Kokoro](https://github.com/supremegamers/kokoro) - Magisk module remount implementation

