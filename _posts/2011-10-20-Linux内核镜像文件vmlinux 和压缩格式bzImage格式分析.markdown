```text
组内搞培训，根据某牛人的blog，学习一下linux初始化过程。我都快要走人了，也给我分了一下任务，唉。



内核镜像文件格式解析
=================

1、 需要使用的命令
================

readelf    -- 显示elf格式文件信息 。  反汇编等
objdump    -- 显示object格式文件信息  例如反汇编
objcopy    -- 复制object文件段落，生成新的object文件。 可用于copy remove 一个object文件的某个段。
nm         -- 列出object 文件的符号信息

gcc      根据c文件，编译得到object文件
ld       连接 object文件，得到elf格式可执行文件



2. 内核镜像文件格式
==================
压缩形式的：
widebright@:~/桌面$ sudo file  /boot/vmlinuz-3.0.0-12-generic
[sudo] password for widebright: 
/boot/vmlinuz-3.0.0-12-generic: Linux kernel x86 boot executable bzImage, version 3.0.0-12-generic (buildd@vernad, RO-rootFS, root_dev 0x801, swap_dev 0x4, Normal VGA

这个是没有压缩形式的:
root@- linux-]# file vmlinux
vmlinux: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), statically linked, not stripped


可以看到内核镜像也是一个elf文件格式来的，编译出来的压缩和没有压缩文件大小：
--------------------------------------------
[root@- linux-]# cat arch/i386/boot/compressed/.head.o.cmd
[root@- linux-]# ls -l -h  arch/i386/boot/compressed/vmlinux
-rwxr-xr-x  1 root root 1.6M Sep  6 03:37 arch/i386/boot/compressed/vmlinux
[root@- linux-]# ls -lh vmlinux 
-rwxr-xr-x  1 root root 34M Sep  6 03:37 vmlinux
---------------------------------------------



3. 没有压缩的内核镜像vmlinux是怎么生成的
=====================================
编译内核源码，在源码根目录生成一个隐藏文件，里面保存了生成内核vmlinux的命令

[root@- linux-]# cat ./.vmlinux.cmd
cmd_vmlinux := ld -m elf_i386  -o vmlinux -T arch/i386/kernel/vmlinux.lds arch/i386/kernel/head.o arch/i386/kernel/init_task.o  init/built-in.o --start-group  usr/built-in.o  arch/i386/mach-generic/built-in.o  arch/i386/kernel/built-in.o  arch/i386/mm/built-in.o  arch/i386/mach-default/built-in.o  arch/i386/crypto/built-in.o  kernel/built-in.o  mm/built-in.o  fs/built-in.o  ipc/built-in.o  security/built-in.o  crypto/built-in.o  kdb/built-in.o  lib/lib.a  arch/i386/lib/lib.a  lib/built-in.o  arch/i386/lib/built-in.o  drivers/built-in.o  sound/built-in.o  arch/i386/pci/built-in.o  arch/i386/power/built-in.o  arch/i386/kdb/built-in.o  net/built-in.o --end-group .tmp_kallsyms3.o

可以看到，ld 把多个 object文件都链接起来了。
其中-T arch/i386/kernel/vmlinux.lds 参数指定了link script脚本配置文件。

------vmlinux.lds 文件部分节选-------
OUTPUT_FORMAT("elf32-i386", "elf32-i386", "elf32-i386")   #####elf格式的
OUTPUT_ARCH(i386)           ##########i386指令
ENTRY(startup_32)              #######连接后的入口点  
jiffies = jiffies_64;
SECTIONS
{
  . = (0xc0000000) + 0x100000;
  /* read-only */
  _text = .; /* Text and read-only data */
  .text : {
 *(.text)
 __sched_text_start = .; *(.sched.text) __sched_text_end = .;
 __lock_text_start = .; *(.spinlock.text) __lock_text_end = .;
 *(.fixup)
 *(.gnu.warning)
 } = 0x9090
  .entry.text : { *(.entry.text) }

 等等如果组装 各个elf的各个小节等，具体可以自己去看一下。
 关于脚本文件写法可以参考 The GNU linker 的文档 “Linker Scripts”
 比如  http://www.math.utah.edu/docs/info/ld_3.html#SE4
 




3. 压缩的内核镜像bzImage是怎么生成的
=================================


[root@- linux-]# cat arch/i386/boot/.bzImage.cmd 
cmd_arch/i386/boot/bzImage := arch/i386/boot/tools/build -b arch/i386/boot/bootsect arch/i386/boot/setup arch/i386/boot/vmlinux.bin CURRENT > arch/i386/boot/bzImage

可以知道 bzImage文件由arch/i386/boot/tools/build 工具，根据bootsect  setup vmlinux.bin 3个文件生成。


-------------------------------
[root@- linux-]# ls arch/i386/boot/tools/build
build    build.c  
[root@- linux-]# file  arch/i386/boot/tools/build
arch/i386/boot/tools/build: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), for GNU/Linux 2.2.5, dynamically linked (uses shared libs), not stripped


/*
 *  $Id: build.c,v 1.5 1997/05/19 12:29:58 mj Exp $
 *
 *  Copyright (C) 1991, 1992  Linus Torvalds
 *  Copyright (C) 1997 Martin Mares
 */

/*
 * This file builds a disk-image from three different files:
 *
 * - bootsect: exactly 512 bytes of 8086 machine code, loads the rest
 * - setup: 8086 machine code, sets up system parm
 * - system: 80386 code for actual system
 *
 * It does some checking that all files are of the correct type, and
 * just writes the result to stdout, removing headers and padding to
 * the right amount. It also writes some system data to stderr.
 */



        file_open(argv[1]);
        i = read(fd, buf, sizeof(buf));
        fprintf(stderr,"Boot sector %d bytes.\n",i);
        if (i != 512)
                die("Boot block must be exactly 512 bytes");
        if (buf[510] != 0x55 || buf[511] != 0xaa)          ///////////////这个启动标志？？
                die("Boot block hasn't got boot flag (0xAA55)");
        buf[508] = minor_root;
        buf[509] = major_root;
        if (write(1, buf, 512) != 512)
                die("Write call failed");
        close (fd);


        if (lseek(1, 497, SEEK_SET) != 497)                 /* Write sizes to the bootsector */
                die("Output: seek failed");
        buf[0] = setup_sectors;
        if (write(1, buf, 1) != 1)
                die("Write of setup sector count failed");
        if (lseek(1, 500, SEEK_SET) != 500)
                die("Output: seek failed");
        buf[0] = (sys_size & 0xff);
        buf[1] = ((sys_size >> 8) & 0xff);
        if (write(1, buf, 2) != 2)
                die("Write of image length failed");

        return 0;                                           /* Everything is OK */



--------------------------------

看看build.c中的代码，可以看到这个build工具其实还是很简单的，其实就是把3个文件内容直接往一个文件里面写而已。仔细看可以看到那些的具体的偏移，也做一些简单的文件格式检查。


其中setup部分来自这个命令，应该是根据arch/i386/boot/setup.S 生成的
----------------------
[root@- linux-]# cat arch/i386/boot/.setup.cmd 
cmd_arch/i386/boot/setup := ld -m elf_i386  -Ttext 0x0 -s --oformat binary -e begtext arch/i386/boot/setup.o -o arch/i386/boot/setup 
-----------------------------------



其中vmlinux.bin的真正的系统的部分来自：
-----------------------------------
[root@- linux-]# cat arch/i386/boot/.vmlinux.bin.cmd
cmd_arch/i386/boot/vmlinux.bin := objcopy -O binary -R .note -R .comment -S  arch/i386/boot/compressed/vmlinux arch/i386/boot/vmlinux.bin
------------------------------------

其实就是objcopy命令删掉 arch/i386/boot/compressed/vmlinux的文件的无用的.comment .note 段得到。



arch/i386/boot/compressed/vmlinux 这个压缩的代码块又是怎么弄出来的呢：
-----------------------------------
[root@- linux-]# file arch/i386/boot/compressed/vmlinux
arch/i386/boot/compressed/vmlinux: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), statically linked, not stripped

[root@- linux-]# cat arch/i386/boot/compressed/.vmlinux.cmd
cmd_arch/i386/boot/compressed/vmlinux := ld -m elf_i386  -Ttext 0x100000 -e startup_32 arch/i386/boot/compressed/head.o arch/i386/boot/compressed/misc.o arch/i386/boot/compressed/piggy.o -o arch/i386/boot/compressed/vmlinux 

       -e entry
       --entry=entry
           Use entry as the explicit symbol for beginning execution of your
           program, rather than the default entry point.  If there is no
           symbol named entry, the linker will try to parse entry as a number,
           and use that as the entry address (the number will be interpreted
           in base 10; you may use a leading 0x for base 16, or a leading 0
           for base 8).


可以看到这个其实是把head.o  /misc.o  piggy.o 3个文件链接得到。

-Ttext 0x100000 脚本指定了加载的位置。 
-e startup_32 表示启动点

--------------------
其中head.o  

[root@- linux-]# cat  arch/i386/boot/compressed/.head.o.cmd
cmd_arch/i386/boot/compressed/head.o := gcc -Wp,-MD,arch/i386/boot/compressed/.head.o.d -nostdinc -iwithprefix include -D__KERNEL__ -Iinclude  -D__ASSEMBLY__ -Iinclude/asm-i386/mach-generic -Iinclude/asm-i386/mach-default -m32 -traditional   -c -o arch/i386/boot/compressed/head.o arch/i386/boot/compressed/head.S

主要代码实在arch/i386/boot/compressed/head.S 汇编文件里面了。
----------------------
其中 misc.o 文件主要由编译 arch/i386/boot/compressed/misc.c 文件的来。

[root@- linux-]# cat  arch/i386/boot/compressed/.misc.o.cmd
cmd_arch/i386/boot/compressed/misc.o := gcc -Wp,-MD,arch/i386/boot/compressed/.misc.o.d -nostdinc -iwithprefix include -D__KERNEL__ -Iinclude  -Wall -Wstrict-prototypes -Wno-trigraphs -fno-strict-aliasing -fno-common -fno-delete-null-pointer-checks -Os -g -Wdeclaration-after-statement -pipe -msoft-float -m32 -fno-builtin-sprintf -fno-builtin-log2 -fno-builtin-puts  -mpreferred-stack-boundary=2 -fno-unit-at-a-time -march=i686 -fno-optimize-sibling-calls  -mregparm=3 -Iinclude/asm-i386/mach-generic -Iinclude/asm-i386/mach-default    -DKBUILD_BASENAME=misc -DKBUILD_MODNAME=misc -c -o arch/i386/boot/compressed/.tmp_misc.o arch/i386/boot/compressed/misc.c


[root@- linux-]# head -n 30 arch/i386/boot/compressed/misc.c 
/*
 * misc.c
 * 
 * This is a collection of several routines from gzip-1.0.3 
 * adapted for Linux.
 *
 * malloc by Hannu Savolainen 1993 and Matthias Urlichs 1994
 * puts by Nick Holloway 1993, better puts by Martin Mares 1995
 * High loaded stuff by Hans Lermen & Werner Almesberger, Feb. 1996
 */


大概看一下misc.c  可以知道，里面的代码是压缩算法的相关实现。
其中内核镜像解压的关键函数  decompress_kernel   就是在这个 misc.c 文件里面。

 326asmlinkage void decompress_kernel(void *rmode, memptr heap,
 327                                  unsigned char *input_data,
 328                                  unsigned long input_len,
 329                                  unsigned char *output)

------------------------------------
cat arch/i386/boot/compressed/.piggy.o.cmd
cmd_arch/i386/boot/compressed/piggy.o := ld -m elf_i386  -r --format binary --oformat elf32-i386 -T arch/i386/boot/compressed/vmlinux.scr arch/i386/boot/compressed/vmlinux.bin.gz -o arch/i386/boot/compressed/piggy.o 


       -r
       --relocatable
           Generate relocatable output  生成可重定位的
       
       -format=input-format
                  objdump -i 可以列出所有BFD 库所支持的格式。


根 据上面分析，可以得知其实这个piggy.o 其实是直接把 arch/i386/boot/compressed/vmlinux.bin.gz 这个gzip压缩文件作为一个二进制格式链接起来。关键是boot/compressed/vmlinux.scr 这个文件，出了指定二进制数据外还导出了input_len input_data 两个可重定位的变量。后面解压的代码根据这两个变量可以知道压缩的内核所在的地址和长度，然后能够正确解压内核。


--------cat boot/compressed/vmlinux.scr---- 
SECTIONS
{
  .data : {
        input_len = .;
        LONG(input_data_end - input_data) input_data = .;
        *(.data)
        input_data_end = .;
        }
}
~----------------------------------------
[root@- linux-]# readelf -a  arch/i386/boot/compressed/piggy.o
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           Intel 80386
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          1532244 (bytes into file)
  Flags:                             0x0
  Size of this header:               52 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           40 (bytes)
  Number of section headers:         5
  Section header string table index: 2

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .data             PROGBITS        00000000 000034 1760ff 00  WA  0   0  1    最终生成的.data段
  [ 2] .shstrtab         STRTAB          00000000 176133 000021 00      0   0  1
  [ 3] .symtab           SYMTAB          00000000 17621c 0000b0 10      4   5  4
  [ 4] .strtab           STRTAB          00000000 1762cc 0000c7 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)

There are no program headers in this file.

There are no relocations in this file.

There are no unwind sections in this file.

Symbol table '.symtab' contains 11 entries:
   Num:    Value  Size Type    Bind   Vis      Ndx Name
     0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 00000000     0 SECTION LOCAL  DEFAULT    1 
     2: 00000000     0 SECTION LOCAL  DEFAULT    2 
     3: 00000000     0 SECTION LOCAL  DEFAULT    3 
     4: 00000000     0 SECTION LOCAL  DEFAULT    4 
     5: 001760ff     0 NOTYPE  GLOBAL DEFAULT    1 _binary_arch_i386_boot_co
     6: 001760ff     0 NOTYPE  GLOBAL DEFAULT    1 input_data_end
     7: 00000004     0 NOTYPE  GLOBAL DEFAULT    1 _binary_arch_i386_boot_co
     8: 00000004     0 NOTYPE  GLOBAL DEFAULT    1 input_data                 导出变量，解压时使用
     9: 00000000     0 NOTYPE  GLOBAL DEFAULT    1 input_len                  导出变量，解压时使用
    10: 001760fb     0 NOTYPE  GLOBAL DEFAULT  ABS _binary_arch_i386_boot_co

No version information found in this file.

--------------------------------------------------------------------


下面尝试手工从piggy.o解出 vmlinux.bin.gz ，以证实上面的理解没错（不懂 linker script啊）


先把里面的.data段当作二进制格式复制出来。
[root@- linux-]# objcopy -j .data -O binary   arch/i386/boot/compressed/piggy.o mybzImage.gz

[root@- linux-]# ls -l arch/i386/boot/compressed/vmlinux.bin.gz
-rw-r--r--  1 root root 1532155 Sep  6 03:37 arch/i386/boot/compressed/vmlinux.bin.gz
[root@- linux-]# ls -l mybzImage.gz
-rw-r--r--  1 root root 1532159 Oct 20 00:41 mybzImage.gz

可以看到objcopy出来的数据比原有的vmlinux.bin.gz 多了4个字节。

[root@- linux-]# 
[root@- linux-]# hexdump -C -n 30 mybzImage.gz
00000000  fb 60 17 00 1f 8b 08 00  9d cd 65 4e 02 03 ec 59  |.`........eN...Y|
00000010  7d 74 14 55 96 af ea 8f  74 07 12 ab 71 12        |}t.U....t...q.|
0000001e
[root@- linux-]# hexdump -C -n 30 arch/i386/boot/compressed/vmlinux.bin.gz
00000000  1f 8b 08 00 9d cd 65 4e  02 03 ec 59 7d 74 14 55  |......eN...Y}t.U|
00000010  96 af ea 8f 74 07 12 ab  71 12 40 e5 23 e1        |....t...q.@.#.|
0000001e


