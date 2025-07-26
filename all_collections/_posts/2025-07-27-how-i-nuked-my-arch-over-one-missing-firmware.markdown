---
layout: post
title: How I Nuked My Arch Over One Missing Firmware
date: 2025-07-27
categories: [Linux]
---

![[Pasted image 20250727014213.png]]

### The Story
Welp, it begins after I send my laptop (the right one in the image) to change its RAM since it gave up itself and considered upgrading it from 16GB to 32GB. After done servicing it, I rolled up my sleeves and begin the installation process. As I did install arch more than 4 times and so confidently install it without referring to [documentation](https://wiki.archlinux.org/title/Installation_guide). Yeah, the overconfidence just took over at that moment. Everything goes smooth as butter with [ml4w dotfiles](https://github.com/mylinuxforwork/dotfiles). 

Soon after, I opened Youtube and notice there is no audio. Hmmm.... I wonder why ???
Well, did I everything ???
Checklist:
- `pipewire`? Installed 
- `wireplumber ?` Yes is there
- `alsa-utils`? You bet 
- `systemctl`? Yes, still alive
Run again with root privileges, classic way to troubleshoot Linux 

Now headache begins, read documentation again (should have done this earlier), checking the config for audio, went to [reddit](https://www.reddit.com/r/archlinux/comments/pleb6c/audio_not_working_arch_linux/) and notice user **JkaSalt** recommended install `sof-firmware` and reboot it. 

![[Pasted image 20250727015934.png]]

I tried with `yay -S sof-firmware` and rebooting it still does not work.  Well, begin my research on how the firmware is loaded. 

### The Learning 
![[Pasted image 20250727021313.png]]
Once start the device, the BIOS loads and executes the MBR boot loader. Once MBR is in the memory, it will executes the GRUB boot loader from hard disk, typically at `/dev/sda` or `/dev/nvme0n1`. Once GRUB starts, it will load the kernel and initramfs image into memory space. Refer this [documentation](https://www.linuxfromscratch.org/blfs/view/svn/postlfs/initramfs.html) to learn more about initramfs, but I try my best to explain along with Claudia.

**Phase 1: Initramfs stage**
Kernel checks for the presence of the initramfs and mounts it as `/` and runs as `/init`. This process of unpacking happens before `do_basic_setup` is called. So, essentially what kernel does is two main things:
- Call `do_basic_setup`: core kernel function to initializes core subsystem right before launching user-space processes. Here is the code snippet of it:
```c
static void __init do_basic_setup(void) {
	cpuset_init_smp();
	usermodehelper_init();
	init_tmpfs();
	driver_init();
	init_irq_proc();
	do_ctors();
	do_initcalls();
}
```
- Call `init_post`: final function execution after kernel init sequence before jumping to user-space

**Phase 2: Kernel Driver Initialization**
When hardware drivers initialize, it call the kernel's firmware by loading API using function like `request_firmware()`.  Once API call returns, the driver has the firmware image accessible in `fw_entry -> {data,size}`. 

```
 kernel(driver): calls request_firmware(&fw_entry, $FIRMWARE, device)

 userspace:
        - /sys/class/firmware/xxx/{loading,data} appear.
        - hotplug gets called with a firmware identifier in $FIRMWARE
          and the usual hotplug environment.
                - hotplug: echo 1 > /sys/class/firmware/xxx/loading

 kernel: Discard any previous partial load.

 userspace:
                - hotplug: cat appropriate_firmware_image > \
                                        /sys/class/firmware/xxx/data

 kernel: grows a buffer in PAGE_SIZE increments to hold the image as it
         comes in.

 userspace:
                - hotplug: echo 0 > /sys/class/firmware/xxx/loading

 kernel: request_firmware() returns and the driver has the firmware
         image in fw_entry->{data,size}. If something went wrong
         request_firmware() returns non-zero and fw_entry is set to
         NULL.

 kernel(driver): Driver code calls release_firmware(fw_entry) releasing
                 the firmware image and any related resource.
```

The main API functions include:
- `request_firmware()` - synchronous loading
- `request_firmware_nowait()` - asynchronous loading
- `request_firmware_direct()` - direct filesystem lookup only

**Phase 3: Firmware Search Paths**
According to this [documentation](https://docs.kernel.org/driver-api/firmware/fw_search_path.html), it suggest the kernel looks for the firmware directly on the root filesystem. Here are the search paths:
- fw_path_para 
- /lib/firmware/updates/UTS_RELEASE/
- /lib/firmware/updates/
- /lib/firmware/UTS_RELEASE/
- /lib/firmware/

The filesystem lookup is implemented in fw_get_filesystem_firmware(), it uses common core kernel file loader facility kernel_read_file_from_path()

**Phase 4: Firmware Loading Mechanisms**
Linux uses different mechanism to load firmware based on hardware device. Here are some of it:
- Built-in Firmware: These files will then be accessible to the kernel at runtime. Firmware can be compiled directly into the kernel. 
- Direct Filesystem Lookup: Direct filesystem lookup is the most common form of firmware lookup performed by the kernel
- Fallback Mechanisms: Check for firmware on the filesystem, load from there if found. If kernel is configured for it  use a sysfs "fallback": create /sys/firmware/\<xxx>/loading, wait for some program to write firmware to it

Now, the question is why didn't `sof-firmware` not able to load into the kernel. Looking at [SOF documentation](https://thesofproject.github.io/latest/architectures/host/linux_driver/architecture/sof_driver_arch.html#sof-platform-driver), especially at the **Firmware Loading and Booting** part:
![[Pasted image 20250727030239.png]]

The key issue is the timing of `sof-firmware` availability. As SOF platform drivers loads very early in the boot process during kernel initialization. Even though I did installed after, the firmware files went to `/lib/firmware/` but not included in the **initramfs** which is the earlier boot filesystem. To recap overall process, here it how:
```
1. Kernel loads 
2. initramfs mounts (sof-firmware not available yet)
3. SOF platform driver initializes
4. Driver looks for firmware -> NOT FOUND
5. Audio hardware fails to initialize
6. After install: /lib/firmware/ mounts (bazinga, firmware is here, but too late)
```

### The Lesson
- Always RTFM :) 
![[Pasted image 20250727032409.png]]
- Run `mkinitcpio` after installing firmware packages WHEN installing arch 

And there is also my first time, totally forgot to allocate additional space for swap memory. 
