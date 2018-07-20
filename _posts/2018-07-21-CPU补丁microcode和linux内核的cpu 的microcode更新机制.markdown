今天在linux的启动的内核日志里面看到一条下面这样的日志，
```text
[Firmware Bug]: TSC_DEADLINE disabled due to Errata; please update microcode to version: 0x20 (or later)
```
好像是CPU的microcode版本太低， 影响“APIC timer”的初始化。所以就花了点时间看看这个 Microcode。


# CPU的微代码Microcode
原来CPU还是可以动态更新一下固件的microcode的。像Intel就不停的为他的CPU发布最新
的 Microcode， 来修复CPU的bug。比如最近很火的CPU安全漏洞的话题“meltdown spectre”。
这个CPU设计的漏洞导致严重的安全问题，Intel都为这个发了好几次Microcode的更新，Linux
内核也为这两个问题修改了很多设计，应该是加了几个相关的内核启动参数。

总之这个“meltdown spectre”问题的修复，不但要刷新bios（就是为了更新这个Microcode补丁吧），
而且要更新linux操作系统的内核才行。


# linux上面查看当前的cpu 的 microcode 版本
```text
dmesg | grep microcode
grep microcode /proc/cpuinfo
```

# /Linux-Processor-Microcode-Data-File
https://downloadcenter.intel.com/download/27945/Linux-Processor-Microcode-Data-File?product=80800

这个就是Intel为各种cpu型号发布的最新的microcode了，专门用于linux平台的。
它的文档大概提到了如何更新CPU的microcode，这个cpu的microcode每次断电都会被重置
的，所以每次重启后都需要重新更新。这个最好还是在BIOS固件里面更新是最好了，后面
操作系统就不需要更新了吧，特别是那种影响系统初始化的microcode的bug。

```text
Intel Processor Microcode Package for Linux
CPU microcode is a mechanism to correct certain errata in existing systems.

The normal preferred method to apply microcode updates is using the system

BIOS, but for a subset of Intel's processors this can be done at runtime 

using the operating system. This package contains those processors that 

support OS loading of microcode updates.

The target users for this package are OS vendors such as Linux distributions

for inclusion in their OS releases. Intel recommends getting the microcode

using the OS vendor update mechanism. Expert users can of course update their

microcode directly outside the OS vendor mechanism. This method is complex and

thus could be error prone.

Microcode is best loaded from the BIOS. Certain microcode must only be applied

from the BIOS. Such processor microcode updates are never packaged in this

package since they are not appropriate for OS distribution. An OEM may receive

microcode packages that might be a superset of what is contained in this

package.

OS vendors may choose to also update microcode that kernel can consume for early

loading. For example, Linux can update processor microcode very early in the kernel 

boot sequence. In situations when the BIOS update isn't available, early loading

is the next best alternative to updating processor microcode. Microcode states

are reset on a power reset; hence, it is required to be updated every time during the 

boot process.

Loading microcode using the initrd method is recommended so that the microcode 

is loaded at the earliest time for best coverage. Systems that cannot tolerate 

downtime may use the late reload method to update a running system without a

reboot.

== About Processor Signature, Family, Model, Stepping and Platform ID ==

Processor signature is a number identifying the model and version of a

Intel processor. It can be obtained using the CPUID instruction, and can

also be obtained via the command lscpu or from the content of /proc/cpuinfo.

It's usually presented as 3 fields: Family, Model and Stepping.

(In the table of updates below, they are shorten as F, MO and S.)

The width of Family/Model/Stepping is 12/8/4bit, but when arranged in the

32bit processor signature raw data is like 0FFM0FMS, hexadecimal.

e.g. if a processor signature is 0x000906eb, it means

Family=0x006, Model=0x9e and Stepping=0xb

A processor product can be implemented for multiple types of platforms,

So in MSR(17H), Intel processors have a 3bit Platform ID field,

that can specify a platform type from at most 8 types.

A microcode file for a specified processor model can support multiple

platforms, so the Platform ID of a microcode (shorten as PI in the table)

is a 8bit mask, each set bit indicates a platform type that it supports.

One can find the platform ID on Linux using rdmsr from msr-tools.

== Microcode update instructions ==

-- intel-ucode/ --

intel-ucode directory contains binary microcode files named in

family-model-stepping pattern. The file is supported in most modern Linux

distributions. It's generally located in the /lib/firmware directory,

and can be updated through the microcode reload interface.

To update early loading initrd, consult your distribution on how to package

microcode files for early loading. Some distros use update-initramfs or dracut.

As recommended above, please use the OS vendors are recommended method to ensure 

microcode file is updated for early loading before attempting the late-load 

procedure below.

To update the intel-ucode package to the system, one need:

1. Ensure the existence of /sys/devices/system/cpu/microcode/reload

2. Copy intel-ucode directory to /lib/firmware, overwrite the files in

/lib/firmware/intel-ucode/

3. Write the reload interface to 1 to reload the microcode files, e.g.

  echo 1 > /sys/devices/system/cpu/microcode/reload

If you are using the OS vendor method to update microcode, the above steps may

have been done automatically during the update process.

-- intel-ucode-with-caveats/ --

This directory holds microcode that might need special handling.

BDX-ML microcode is provided in directory, because it need special commits in

the Linux kernel, otherwise, updating it might result in unexpected system

behavior. 

OS vendors must ensure that the late loader patches (provided in

linux-kernel-patches\) are included in the distribution before packaging the

BDX-ML microcode for late-loading.

```