(gdb) p /x  1532155
$3 = 0x1760fb


可以看到多出来的前面4个字节其实就是那个 input_len  来的。

[root@- linux-]# tail -c 1532155  mybzImage.gz  > mybzImage2.gz

[root@- linux-]# diff mybzImage2.gz arch/i386/boot/compressed/vmlinux.bin.gz
把前面的这个input_len去掉之后就得到原有的vmlinux.bin.gz了。

-----------------------------------------------
这个vmlinux.bin.gz 又是怎么来的呢？ 

[root@- linux-]# cat  arch/i386/boot/compressed/.vmlinux.bin.gz.cmd
cmd_arch/i386/boot/compressed/vmlinux.bin.gz := gzip -f -9 < arch/i386/boot/compressed/vmlinux.bin > arch/i386/boot/compressed/vmlinux.bin.gz

[root@- linux-]# cat arch/i386/boot/compressed/.vmlinux.bin.cmd
cmd_arch/i386/boot/compressed/vmlinux.bin := objcopy -O binary -R .note -R .comment -S  vmlinux arch/i386/boot/compressed/vmlinux.bin

可以看到其实就是使用gzip把 elf文件压缩而成的文件而已。
objcopy 把vmlinux 的无用信息去掉就得到一个新的elf格式文件 vmlinux.bin
-----------------------------------------------

