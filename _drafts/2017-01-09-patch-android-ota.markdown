---
layout: post
title:  "Maintaining an Upgrade Path With Modified OTAs"
date:   2017-01-09 16:32:00 -0800
tags: android
---

If you've managed to [Securely Install a Patched Android Image][], the new challenge is then to  becomes upgrading it without having to factory reset every time. With Google now releasing monthly security updates, it becomes prohibitive to 

mkdir -p boot/{orig,modified}

# unpack a copy of the unmodified boot.img
pushd boot/orig/
unzip /data/nexus5x/download/bullhead-ota-nrd91n-888dc7ab.zip boot.img
abootimg -x boot.img 
popd


# make a modified boot.img with necessary changes in the initrd
pushd boot/modified
sudo abootimg-unpack-initrd ../orig/initrd.img 

cat <<END_PROPS | sudo tee -a ramdisk/default.prop
# ivan20160210
net.tethering.noprovisioning=true
END_PROPS

sudo abootimg-pack-initrd initrd.img ramdisk
abootimg --create boot-unsigned.img \
  -f ../orig/bootimg.cfg \
  -k ../orig/zImage \
  -r ./initrd.img 

/data/nexus5x/aosp/aosp-marshmallow/out/host/linux-x86/bin/boot_signer \
  /boot \
  boot-unsigned.img \
  /data/nexus5x/ivan-key/boot/boot-verity.pk8 \
  /data/nexus5x/ivan-key/boot/boot-verity.x509.pem \
  boot.img

popd



cp ../../download/bullhead-ota-nrd91n-888dc7ab.zip .

# update boot.img inside the zip. Use "-j" to "junk" the path so it won't put boot.img inside directories within the zip
# A warning here about CRC CD not matching is OK I guess. (Did not cause a problem 2016-11-10 nrd91n)
zip bullhead-ota-nrd91n-888dc7ab.zip -j boot/modified/boot.img 

# Sign the OTA zip with signapk.jar using personal OTA key
#
# -w means sign whole file
#
#   signapk has an extra 'sign whole file' mode, enabled with the -w option. When in this
#   mode, in addition to signing each individual JAR entry, the tool generates a
#   signature over the whole archive as well. This mode is not supported by jarsigner
#   and is specific to Android. So why sign the whole archive when each of the individual
#   files is already signed? In order to support over the air updates (OTA).
#     http://nelenkov.blogspot.com/2013/04/android-code-signing.html

java -jar /data/nexus5x/aosp/aosp-marshmallow/prebuilts/sdk/tools/lib/signapk.jar \
  -w \
  /data/nexus5x/ivan-key/ota/ota-key.x509.pem \
  /data/nexus5x/ivan-key/ota/ota-key.pk8 \
  ./bullhead-ota-nrd91n-888dc7ab.zip \
  ./signed.zip

adb reboot recovery
- power + vol up
- apply update from ADB
adb sideload signed.zip 



Aside: props precendence



* Description of android property system

* Property system implementation
  * http://rxwen.blogspot.com/2010/01/android-property-system.html
  * https://android.googlesource.com/platform/system/core/+/nougat-release/init/property_service.cpp

* List of prop files, and precedence:

These filenames are defined in [bionic/libc/include/sys/_system_properties.h][_system_properties.h]

```c++
#define PROP_PATH_RAMDISK_DEFAULT  "/default.prop"
#define PROP_PATH_SYSTEM_BUILD     "/system/build.prop"
#define PROP_PATH_VENDOR_BUILD     "/vendor/build.prop"
#define PROP_PATH_LOCAL_OVERRIDE   "/data/local.prop"
#define PROP_PATH_FACTORY          "/factory/factory.prop"
```

The system loads prop files in an order that matches how they become available as the system mounts different filesystems during boot.

1. *Load properties from the initrd.*\\
 Soon after the kernel has started the init process, but before any filesystems have been mounted, init's [main][main] calls [property_load_boot_defaults][property_load_boot_defaults].
  * `PROP_PATH_RAMDISK_DEFAULT` /default.prop

2. *Load properties from the system image.*\\
init continues to boot up and load [init.rc][init.rc], which is written in [Android Init Language][Android Init Language] and fulfills a function similar to SystemD's unit files.\\
  During [on late-init][on late-init], after mounting filesystems, eventually [load_system_props][load_system_props] is called.
  * `PROP_PATH_SYSTEM_BUILD` /system/build.prop
  * `PROP_PATH_VENDOR_BUILD` /vendor/build.prop
  * `PROP_PATH_FACTORY` /factory/factory.prop, only prefix matching "ro.*"

3. *Load properties from /data.*\\
The system continues to boot up and mounts /data, then calls [load_persist_props](https://android.googlesource.com/platform/system/core/+/nougat-release/init/property_service.cpp#476), which does a couple things:
  [load_override_properties](https://android.googlesource.com/platform/system/core/+/nougat-release/init/property_service.cpp#462) - if `ALLOW_LOCAL_PROP_OVERRIDE` (a [build-time makefile variable](https://android.googlesource.com/platform/system/core/+/nougat-release/init/Android.mk)) AND `ro.debuggable` are set, then
  * `PROP_PATH_LOCAL_OVERRIDE` "/data/local.prop"
  [load_persistent_properties](https://android.googlesource.com/platform/system/core/+/nougat-release/init/property_service.cpp#400) - looks for files in
  * `PERSISTENT_PROPERTY_DIR` "/data/property" starting with "persist."


* Modifying prop files
  * should be done as early as possible
  * it may be possible to set a property in default.prop and only have to modify the boot.img where the initrd is.
  * If properties are overridden later on, then you have to customize other files.
  * Certain files like /data/local.prop are only read if flags are set at build time, and "debugging" mode.


[load_system_props]: https://android.googlesource.com/platform/system/core/+/nougat-release/init/property_service.cpp#520
[_system_properties.h]: https://android.googlesource.com/platform/bionic/+/nougat-release/libc/include/sys/_system_properties.h#82
[main]: https://android.googlesource.com/platform/system/core/+/nougat-release/init/init.cpp#677
[property_load_boot_defaults]: https://android.googlesource.com/platform/system/core/+/nougat-release/init/property_service.cpp#458
[Android Init Language]:  https://android.googlesource.com/platform/system/core/+/nougat-release/init/readme.txt 
[on late-init]: https://android.googlesource.com/platform/system/core/+/nougat-release/rootdir/init.rc#250
[init.rc]: https://android.googlesource.com/platform/system/core/+/nougat-release/rootdir/init.rc