# microcode_ctl RPM

这个CPU的microcode的更新最好还是在BIOS里面的，如果BIOS没有更新才会考虑在Linux里面
来更新。 在Linux系统里面更新又分为 “early 更新” 和 “后更新”， 前者在linux内核初始化
前就开始更新吧，后者在系统启动完成之后可以随时通过
“echo 1 > /sys/devices/system/cpu/microcode/reload” 来加载 /lib/firmware/intel-ucode
目录下的microcode来更新。

这个肯定是越早更新越好了， Redhat 或者 Centos是通过 “microcode_ctl” 这个rpm包来
做这个事情的，这个包不单单提供 /lib/firmware/intel-ucode 目录下的那些microcode文件，
而且配置dracut的规则，告诉系统怎么创建“early cpio” 这种包含辅助更新microcode的initramfs包。

ubuntu 上面应该也有类似的包，好像是就叫做 “intel-ucode” ，可以通过apt 命令来安装的吧。


# early_cpio
这种cpio包应该是多个cpio包组合自一起形成的（后面有说）， 由dracut辅助工具生成的。
microcode_ctl 配置了dracut规则， dracut生成initramfs的image的时候就会生成这种组合的cpio了。

查看这种组合的cpio可以使用dracut的lsinitrd命令，比如列出某个组合cpio包里面的所有文件：
```text
[root@localhost ]# lsinitrd  /boot/initramfs-3.10.0-862.3.2.el7.x86_64.img
Image: /boot/initramfs-3.10.0-862.3.2.el7.x86_64.img: 20M
========================================================================
Early CPIO image
========================================================================
drwxr-xr-x   3 root     root            0 Jun  1 20:37 .
-rw-r--r--   1 root     root            2 Jun  1 20:37 early_cpio
drwxr-xr-x   3 root     root            0 Jun  1 20:37 kernel
drwxr-xr-x   3 root     root            0 Jun  1 20:37 kernel/x86
drwxr-xr-x   2 root     root            0 Jun  1 20:37 kernel/x86/microcode
-rw-r--r--   1 root     root        99328 Jun  1 20:37 kernel/x86/microcode/GenuineIntel.bin
========================================================================
Version: dracut-033-535.el7
```

/usr/bin/lsinitrd  是一个shel脚本，可以看看他是怎么解压查看initramfs里面的内容的。
有使用了dracut包里面的skipcpio工具

# 手工解压这种组合式的initramfs的镜像
必须循环的读取cpio文件, 直到文件结尾才行
```text
/usr/lib/dracut/skipcpio initramfs-3.10.0-862.3.2.el7.x86_64.img | gunzip -c | cpio -idv
```

# dracut
https://dracut.wiki.kernel.org/index.php/Main_Page

这个就是CentOS专门用来创建系统的initramfs的辅助工具了， mkinitrd工具现在也就是dracut的包装了。

这个dracut还是有点意思的，通过脚本来告诉dracut怎么生成的最终的initramfs，要把那些文件或者内核模块保护进去等等。
可以看看系统的规则比如 /usr/lib/dracut/modules.d/05busybox/module-setup.sh 等，还有很多其他模块，可以看看。
不过感觉有点复杂，创建好了用起来倒是很方便。

比如告诉他创建那种early microcode的特殊的cpio的系统启动景象应该是用这个参数吧。
```text
 man dracut

       --early-microcode
           Combine early microcode with ramdisk
```