所以整个bzImage其实就是由直接可以执行的head.S 和miss.c 的解压代码段组成，这段代码直接可以在boot loader加载镜像之后就执行。然后bzImage里面另外一部分压缩形式存在的就需要由这前面部分解压之后再能够执行了。



简单的看一下解压的代码。

---------------------------
arch/i386/boot/compressed/head.S

/*
 * Do the decompression, and jump to the new kernel..
 */
        subl $16,%esp   # place for structure on the stack
        movl %esp,%eax
        pushl %esi      # real mode pointer as second arg
        pushl %eax      # address of structure as first arg
        call decompress_kernel     #########################



arch/i386/boot/compressed/misc.c  

asmlinkage int decompress_kernel(struct moveparams *mv, void *rmode)
{
    real_mode = rmode;

    if (RM_SCREEN_INFO.orig_video_mode == 7) {
        vidmem = (char *) 0xb0000;
        vidport = 0x3b4;
    } else {
        vidmem = (char *) 0xb8000;
        vidport = 0x3d4;
    }

    lines = RM_SCREEN_INFO.orig_video_lines;
    cols = RM_SCREEN_INFO.orig_video_cols;

    if (free_mem_ptr < 0x100000) setup_normal_output_buffer();
    else setup_output_buffer_if_we_run_high(mv);

    makecrc();
    putstr("Uncompressing Linux... ");
    gunzip();                                 #################
    putstr("Ok, booting the kernel.\n");
    if (high_loaded) close_output_buffer_if_we_run_high(mv);
    return high_loaded;
}


