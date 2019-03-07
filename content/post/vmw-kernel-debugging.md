+++
date = "2019-03-07T16:33:12+02:00"
description = ""
draft = false
tags = []
title = "Debugging the Linux kernel with VMware"
topics = []

+++

I am playing with emulated HID devices in Linux and found a kernel bug when using the `usb_f_hid` and `dummy_hcd` kernel modules.
I won't go into details of what I am trying to achieve (saving this for a future post) but focus on how I troubleshooted this
particular bug. As of this writing, it is 100% reproducible with 4.15.0-45 kernel and these steps:

 1. Load `libcomposite` and `dummy_hcd` into the kernel
 2. Create an emulated HID device with configfs
 3. Write something to `/dev/hidg0`

After executing step 3, the machine hangs in a way which makes it clear that it's not a userspace problem but a kernel one.

I know that [VMware Workstation](https://www.vmware.com/products/workstation-pro.html) has support for kernel debugging, so 
I thought this is a great chance to give it a try.
I loaded Ubuntu 18.04 into a VM and executed the steps above. The VM hanged and the corresponding `vmware-vmx` process in the
host OS went to 100% CPU usage. This means that when the problem occurs, the kernel goes into some kind of busy loop.
Let's see how we can debug this further.

VM config
---
VMware Workstation has a nice feature which allows to debug the Linux kernel running inside the VM with `gdb` on the host.
This is enabled by adding a single line to the VM's configuration file:

```
debugStub.listen.guest64 = "TRUE"
```

Now when the VM is started, port 8864 is opened on the host and we can connect to it with `gdb` for remote debugging.

Preparing the guest OS
---
First, we want to disable [KASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization) 
as it will make the debugging harder. This is done by booting the kernel with the `nokaslr` option.

Open `/etc/default/grub`, find the line starting with `GRUB_CMDLINE_LINUX_DEFAULT` and append `nokaslr` at the end.
Then update GRUB:

`sudo update-grub`

and reboot the VM.

Next, we want to obtain debug symbols for the running kernel and its modules.
For Ubuntu there is a dedicated [repository](https://wiki.ubuntu.com/Debug%20Symbol%20Packages#Getting_-dbgsym.ddeb_packages):

```
echo "deb http://ddebs.ubuntu.com $(lsb_release -cs) main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-updates main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-proposed main restricted universe multiverse" | \
sudo tee -a /etc/apt/sources.list.d/ddebs.list
sudo apt install ubuntu-dbgsym-keyring
sudo apt-get update
```

Install kernel with debug symbols:

```
sudo apt-get install linux-image-`uname -r`-dbgsym
```

Copy the following files to the host OS as we need to load them into `gdb`:

```
/usr/lib/debug/boot/vmlinux-4.15.0-45-generic
/usr/lib/debug/lib/modules/4.15.0-45-generic/kernel/drivers/usb/gadget/udc/udc-core.ko
/usr/lib/debug/lib/modules/4.15.0-45-generic/kernel/drivers/usb/gadget/function/usb_f_hid.ko
dummy_hcd.ko
```
_Note_: `dummy_hcd` is missing in Ubuntu (see this [bug](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1073089)), I built it by myself.

Using GDB
---
Now that we have the debug symbols, we are ready to fire off `gdb` in the host OS:

```
$ gdb vmlinux-4.15.0-45-generic

(gdb) set architecture i386:x86-64
(gdb) target remote localhost:8864
(gdb) c
```
Hitting Ctrl+C will pause the VM:

[{{% fluid_img class="pure-u-1-1" src="/images/debug-vm-small.png" alt="debugvm" %}}](/images/debug-vm.png "debugvm")

so we can inspect the state in the debugger.

We also need to load the symbols for the kernel modules that we want to debug.
However, modules can be dynamically loaded at any address and we need to feed this information into `gdb`.
Let's take for example `usb_f_hid`. We can get the corresponding addresses in the guest OS like this:

```
$ cd /sys/module/usb_f_hid/sections
$ sudo cat .text .data .bss 
0xffffffffc06d7000
0xffffffffc06da000
0xffffffffc06da740
```

Now we can add the symbol file in `gdb` using the addresses from above:

```
(gdb) add-symbol-file usb_f_hid.ko 0xffffffffc06d7000 -s .data 0xffffffffc06da000 -s .bss 0xffffffffc06da740
```

We do the same for `udc-core` and `dummy_hcd`.
Now that we have all symbols loaded, we can trigger the bug in the guest OS and then hit Ctrl+C in `gdb` to
inspect the backtrace:

```
^C
Program received signal SIGINT, Interrupt.
0xffffffff810de405 in rep_nop () at /build/linux-uQJ2um/linux-4.15.0/arch/x86/include/asm/processor.h:647
647 /build/linux-uQJ2um/linux-4.15.0/arch/x86/include/asm/processor.h: No such file or directory.
(gdb) bt
#0  0xffffffff810de405 in rep_nop () at /build/linux-uQJ2um/linux-4.15.0/arch/x86/include/asm/processor.h:647
#1  cpu_relax () at /build/linux-uQJ2um/linux-4.15.0/arch/x86/include/asm/processor.h:652
#2  virt_spin_lock (lock=<optimized out>) at /build/linux-uQJ2um/linux-4.15.0/arch/x86/include/asm/qspinlock.h:69
#3  native_queued_spin_lock_slowpath (lock=0xffff88007196984c, val=1) at /build/linux-uQJ2um/linux-4.15.0/kernel/locking/qspinlock.c:305
#4  0xffffffff8199f407 in pv_queued_spin_lock_slowpath (val=<optimized out>, lock=<optimized out>) at /build/linux-uQJ2um/linux-4.15.0/arch/x86/include/asm/paravirt.h:669
#5  queued_spin_lock_slowpath (val=<optimized out>, lock=<optimized out>) at /build/linux-uQJ2um/linux-4.15.0/arch/x86/include/asm/qspinlock.h:30
#6  queued_spin_lock (lock=<optimized out>) at /build/linux-uQJ2um/linux-4.15.0/include/asm-generic/qspinlock.h:90
#7  do_raw_spin_lock_flags (flags=<optimized out>, lock=<optimized out>) at /build/linux-uQJ2um/linux-4.15.0/include/linux/spinlock.h:172
#8  __raw_spin_lock_irqsave (lock=<optimized out>) at /build/linux-uQJ2um/linux-4.15.0/include/linux/spinlock_api_smp.h:119
#9  _raw_spin_lock_irqsave (lock=0xffff88007196984c) at /build/linux-uQJ2um/linux-4.15.0/kernel/locking/spinlock.c:152
#10 0xffffffffc06d7410 in f_hidg_req_complete (ep=<optimized out>, req=<optimized out>) at /build/linux-uQJ2um/linux-4.15.0/drivers/usb/gadget/function/f_hid.c:328
#11 0xffffffffc06a390a in usb_gadget_giveback_request ()
#12 0xffffffffc06cdff2 in dummy_queue ()
#13 0xffffffffc06a2b96 in usb_ep_queue ()
#14 0xffffffffc06d7eb6 in f_hidg_write (file=<optimized out>, buffer=<optimized out>, count=5, offp=<optimized out>) at /build/linux-uQJ2um/linux-4.15.0/drivers/usb/gadget/function/f_hid.c:394
#15 0xffffffff8127730b in __vfs_write (file=<optimized out>, p=<optimized out>, count=<optimized out>, pos=<optimized out>) at /build/linux-uQJ2um/linux-4.15.0/fs/read_write.c:481
#16 0xffffffff812774d1 in vfs_write (file=0xffff880077b77c00, buf=0x55841858c7c0 "fira\n", count=<optimized out>, pos=0xffffc90000bbbef8) at /build/linux-uQJ2um/linux-4.15.0/fs/read_write.c:569
#17 0xffffffff81277725 in SYSC_write (count=<optimized out>, buf=<optimized out>, fd=<optimized out>) at /build/linux-uQJ2um/linux-4.15.0/fs/read_write.c:615
#18 SyS_write (fd=<optimized out>, buf=94025832515520, count=5) at /build/linux-uQJ2um/linux-4.15.0/fs/read_write.c:607
#19 0xffffffff81003ae3 in do_syscall_64 ()
#20 0xffffffff81a00081 in entry_SYSCALL_64 () at /build/linux-uQJ2um/linux-4.15.0/arch/x86/entry/entry_64.S:237
#21 0x00007f8e8d85d760 in ?? ()
#22 0x00007f8e8d85e2a0 in ?? ()
#23 0x0000000000000005 in irq_stack_union ()
Backtrace stopped: previous frame inner to this frame (corrupt stack?)
```

This reveals a deadlock involving `hidg->write_spinlock` in `f_hid.c` which explains why the CPU goes to 100% when the bug occurs.
The spinlock is acquired in the `f_hidg_write()` function before calling `usb_ep_queue()` which callbacks to `f_hidg_req_complete()` which
tries to acquire the same spinlock again.

My attempt to fix this bug is this [patch](https://www.spinics.net/lists/linux-usb/msg177735.html) which I submitted to the maintainers of the USB subsytem.
Let's see how that goes :)
