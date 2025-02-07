---
layout: post
title: Making ramdisks for iOS 8.0 – 16.3.1 on 64-bit checkm8 devices
date: 2025-02-07 13:55 +1300
description: A guide for manually creating ramdisks for iOS 8.0 – 16.3.1 on 64-bit checkm8 devices
---

> **Note:**  
> When you see angle brackets (`< >`), they indicate placeholders. **Do not** include the brackets in your input. For example, `<enter>` means press the Enter key, and `<default value - 4>` means input the default value minus 4.

---

## Credits

- [verygenericname](https://github.com/verygenericname) for [SSHRD_Script](https://github.com/verygenericname/SSHRD_Script) which this guide used some commands from, and also for [sshtars](https://github.com/verygenericname/sshtars)  
- [mcg29](https://x.com/mcg29_) and [Ralph0045](https://x.com/ralph0045) for [dualbootfun](https://dualbootfun.github.io/downgrade) which this guide used some commands from  
> **Note:**  
> For the rest of the credits, see [Prerequisites](#prerequisites)
---

## Prerequisites

- **A macOS device**  
  *(You can do this on Linux using hfsplus from [xpwn](https://github.com/Kaiden-AC/xpwn), but this guide focuses on macOS.)*
- **An iPSW for your target ramdisk version**  
  *(Get these from [The Apple Wiki](https://theapplewiki.com/wiki/Firmware). For targets 11.4.2 or below, you will also need an iOS 12.0 iPSW for your device.)*
- **Firmware keys and file names for your target ramdisk version**  
  *(Obtain these from [The Apple Wiki](https://theapplewiki.com/wiki/Firmware_Keys). If file names aren’t provided, check the BuildManifest.plist in the iPSW or refer to [The iPhone Wiki](https://www.theiphonewiki.com/wiki/Firmware_Keys).)*
- **A tool to put your device in pwned-DFU mode**  
  *(Recommended: [gaster](https://github.com/0x7ff/gaster) or [ipwnder_lite](https://github.com/dora2ios/ipwnder_lite).)*
- **Python 3.6 or above**
- [zstd](https://formulae.brew.sh/formula/zstd) by Meta  
- [img4lib](https://github.com/xerub/img4lib) by xerub  
- [img4tool](https://github.com/tihmstar/img4tool) by tihmstar  
- [tsschecker](https://github.com/tihmstar/tsschecker) by tihmstar  
- [libirecovery](https://formulae.brew.sh/formula/libirecovery) by libimobiledevice  
- [KPlooshFinder](https://github.com/palera1n/KPlooshFinder) by plooshi and palera1n  
- [kerneldiff](https://github.com/mcg29/kerneldiff) by mcg29  
- [ssh.tar.zst](/assets/checkm8-x64-ramdisk/ssh.tar.zst) by verygenericname  
  *(Modified by me to add a few HFS+ tools.)*

---

## Preparations

1. **Get device info:**  
   Run the following with your device connected in DFU or recovery mode:  
   ```bash
   irecovery -q
   ```

2. **Fetch SHSH blobs:**  
   ```bash
   tsschecker -d <PRODUCT> -l -e <ECID> -B <MODEL> -l -s
   ```

3. **Convert blob to IM4M:**  
   ```bash
   img4tool -e -s *.shsh2 -m IM4M
   ```

---

## Patching Bootchain

1. **Decrypt iBSS and iBEC from the target iPSW:**  
   ```bash
   img4 -i <iBSS> -o iBSS.dec -k <ivkey>
   img4 -i <iBEC> -o iBEC.dec -k <ivkey>
   ```

2. **Patch with kairos:**  
   ```bash
   kairos iBSS.dec iBSS.patched
   kairos iBEC.dec iBEC.patched -b "rd=md0 debug=0x2014e -v wdt=-1"
   ```  
   > **Note:**  
   > If your device CPID is `0x8960`, `0x7000`, or `0x7001`, add `nand-enable-reformat=1 -restore` to iBEC boot args.  
   > For iOS 9 and below, also add `amfi=0xff cs_enforcement_disable=1`.

3. **Pack the patched images:**  
   ```bash
   img4 -i iBSS.patched -o iBSS.img4 -M IM4M -A -T ibss
   img4 -i iBEC.patched -o iBEC.img4 -M IM4M -A -T ibec
   ```

---

## Decrypting & Patching Components

### Kernelcache

#### For iOS 10 and up

1. **Decrypt the kernelcache:**  
   ```bash
   img4 -i <kernelcache> -o kcache.raw
   ```

2. **Patch the kernelcache:**  
   ```bash
   KPlooshFinder kcache.raw kcache.patched
   python kerneldiff.py kcache.raw kcache.patched
   img4 -i <kernelcache> -o kernelcache.img4 -M IM4M -T rkrn -P kc.bpatch
   ```

#### For iOS 9 and below

For iOS 9 and below, **skip the patching step** and instead use:
   ```bash
   img4 -i <kernelcache> -o kernelcache.im4p -k <ivkey> -D
   img4 -i kernelcache.im4p -o kernelcache.img4 -M IM4M -T rkrn
   ```

### DeviceTree

#### For iOS 10 and up

1. **Decrypt the devicetree:**  
   ```bash
   img4 -i <devicetree> -o devicetree.img4 -M IM4M -T rdtr
   ```

#### For iOS 9 and below

1. **Decrypt using the IV key:**  
   ```bash
   img4 -i <devicetree> -o dtree.raw -k <ivkey>
   img4 -i dtree.raw -o devicetree.img4 -A -M IM4M -T rdtr
   ```

### Restore Ramdisk

#### For iOS 10 and up

1. **Decrypt the restore ramdisk:**  
   ```bash
   img4 -i <restore_ramdisk> -o ramdisk.dmg
   ```

#### For iOS 9 and below

1. **Decrypt using the IV key:**  
   ```bash
   img4 -i <restore_ramdisk> -o ramdisk.dmg -k <ivkey>
   ```

### Trustcache (iOS 12.0+ ramdisks only)

1. **Decrypt the trustcache:**  
   ```bash
   img4 -i <restore_ramdisk_trustcache> -o trustcache.img4 -M IM4M -T rtsc
   ```

---

## Making the Ramdisk

1. **Extract the SSH ramdisk tar to the restore ramdisk:**

   - **Resize the disk image:**  
     ```bash
     hdiutil resize -size 210MB ramdisk.dmg
     ```
   - **Attach the ramdisk:**  
     ```bash
     hdiutil attach ramdisk.dmg -mountpoint /tmp/rd
     sudo diskutil enableOwnership /tmp/rd
     ```
   - **Extract the tarball:**  
     ```bash
     sudo tar --zstd -xvf ssh.tar.zst -C /tmp/rd
     ```

2. **Additional step for targets 11.4.2 or below**  
   *(Run this after the SSH tar has been extracted, if applicable)*

   If your target ramdisk version is 11.4.2 or below, copy some dylibs from an iOS 12.0 restore ramdisk:

   - **Decrypt the 12.0 restore ramdisk:**  
     ```bash
     img4 -i <12_restore_ramdisk> -o ramdisk12.dmg
     ```
   - **Mount the 12.0 restore ramdisk and copy libraries:**  
     ```bash
     hdiutil attach ramdisk12.dmg -mountpoint /tmp/rd12
     sudo diskutil enableOwnership /tmp/rd12
     sudo cp -a /tmp/rd12/usr/lib/libiconv.2.dylib /tmp/rd12/usr/lib/libcharset.1.dylib /tmp/rd/usr/lib
     ```
   - **Detach any mounted ramdisks:**  
     ```bash
     hdiutil detach /tmp/rd12
     hdiutil detach /tmp/rd
     ```

   *If you did not perform the additional step, be sure to detach the primary ramdisk after extraction:*  
   ```bash
   hdiutil detach /tmp/rd
   ```

3. **Optimize the ramdisk image size:**  
   Before packing the ramdisk into img4, use `hdiutil resize -limits` to determine the minimal size needed and then resize the image accordingly. For example:
   
   ```bash
   hdiutil resize -limits ramdisk.dmg
   ```
   
   Review the output to find the minimum size required, and then run:
   
   ```bash
   hdiutil resize -size <MINIMUM_SIZE> ramdisk.dmg
   ```
   
   Replace `<MINIMUM_SIZE>` with the smallest size value provided in the limits output.

4. **Pack the ramdisk into img4:**  
   ```bash
   img4 -i ramdisk.dmg -o ramdisk.img4 -M IM4M -A -T rdsk
   ```

---

## Booting the Ramdisk

1. **Put the device into pwned-DFU mode.**

2. **Send bootchain components in order:**

   - **Send iBSS:**  
     ```bash
     irecovery -f iBSS.img4
     ```
   - **Send iBEC:**  
     ```bash
     irecovery -f iBEC.img4
     ```
   - **Send the ramdisk:**  
     ```bash
     irecovery -f ramdisk.img4
     ```
   - **Execute the ramdisk:**  
     ```bash
     irecovery -c ramdisk
     ```
   - **Send the DeviceTree:**  
     ```bash
     irecovery -f devicetree.img4
     ```
   - **Execute the DeviceTree:**  
     ```bash
     irecovery -c devicetree
     ```
   - **(For iOS 12.0+ only) Send the Trustcache:**  
     ```bash
     irecovery -f trustcache.img4
     ```
   - **(For iOS 12.0+ only) Execute the Trustcache:**  
     ```bash
     irecovery -c trustcache
     ```
   - **Send the Kernelcache:**  
     ```bash
     irecovery -f kernelcache.img4
     ```

3. **Boot the device:**  
   ```bash
   irecovery -c bootx
   ```

---

## SSH into the Device

1. **Start an iproxy tunnel in one terminal:**  
   ```bash
   iproxy 2222 22
   ```

2. **Open another terminal and SSH:**  
   ```bash
   ssh -p2222 -oStrictHostKeyChecking=no root@localhost
   ```

**Done!**
