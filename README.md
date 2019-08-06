# Multiboot Chromium OS with another OSes on Surface devices

## Build the whole Chromium OS by yourself

Basically, follow this guide:
- [Chromium OS Docs - Chromium OS Developer Guide](https://chromium.googlesource.com/chromiumos/docs/+/master/developer_guide.md)

I'll explain what is not described in the above guide.

References (some sources written in Japanese):
- [Chromium OS Docs - Chromium OS Developer Guide](https://chromium.googlesource.com/chromiumos/docs/+/master/developer_guide.md)
- [Chromium OSをビルド - Qiita](https://qiita.com/maeda_mikio/items/841ef4b64a78dd1d1d28)
- [Chromium OSでバッテリ駆動時間は伸びるのか? NEC LaVie Zで試す - ビルド実行編](https://blog.c6h12o6.org/post/chromiumos-self-build-build/)



### Storage requirement

It used up to about 200GB while building `stabilize-11895.118.B`


### How to checkout repo to specific branch?
```bash
# (outside cros_sdk)

# first, checkout to master
repo init -u https://chromium.googlesource.com/chromiumos/manifest.git --repo-url https://chromium.googlesource.com/external/repo.git -b master
# if this is for the first time to sync, `-j8` is recommended
repo sync -j8

# then, checkout to another branch if you want
# you can see the list of branches here:
# https://chromium.googlesource.com/chromiumos/manifest.git

# checkout to release-R76-12239.B
repo init -u https://chromium.googlesource.com/chromiumos/manifest.git --repo-url https://chromium.googlesource.com/external/repo.git -b release-R76-12239.B
# then, sync
repo sync -j16

# checkout to stabilize-12331.B
repo init -u https://chromium.googlesource.com/chromiumos/manifest.git --repo-url https://chromium.googlesource.com/external/repo.git -b stabilize-12331.B
# then, sync
repo sync -j16
```

### Build a Chromium OS dev image with modified kernel
```bash
# before entering chroot, I recommend to include your Linux distribution's firmware
# Especially, don't forget to add IPTS firmware
#cp -r /lib/firmware/* $HOME/chromiumos/chroot/build/amd64-generic/lib/firmware/
# Or, clone it from kernel.org
cd ~
git clone --depth 1 https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git
cp -r linux-firmware/* $HOME/chromiumos/chroot/build/amd64-generic/lib/firmware/

cros_sdk # (inside cros_sdk)
export BOARD=amd64-generic
setup_board --board=${BOARD}

# If you want to modify kernel, do it here
# Follow my guide "How to build a kernel (using `cros_sdk`)"
# https://github.com/kitakar5525/chromeos-kernel-linux-surface#fixme-module-load-order-for-built-in-modules

# Build a dev image
FEATURES="noclean" ./build_packages --board=${BOARD} && ./build_image --board=${BOARD} --noenable_rootfs_verification dev

# also, if you want, generate a VM image here
#INFO    : To convert it to a VM image, use:
#INFO    :   ./image_to_vm.sh --from=../build/images/amd64-generic/R77-12342.0.2019_07_14_1634-a1 --board=amd64-generic
#You can start the image with:
#cros_vm --start --image-path /mnt/host/source/src/build/images/amd64-generic/R77-12342.0.2019_07_14_1634-a1/chromiumos_qemu_image.bin
```



### How to increase ROOT partition size of already built image?

```bash
export IMAGE_NAME=chromiumos_image-surface.bin
truncate -s +3g $IMAGE_NAME # increase by 3BG
LOOP_LOCATION=$(losetup -f) # /dev/loopX # get where the image will be mounted
sudo losetup $LOOP_LOCATION -P $IMAGE_NAME
gparted $LOOP_LOCATION # manipulate partitions here
sudo losetup -d $LOOP_LOCATION # unmount that device
```

References
- [Shrinking images on Linux - Softwarebakery](https://softwarebakery.com/shrinking-images-on-linux)

### How to increase ROOT partition size when building a image?

You may achieve this by editing
`(inside cros_sdk) ~/trunk/src/overlays/overlay-amd64-generic/scripts/disk_layout.json`? (not tested)



## How to retrieve the built image to home directory
```bash
(outside cros_sdk)
cd ~
export IMAGE_EXPORT_DIR=~/chromiumos-$(basename $(realpath $HOME/chromiumos/src/build/images/amd64-generic/latest))
mkdir $IMAGE_EXPORT_DIR
cp -r $HOME/chromiumos/src/build/images/amd64-generic/latest/* $IMAGE_EXPORT_DIR
```



## Or, use ArnoldTheBat's or CloudReady or any other prebuilt image with modified kernel

Copy module into `ROOT-A/lib/modules` and copy vmlinuz to `EFI-SYSTEM/syslinux/vmlinuz.A`.
Also, I recommend to include your Linux distribution's firmware.
Especially, don't forget to add IPTS firmware.
```bash
cp -r /lib/firmware/* ROOT-A/lib/firmware/
```



---



## Modify the image to use on Surface devices

Modify the image before writing to your internal storage.
Of course you can do it after writing to your internal storage, but in case
you need to reuse this image.

First, mount ROOT-A partition of the image
```bash
# first, copy or rename the image name
#cp chromiumos_image.bin chromiumos_image-surface.bin

export IMAGE_NAME=chromiumos_image-surface.bin 
LOOP_LOCATION=$(losetup -f) # /dev/loopX # get where the image will be mounted
sudo losetup $LOOP_LOCATION -P $IMAGE_NAME
sleep 1 # for lsblk to recognize the changes

# get location of ROOT-A
LOCATION_ROOT_A=$(lsblk -o NAME,PARTLABEL -r | grep $(basename $LOOP_LOCATION) | grep ROOT-A | cut -d" " -f1) # loopXpY
mkdir -p /tmp/mnt/ROOT-A; sudo mount /dev/$LOCATION_ROOT_A /tmp/mnt/ROOT-A # mount ROOT-A
```

### Modify `/usr/sbin/write_gpt.sh`

It seems that, at least to just boot the image, you only need `STATE` partition number.
I recommend to use something greater number (explain later how to change actual partition number)
to distinguish Chromium OS pertitions from the other OS partitions.
Another reason is, if I set the number to somewhat greater number,
the image can also be used on another devices without editing `write_gpt.sh`.
I personally use `52`, so, the file will look like this:

```bash
load_base_vars() {
  PARTITION_SIZE_STATE=2147483648
    RESERVED_EBS_STATE=0
       DATA_SIZE_STATE=2147483648
          FORMAT_STATE=
       FS_FORMAT_STATE=ext4
      FS_OPTIONS_STATE=""
   PARTITION_NUM_STATE="52"
  PARTITION_SIZE_52=2147483648
    RESERVED_EBS_52=0
       DATA_SIZE_52=2147483648
          FORMAT_52=
       FS_FORMAT_52=ext4
      FS_OPTIONS_52=""
   PARTITION_NUM_52="52"
}

load_partition_vars() {
  PARTITION_SIZE_STATE=2147483648
    RESERVED_EBS_STATE=0
       DATA_SIZE_STATE=2147483648
          FORMAT_STATE=
       FS_FORMAT_STATE=ext4
      FS_OPTIONS_STATE=""
   PARTITION_NUM_STATE="52"
  PARTITION_SIZE_52=2147483648
    RESERVED_EBS_52=0
       DATA_SIZE_52=2147483648
          FORMAT_52=
       FS_FORMAT_52=ext4
      FS_OPTIONS_52=""
   PARTITION_NUM_52="52"
}
```

### another modifications to use on Surface devices

#### linux-firmware
You should want to copy firmware files to the image if you did not before.
Otherwise, marvel mwifiex wifi will not work.
```bash
# from kernel.org
# git clone --depth 1 https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git
# Or, from arch
wget --trust-server-names https://www.archlinux.org/packages/core/any/linux-firmware/download/
mkdir linux-firmware-20190628.70e4394-1-any.pkg
tar -xf linux-firmware-20190628.70e4394-1-any.pkg.tar.xz -C linux-firmware-20190628.70e4394-1-any.pkg
# don't forget to add ipts firmware
cp -r ipts linux-firmware-20190628.70e4394-1-any.pkg/usr/lib/firmware/intel/
sudo cp -r linux-firmware-20190628.70e4394-1-any.pkg/usr/lib/firmware/* /tmp/mnt/ROOT-A/lib/firmware
```

#### Sound on Surface Book 1 may not working by default.
You may need to comment out the line in a file `/etc/modprobe.d/alsa-skl.conf`
```
blacklist snd_hda_intel
```

#### Sound is not working on Surface 3 by default. Add UCM file:
```bash
git clone --depth 1 https://github.com/plbossart/UCM ucm
sudo cp -r ucm/chtrt5645 /tmp/mnt/ROOT-A/usr/share/alsa/ucm/
```

#### Tap to click is not working by default. Edit `/etc/gesture/40-touchpad-cmt.conf`
```diff
Section "InputClass"
    Identifier      "touchpad"
[...]
    # CMT devices potentially process keyboard events
    Option          "XkbModel" "pc"
    Option          "XkbLayout" "us"
+
+    # for Surface series touchpad tap to click
+    Option          "libinput Tapping Enabled" "1"
+    Option          "Tap Minimum Pressure" "0.1"
EndSection
```

then, `sudo restart ui`

References:
- [Problem With alps touchpad ? Issue #128 ? arnoldthebat/chromiumos](https://github.com/arnoldthebat/chromiumos/issues/128)

#### Disable hardware TPM (valid only if you build your tpm module as external module, not built-in)
Make a new file `etc/modprobe.d/hw-tpm-blacklist.conf`
```
install tpm_tis /bin/false
blacklist tpm_tis
```

and add vtpm init conf file:
Make a new file `etc/init/_vtpm.conf`
```
start on started boot-services

script
    mkdir -p /var/lib/trunks
    modprobe tpm_vtpm_proxy
    swtpm chardev --vtpm-proxy --tpm2 --tpmstate dir=/var/lib/trunks --ctrl type=tcp,port=10001
    swtpm_ioctl --tcp :10001 -i
end script
```

Install swtpm
```
wget https://github.com/imperador/chromefy/raw/master/swtpm.tar
tar -xf swtpm.tar
cd swtpm/usr/lib64/
ln -s libswtpm_libtpms.so.0.0.0 libswtpm_libtpms.so.0
ln -s libswtpm_libtpms.so.0 libswtpm_libtpms.so
ln -s libtpms.so.0.6.0 libtpms.so.0
ln -s libtpms.so.0 libtpms.so
ln -s libtpm_unseal.so.1.0.0 libtpm_unseal.so.1
ln -s libtpm_unseal.so.1 libtpm_unseal.so
cd -
sudo cp -r swtpm/usr /tmp/mnt/ROOT-A/
```

#### You may also want to disable `mwlwifi` module:
Make a new file `etc/modprobe.d/mwlwifi-blacklist.conf`
```
blacklist mwlwifi
```

#### Useful chrome flags
Edit `etc/chrome_dev.conf`
```
--enable-features=DoubleTapToZoomInTabletMode,DriveFS,PDFAnnotations,VirtualDesks,VizDisplayCompositor,VizHitTest
--force-tablet-mode=touch_view
--top-chrome-touch-ui=disabled
--ash-debug-shortcuts
--show-taps
--pull-to-refresh=1

#--policy-switches-begin
--force-gpu-rasterization
--enable-oop-rasterization
--enable-zero-copy
--ignore-gpu-blacklist
--enable-features=VizDisplayCompositor
--disable-gpu-driver-workarounds
#--policy-switches-end
```

#### You may want to add additional fonts
```bash
sudo cp -r /usr/share/fonts/noto* usr/share/fonts/

# Or, get from arch
wget --trust-server-names https://www.archlinux.org/packages/extra/any/noto-fonts/download/
wget --trust-server-names https://www.archlinux.org/packages/extra/any/noto-fonts-emoji/download/
wget --trust-server-names https://www.archlinux.org/packages/extra/any/noto-fonts-cjk/download/
wget --trust-server-names https://www.archlinux.org/packages/community/x86_64/powerline-fonts/download/
wget --trust-server-names https://www.archlinux.org/packages/community/any/awesome-terminal-fonts/download/

mkdir noto-fonts-20190111-2-any.pkg noto-fonts-emoji-20180810-2-any.pkg noto-fonts-cjk-20190409-1-any.pkg
mkdir powerline-fonts-2.7-3-x86_64.pkg awesome-terminal-fonts-1.1.0-2-any.pkg

tar -xf noto-fonts-20190111-2-any.pkg.tar.xz -C noto-fonts-20190111-2-any.pkg
tar -xf noto-fonts-emoji-20180810-2-any.pkg.tar.xz -C noto-fonts-emoji-20180810-2-any.pkg
tar -xf noto-fonts-cjk-20190409-1-any.pkg.tar.xz -C noto-fonts-cjk-20190409-1-any.pkg
tar -xf powerline-fonts-2.7-3-x86_64.pkg.tar.xz -C powerline-fonts-2.7-3-x86_64.pkg
tar -xf awesome-terminal-fonts-1.1.0-2-any.pkg.tar.xz -C awesome-terminal-fonts-1.1.0-2-any.pkg

sudo cp -r noto-fonts-20190111-2-any.pkg/{etc,usr} /tmp/mnt/ROOT-A/
sudo cp -r noto-fonts-emoji-20180810-2-any.pkg/{etc,usr} /tmp/mnt/ROOT-A/
sudo cp -r noto-fonts-cjk-20190409-1-any.pkg/{etc,usr} /tmp/mnt/ROOT-A/
sudo cp -r awesome-terminal-fonts-1.1.0-2-any.pkg/{etc,usr} /tmp/mnt/ROOT-A/
sudo cp -r powerline-fonts-2.7-3-x86_64.pkg/{etc,usr} /tmp/mnt/ROOT-A/
```

#### (optional) if you want to boot this image from USB storage, change STATE partition number
```bash
sudo gdisk $LOOP_LOCATION
Command (? for help): x
Command (? for help): t
# assume STATE is currently partition 1, change if not
Partition number (1-12): 1
New partition number (1-128, default 13): 52
Expert command (? for help): w
```

#### (optional) change boot parameter if you want
```bash
LOCATION_EFI_SYSTEM=$(lsblk -o NAME,PARTLABEL -r | grep $(basename $LOOP_LOCATION) | grep EFI-SYSTEM | cut -d" " -f1) # loopXpY
mkdir -p /tmp/mnt/EFI-SYSTEM; sudo mount /dev/$LOCATION_EFI_SYSTEM /tmp/mnt/EFI-SYSTEM
sudoedit /tmp/mnt/EFI-SYSTEM/efi/boot/grub.cfg
sudo umount /tmp/mnt/EFI-SYSTEM
```

#### unmount the loop device
```bash
sudo umount /tmp/mnt/*
sudo losetup -d $LOOP_LOCATION
```



---



## How to install Chromium OS for multiboot?

First, run gparted and create partitions labeled
- EFI-SYSTEM, vfat or ext4(?), 128MB
- ROOT-A, ext4, 5GB
- STATE, ext4, (size is as you like)
- ROOT-B, ext4, 5GB (If you want to do A/B update)

then, use `gdisk` to change partition numbers

```bash
# specify your internal storage, check with
# lsblk -o NAME,LABEL,PARTLABEL,SIZE,FSTYPE,MOUNTPOINT,PARTUUID
# in my case, it is nvme0n1
sudo gdisk /dev/nvme0n1
Command (? for help): x
# type "p" to print current partition numbers
Command (? for help): p
# change STATE to 52, in my case, it is currently 8
Expert command (? for help): t
Partition number (1-53): 8
New partition number (1-128, default 54): 52

# change another partitions as you like.
# on Surface Book, I personally use:
# EFI-SYSTEM on 50
# ROOT-A on 51
# ROOT-B on 53
# However, on some other devices (e.g. Surface 3), ROOT partition number can be used only up to `15`.
```

Mount your EFI-SYSTEM and ROOT-A of your internal storage and your built image
```bash
mkdir -p /tmp/mnt/{EFI-SYSTEM,ROOT-A,EFI-SYSTEM-internal,ROOT-A-internal}

# find partition location of image
export IMAGE_NAME=chromiumos_image-surface.bin
LOOP_LOCATION=$(losetup -f) # /dev/loopX
sudo losetup $LOOP_LOCATION -P $IMAGE_NAME
sleep 1 # for lsblk to recognize the changes
LOCATION_ROOT_A=$(lsblk -o NAME,PARTLABEL -r | grep $(basename $LOOP_LOCATION) | grep ROOT-A | cut -d" " -f1) # loopXpY
LOCATION_EFI_SYSTEM=$(lsblk -o NAME,PARTLABEL -r | grep $(basename $LOOP_LOCATION) | grep EFI-SYSTEM | cut -d" " -f1) # loopXpY

# mount image partitions
sudo mount /dev/$LOCATION_EFI_SYSTEM /tmp/mnt/EFI-SYSTEM
sudo mount /dev/$LOCATION_ROOT_A /tmp/mnt/ROOT-A

# mount internal disk partiton
sudo mount /dev/disk/by-label/EFI-SYSTEM /tmp/mnt/EFI-SYSTEM-internal
sudo mount /dev/disk/by-label/ROOT-A /tmp/mnt/ROOT-A-internal

# copy all the contents
sudo cp -a /tmp/mnt/EFI-SYSTEM/* /tmp/mnt/EFI-SYSTEM-internal/
sudo cp -a /tmp/mnt/ROOT-A/* /tmp/mnt/ROOT-A-internal/

# overwrite ROOT-A and ROOT-B PARTUUID of your internal storage on
# /tmp/mnt/EFI-SYSTEM-internal/efi/boot/grub.cfg
# check your PARTUUID by
# lsblk -o NAME,LABEL,PARTUUID
# root=PARTUUID=your-partuuid

# unmount all you mounted
sudo umount /tmp/mnt/*
sudo losetup -d $LOOP_LOCATION
```

Add boot entry to your bootloader. I personaly use rEFInd.
(Sorry I don't know about GRUB.)
Maybe your bootloeader will autodetect Chromium OS's bootloader depending on your bootloader setting, though.
```
menuentry "Chromium OS grub (manual entry)" {
    icon     /EFI/refind/icons/os_chrome.png
    volume   EFI-SYSTEM
    loader   /efi/boot/bootx64.efi
}
```