# The Linux Microcode Loader
https://elixir.bootlin.com/linux/latest/source/Documentation/x86/microcode.txt
找了一下，终于看到linux的内核文档，说linux的microcode的更新机制是怎么实现的了。

```text
	The Linux Microcode Loader

Authors: Fenghua Yu <fenghua.yu@intel.com>
	 Borislav Petkov <bp@suse.de>

The kernel has a x86 microcode loading facility which is supposed to
provide microcode loading methods in the OS. Potential use cases are
updating the microcode on platforms beyond the OEM End-Of-Life support,
and updating the microcode on long-running systems without rebooting.

The loader supports three loading methods:

1. Early load microcode
=======================

The kernel can update microcode very early during boot. Loading
microcode early can fix CPU issues before they are observed during
kernel boot time.

The microcode is stored in an initrd file. During boot, it is read from
it and loaded into the CPU cores.

The format of the combined initrd image is microcode in (uncompressed)
cpio format followed by the (possibly compressed) initrd image. The
loader parses the combined initrd image during boot.

The microcode files in cpio name space are:

on Intel: kernel/x86/microcode/GenuineIntel.bin
on AMD  : kernel/x86/microcode/AuthenticAMD.bin

During BSP (BootStrapping Processor) boot (pre-SMP), the kernel
scans the microcode file in the initrd. If microcode matching the
CPU is found, it will be applied in the BSP and later on in all APs
(Application Processors).

The loader also saves the matching microcode for the CPU in memory.
Thus, the cached microcode patch is applied when CPUs resume from a
sleep state.

Here's a crude example how to prepare an initrd with microcode (this is
normally done automatically by the distribution, when recreating the
initrd, so you don't really have to do it yourself. It is documented
here for future reference only).

---
  #!/bin/bash

  if [ -z "$1" ]; then
      echo "You need to supply an initrd file"
      exit 1
  fi

  INITRD="$1"

  DSTDIR=kernel/x86/microcode
  TMPDIR=/tmp/initrd

  rm -rf $TMPDIR

  mkdir $TMPDIR
  cd $TMPDIR
  mkdir -p $DSTDIR

  if [ -d /lib/firmware/amd-ucode ]; then
          cat /lib/firmware/amd-ucode/microcode_amd*.bin > $DSTDIR/AuthenticAMD.bin
  fi

  if [ -d /lib/firmware/intel-ucode ]; then
          cat /lib/firmware/intel-ucode/* > $DSTDIR/GenuineIntel.bin
  fi

  find . | cpio -o -H newc >../ucode.cpio
  cd ..
  mv $INITRD $INITRD.orig
  cat ucode.cpio $INITRD.orig > $INITRD

  rm -rf $TMPDIR
---

The system needs to have the microcode packages installed into
/lib/firmware or you need to fixup the paths above if yours are
somewhere else and/or you've downloaded them directly from the processor
vendor's site.

2. Late loading
===============

There are two legacy user space interfaces to load microcode, either through
/dev/cpu/microcode or through /sys/devices/system/cpu/microcode/reload file
in sysfs.

The /dev/cpu/microcode method is deprecated because it needs a special
userspace tool for that.

The easier method is simply installing the microcode packages your distro
supplies and running:

# echo 1 > /sys/devices/system/cpu/microcode/reload

as root.

The loading mechanism looks for microcode blobs in
/lib/firmware/{intel-ucode,amd-ucode}. The default distro installation
packages already put them there.

3. Builtin microcode
====================

The loader supports also loading of a builtin microcode supplied through
the regular builtin firmware method CONFIG_EXTRA_FIRMWARE. Only 64-bit is
currently supported.

Here's an example:

CONFIG_EXTRA_FIRMWARE="intel-ucode/06-3a-09 amd-ucode/microcode_amd_fam15h.bin"
CONFIG_EXTRA_FIRMWARE_DIR="/lib/firmware"

This basically means, you have the following tree structure locally:

/lib/firmware/
|-- amd-ucode
...
|   |-- microcode_amd_fam15h.bin
...
|-- intel-ucode
...
|   |-- 06-3a-09
...

so that the build system can find those files and integrate them into
the final kernel image. The early loader finds them and applies them.

Needless to say, this method is not the most flexible one because it
requires rebuilding the kernel each time updated microcode from the CPU
vendor is available.
```

这个文档就比较清楚了， 还告诉你怎么手工创建这种组合的用于更新microcode的initramfs包。