#define get_byte()  (inptr < insize ? inbuf[inptr++] : fill_inbuf())


/* ===========================================================================
 * Fill the input buffer. This is called only when the buffer is empty
 * and at least one byte is really needed.
 */
static int fill_inbuf(void)
{
    if (insize != 0) {
        error("ran out of input data");
    }

    inbuf = input_data;     内核其实地址
    insize = input_len;     得到 内核代码压缩后的长度
    inptr = 1;
    return inbuf[0];
}


内核源码中的./lib/inflate.c

#define NEXTBYTE()  ({ int v = get_byte(); if (v < 0) goto underrun; (uch)v; })


/*
 * Do the uncompression!
 */
static int gunzip(void)
{
    uch flags;
    unsigned char magic[2]; /* magic header */
    char method;
    ulg orig_crc = 0;       /* original crc */
    ulg orig_len = 0;       /* original uncompressed length */
    int res;

    magic[0] = NEXTBYTE();        ###########
    magic[1] = NEXTBYTE();
    method   = NEXTBYTE();


   flags  = (uch)get_byte();       ###############33






nm arch/i386/boot/compressed/misc.o
0000c044 b bb
0000c048 b bk
00000000 r border
0000400c b bytes_out
00001cfd t close_output_buffer_if_we_run_high
0000c040 b cols
00000120 r cpdext
000000e0 r cpdist
00000060 r cplens
000000a0 r cplext
0000c460 b crc
0000c060 b crc_32_tab
00000188 r dbits
00001d36 T decompress_kernel
         U end
00001bec t error
00001abe t fill_inbuf
00001bd2 t flush_window
00001b5f t flush_window_high
00001afb t flush_window_low
00001972 t free
0000c028 b free_mem_end_ptr
00000000 d free_mem_ptr
0000139b t gunzip
00001977 t gzip_mark
00001984 t gzip_release
0000c034 b high_buffer_start
00004014 b high_loaded
00000000 t huft_build
00000566 t huft_free
0000c04c b hufts
00004018 b inbuf
0000128d t inflate
000011ab t inflate_block
00000582 t inflate_codes
00000bdb t inflate_dynamic
00000a92 t inflate_fixed
0000091c t inflate_stored
00004004 b inptr
         U input_data   这个链的是arch/i386/boot/compressed/piggy.o 导出的值。
         U input_len
00004000 b insize
00000184 r lbits
0000c03c b lines
0000c02c b low_buffer_end
0000c030 b low_buffer_size
0000131e t makecrc
0000191e t malloc
00000160 r mask_bits
00001a9f t memcpy
00001a88 t memset
00004008 b outcnt
0000c024 b output_data
00004010 b output_ptr
000001a0 r p.0
000019ea t putstr
0000c020 b real_mode
00001990 t scroll
00001c0f t setup_normal_output_buffer
00001c4e t setup_output_buffer_if_we_run_high
00000008 D stack_start
00000000 B user_stack
00000004 d vidmem
0000c038 b vidport
00004020 b window



[root@- linux-]# nm arch/i386/boot/compressed/piggy.o 
001760ff D _binary_arch_i386_boot_compressed_vmlinux_bin_gz_end
001760fb A _binary_arch_i386_boot_compressed_vmlinux_bin_gz_size
00000004 D _binary_arch_i386_boot_compressed_vmlinux_bin_gz_start
00000004 D input_data    
001760ff D input_data_end
00000000 D input_len    这里提供input_len变量的导出



最新版本的内核有点不一样
http://lxr.linux.no/#linux+v3.0.4/arch/x86/boot/compressed
可以看看这个目录下的Makefile文件和 mkpiggy.c  文件。
-----------------------------
head.S  的代码

 139/*
 140 * Do the decompression, and jump to the new kernel..
 141 */
 142        leal    z_extract_offset_negative(%ebx), %ebp
 143                                /* push arguments for decompress_kernel: */
 144        pushl   %ebp            /* output address */
 145        pushl   $z_input_len    /* input_len */
 146        leal    input_data(%ebx), %eax
 147        pushl   %eax            /* input_data */
 148        leal    boot_heap(%ebx), %eax
 149        pushl   %eax            /* heap area */
 150        pushl   %esi            /* real mode pointer */
 151        call    decompress_kernel
 152        addl    $20, %esp
 
 


misc.c 的解压函数

 326asmlinkage void decompress_kernel(void *rmode, memptr heap,
 327                                  unsigned char *input_data,
 328                                  unsigned long input_len,
 329                                  unsigned char *output)


=================================================
```
