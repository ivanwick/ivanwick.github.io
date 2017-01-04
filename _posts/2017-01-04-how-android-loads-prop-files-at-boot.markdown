---
layout: post
title:  "How Android Loads Prop Files At Boot"
date:   2017-01-04 12:42:00 -0800
categories: android
---
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

