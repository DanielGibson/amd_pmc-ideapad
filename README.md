## amd_pmc Kernel Module fixed for some IdeaPads and backported

This kernel module is a driver for the *AMD SoC Power Management Controller* and usually part of
the Linux kernel, under drivers/platform/x86/amd/pmc/

Taken from v7.0 and patched to work around a bug on some Lenovo laptops, mostly IdeaPad Slim 3
models with either Zen3 or Zen3+ CPU. That bug manifests as broken keyboard and lid switch
after suspend+resume.

See also https://lore.kernel.org/platform-driver-x86/20260501032655.283789-1-daniel@gibson.sh/T/
and https://bugzilla.kernel.org/show_bug.cgi?id=221383

I modified the source so it compiles with Linux Kernel 6.5 and newer.  
**Note:** I made sure it compiles with various kernel versions, but only tested the functionality
with 6.18 and 7.0. Please let me know if it doesn't build or work on other versions (>= 6.5).

### Building and installing

First you need to get this project on your computer, either by cloning it with git or by
downloading and extracting a source ZIP.

You'll need the headers for your currently running kernel installed and also a compiler and GNU make.

#### ... with DKMS

Add it to dkms (will copy the source to /usr/src/amd_pmc-0.0.1/, among other things):  
`$ sudo dkms add path/to/amd_pmc/`  
*(replace `path/to/amd_pmc/` with the path to the directory with the source)*

Build a module for your currently running kernel:  
`$ sudo dkms build amd_pmc/0.0.1`

Install the freshly built module:  
`$ sudo dkms install amd_pmc/0.0.1`

Now you could either just reboot or unload the old module and load the new one, like:  
`$ sudo rmmod amd-pmc`  
`$ sudo modprobe amd-pmc`

See [below](#making-sure-it-works) for how to make sure all this actually worked.

#### ... manually

Build it:  
```
$ make -C /lib/modules/`uname -r`/build M=$PWD
```

To test it without installing, you can do  
`$ sudo rmmod amd-pmc`  
to unload the currently loaded (old) amd_pmc module, then  
`$ sudo insmod ./amd-pmc.ko`  
to load the freshly built module.  
*(See [below](#making-sure-it-works) for how to make sure it works)*

To actually install this module for the current kernel, run  
```
$ sudo make -C /lib/modules/`uname -r`/build M=$PWD modules_install
```

### Making sure it works

The easiest check to see if the new module has been loaded is  
`ls /sys/module/amd_pmc/parameters/delay_suspend`  

If that file actually exists, the patched module is loaded.

If you have a Lenovo laptop and run  
`sudo dmidecode -s system-product-name`  
and it shows a name that either starts with "82X" (like "82XR") or "83K" (like "83K6"),
the workaround will be used automatically.

If you have another laptop and want to try if this workaround helps there as well, you can enforce its
usage with  
`$ echo 1 | sudo tee /sys/module/amd_pmc/parameters/delay_suspend`

Now to **test**, just suspend the laptop. Wait 10 seconds or so until it's really asleep, then wake
it up again. Try if the keyboard, incl. the Fn keys/multimedia keys still work.

You can check in dmesg if this workaround was used.  
There will be a line like  
`[   65.812742] amd_pmc AMDI0005:00: Delaying suspend by 2.5s to avoid platform bug`  
if your laptop was automatically detected as affected, or  
`[   65.812742] amd_pmc AMDI0005:00: Delaying suspend by 2.5s because delay_suspend=1`  
if you enforced using it by setting the module parameter.

If your laptop is affected and *not* automatically detected, please report that to me so I can add it.

To **permanently enforce using it** (on laptops not automatically detected as affected), add a file
`/etc/modprobe.d/my_amd_pmc.conf` with the following content:

```
# always enable the delay_suspend workaround when amd_pmc is loaded
options amd_pmc delay_suspend=1

```

### License

The original source is:

Copyright (c) 2020-2026, Advanced Micro Devices, Inc.

Licensed under GPLv2, most files GPLv2 or later.
See LICENSE for the GPLv2 text.
