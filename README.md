# Full Disk Encryption with TPM 2.0 on Garuda Linux 

***Very important note:*** *Make a backup of your data. Any incompatibilities or mistakes while following the procedures in this guide could lead to irreversible data loss!*


# Introduction
**Tested on:**
* OS: "garuda-dr460nized-linux-zen-210406"
* Kernel: "5.11.11-zen1-1-zen" and "5.10.28-1-lts"
* Installed using default GUI installer
* Installed using default partitioning (swap or no swap) + FDE option enabled.

**Requirements:**
* System provisioned with TPM
* TPM chip enabled in UEFI settings

***Notes:***
* *It should work on Arch Linux with some minor changes, but I haven't tested it.*
* *Custom partition layout should work as well, but you'll have to adjust the crypttab and TPM bindings accordingly.*
* *This setup can work with Secure Boot if you sign your EFI image, but this guide will not go there...*
* *If you are stuck on the loading screen press ESC.*

## Preparations
***Very important note**: Do not reboot your system until you've finished all the steps, or you won't be able to boot.*
1. Edit the file /etc/crypttab and change:</br>
 Choose depending on your partition setup **A. (if you you're not using swap)** or **B. (if you're using swap)**.

***Note**: The device with "/crypto_keyfile.bin luks" parameters should be the swap partition, where the device with "none discard" parameters should be the root partition.*

**(A)** From:
```
# <name>               <device>                         <password> <options>
luks-<id> UUID=<id>     /crypto_keyfile.bin luks
```
**(A)** To:

```
# <name>               <device>                         <password> <options>
luks-<id> UUID=<id> none discard
```

**(B)** From:
```
# <name>               <device>                         <password> <options>
luks-<id> UUID=<id>     /crypto_keyfile.bin luks
luks-<id> UUID=<id>     /crypto_keyfile.bin luks
```
**(B)** To:
```
# <name>               <device>                         <password> <options>
luks-<id> UUID=<id> none discard
luks-<id> UUID=<id>     /crypto_keyfile.bin luks
```
3. Delete the file "/crypto_keyfile.bin" (skip this step if you are using swap):

4. ***(do NOT skip this step!)*** Edit the intial ramdisk conf file `/etc/mkinitcpio.conf` and change this line from: `FILES="/crypto_keyfile.bin"`
to:
`#FILES="/crypto_keyfile.bin"`

5. Edit `/etc/mkinitcpio.conf`, set the respective hooks based of which method you choose:</br>
Change this line `HOOKS="base udev autodetect modconf block keyboard keymap consolefont plymouth encrypt filesystems"` to the following:

* **For Method 1 - Clevis:**</br>
`HOOKS="base udev autodetect modconf block keyboard keymap consolefont clevis encrypt filesystems"`

* **For Method 2 - Custom (WIP Avoid for now)**</br>
`HOOKS="base udev autodetect modconf block keyboard keymap consolefont encrypt-tpm encrypt filesystems"`

## Method 1 - Clevis

**Setup**:

1. Install the following packages.
```
pacman --needed -S clevis tpm2-tools luksmeta libpwquality
```
2. Add `clevis` binding to your LUKS device </br>
**Note:** Set the [PCR registers](https://wiki.archlinux.org/index.php/Trusted_Platform_Module#Accessing_PCR_registers) based on your paranoia setting...
```
clevis luks bind -d <device> tpm2 '{"pcr_bank":"sha256","pcr_ids":"0,1,2,4,7,8"}'
```
3. Install the `clevis` hook from [mkinitcpio-clevis-hook](https://github.com/kishorv06/arch-mkinitcpio-clevis-hook)
```
git clone https://github.com/kishorv06/arch-mkinitcpio-clevis-hook.git
cd arch-mkinitcpio-clevis-hook
./install.sh
```

4. Regenerate `initramfs` image.
```
mkinitcpio -P
```
5. Reboot your system. Now your disk should get decrypted using the key from TPM.

**Note:**
If integrity on your system is changed, you will get prompted to manually enter the password for decryption since TPM will not be able to unseal the key.

It is actually recomended to test this.
1. Open your UEFI settings. 
2. Find the TPM settings (most common location is in security menu/tab).
3. Delete the keys.
4. Boot. 
Now you will be notified that the TPM key could not be unsealed, and you will be prompted to enter a password for decryption, to fix this follow the next section **"Clevis Binding"**.

**Regenerate Clevis Binding**</br>
To regenerate a Clevis binding after changes in system's configuration that result in different PCR values:

1. Find the slot used for the Clevis pin
`cryptsetup luksDump <luskDevice>`
2. Remove the Clevis binding, run:
`clevis luks regen -d <luksDevice> -s <keySlot>`
3. Add a new Clevis binding.
`
clevis luks bind -d <luksDevice> tpm2 '{"pcr_bank":"sha256","pcr_ids":"0,1,2,4,7,8"}'
`
4. Reboot, the disk now should be decrypted using the key from TPM.


**Remove Clevis Binding**</br>
To remove a Clevis binding:

1. Find the slot used for the Clevis pin
`cryptsetup luksDump <luksDevice>`
2. Remove the Clevis binding, run:
`clevis luks unbind -d <luksDevice> -s <keySlot>`

**Avoid password prompt in GRUB (OPTIONAL)**
1.Rebuilt the EFI image using:
```
objcopy \
    --add-section .osrel="/usr/lib/os-release" --change-section-vma .osrel=0x20000 \
    --add-section .cmdline="/proc/cmdline" --change-section-vma .cmdline=0x30000 \
    --add-section .linux="/boot/vmlinuz-linux-zen" --change-section-vma .linux=0x40000 \
    --add-section .initrd="/boot/initramfs-linux-zen.img" --change-section-vma .initrd=0x3000000 \
    "/usr/lib/systemd/boot/efi/linuxx64.efi.stub" "/boot/efi/EFI/Garuda/grubx64.efi"
```
2. Regenerate the Clevis binding. 

**Note:**
If you are using LTS  change `vmlinuz-linux-zen.img` and `initramfs-linux-zen.img` to `vmlinuz-linux-lts.img` and `initramfs-linux-lts.img`


## Method 2 - Custom

1. Create the key
```
dd if=/dev/random of=/root/secret.bin bs=32 count=1
```

2. Add the key to luks
```
cryptsetup luksAddKey /dev/<luksDevice> /root/secret.bin
```

3. Add key to TPM
```
tpm2_createpolicy --policy-pcr -l sha1:0,2,4,7 -L policy.digest
tpm2_createprimary -C e -g sha1 -G rsa -c primary.context
tpm2_create -g sha256 -u obj.pub -r obj.priv -C primary.context -L policy.digest -a "noda|adminwithpolicy|fixedparent|fixedtpm" -i /root/secret.bin
tpm2_load -C primary.context -u obj.pub -r obj.priv -c load.context
tpm2_evictcontrol -C o -c load.context 0x81000000
rm load.context obj.priv obj.pub policy.digest primary.context
```

4. Unseal:
```sh
tpm2_unseal -c 0x81000000 -p pcr:sha1:0,7 -o /crypto_keyfile.bin
```
5. Install the hooks:</br>
```
git clone https://github.com/SubXi/GarudaLinux-FDE_and_TPM-Guide.git
chmod +x install.sh
sudo ./install.sh
```
6. Rebuild inital ramdisk 
```
mkinitcpio -P
```
7. (OPTIONAL) Avoid password prompt in GRUB using EFI kernel STUB.</br>
Rebuilt the EFI image using:
```
objcopy \
    --add-section .osrel="/usr/lib/os-release" --change-section-vma .osrel=0x20000 \
    --add-section .cmdline="/proc/cmdline" --change-section-vma .cmdline=0x30000 \
    --add-section .linux="/boot/vmlinuz-linux-zen" --change-section-vma .linux=0x40000 \
    --add-section .initrd="/boot/initramfs-linux-zen.img" --change-section-vma .initrd=0x3000000 \
    "/usr/lib/systemd/boot/efi/linuxx64.efi.stub" "/boot/efi/EFI/Garuda/grubx64.efi"
```

**Note:**
After reboot, you should get prompted to input the password manually. This behavior is expected since you changed your EFI image. Remove the key from TPM using the below command and redo step 3:
```
tpm2_evictcontrol -C o -c 0x81000000
```


## Some other important notes:
If the system unexpectedly asks for LUKS password after reboot it may indicate that your system was compromised.

Always test your system to see if the TPM is handled properly.</br> 
I suggest the following procedure:
1. Bind the TPM keys, test if you successfully boot without issues.

   A. If you don't boot: troubleshoot.
   
   B. If you do boot: continue to the next step.
   
2. Clear/delete the keys stored on TPM, preferably by using the UEFI menu.

   A. If the system boots: you have a serious issue and you should troubleshoot.
   
   B. If the system prompts for password input: you are good to go just regenerate the TPM binding.

***
Sources and relevant material:

https://wiki.archlinux.org/index.php/Trusted_Platform_Module

https://github.com/pawitp/arch-luks-tpm

https://pawitp.medium.com/full-disk-encryption-on-arch-linux-backed-by-tpm-2-0-c0892cab9704

https://github.com/kishorv06/arch-mkinitcpio-clevis-hook

https://bentley.link/secureboot/

https://github.com/archont00/arch-linux-luks-tpm-boot

https://github.com/saucepan14/TPMSecuredArch
