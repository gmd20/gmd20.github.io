```text
Crash 工具主页
http://people.redhat.com/anderson/

（1） “ crash + dump 文件 + 调试版本内核“命令，启动crash
例如：
crash vmcore vmlinux-2.6.18-92.EL.fulldebug

注意这个内核应该和你dump的机器的内核相对应，这里还可以指定 system.map的符号文件等。


（2） bt 命令查看调用堆栈，估计的最有效的一个命令的了。

bt + pid 列出相应的进程堆栈

bt -f 会列出所有堆栈里面数据  

比如

crash> bt -f 1854
PID: 1854   TASK: ffff810834d770c0  CPU: 3   COMMAND: "oracle"
 #0 [ffff81011d6bbbb0] crash_kexec at ffffffff800aa977
    ffff81011d6bbbb8: ffff81085d100000 ffff81085d0f0000 
    ffff81011d6bbbc8: 0000000000000004 0000000000000004 
    ffff81011d6bbbd8: 0000000000000040 ffffffff881922dd 
    ffff81011d6bbbe8: ffff81085d838000 0000000000000009 
    ffff81011d6bbbf8: ffff81085e6a11c0 0000000000000000 
    ffff81011d6bbc08: 0000000000000000 0000000000000000 
    ffff81011d6bbc18: 00000000000003f9 0000000000000000 
    ffff81011d6bbc28: 0000000000000000 ffffffff80198c24 
    ffff81011d6bbc38: ffffffff800aa956 0000000000000010 
    ffff81011d6bbc48: 0000000000000046 ffff81011d6bbbb8 
    ffff81011d6bbc58: 0000000000000018 ffffffff80080605 
    ffff81011d6bbc68: ffffffff881922dd ffffffff8008f44e 
 #1 [ffff81011d6bbc70] panic at ffffffff8008f44e
    ffff81011d6bbc78: 0000003000000020 ffff81011d6bbd68 
    ffff81011d6bbc88: ffff81011d6bbc98 ffff81011e82eac0 
    ffff81011d6bbc98: 0000000000000007 ffff81085d1b3350 
    ffff81011d6bbca8: 00000000000040c8 0000000000004400 
    ffff81011d6bbcb8: ffff8104ab8e1e80 0000000000000004 
    ffff81011d6bbcc8: 0000000000000000 ffff81011e82eac0 
    ffff81011d6bbcd8: 0000000000000002 0000000000000000 
    ffff81011d6bbce8: 0000000000000004 ffffffff8818e317 
    ffff81011d6bbcf8: ffff8100351e9380 ffff810000000002 
    ffff81011d6bbd08: ffff81085d1b3350 0000022000000046 
    ffff81011d6bbd18: ffff81011e87ae60 0000000000004400 
    ffff81011d6bbd28: ffff81011dc6f790 ffff81011dc6f790 
    ffff81011d6bbd38: 0000000000000000 ffff81011dc6f790 
    ffff81011d6bbd48: ffff81011dc6f790 ffff8100351e9380 
    ffff81011d6bbd58: 0000000000000040 ffffffff8818e734 
 #2 [ffff81011d6bbd60] myfunction1 at ffffffff8818e734    ///ffff81011d6bbd60 这里是call指令压入的返回地址
    ffff81011d6bbd68: ffff81085d1b3350 ffff81085d322d00 
    ffff81011d6bbd78: ffff81085d1b3350 00000000000020c8 
    ffff81011d6bbd88: ffff81085d099400 ffffffff88186c4e   //ffff81011d6bbd88 是函数栈的起始位置
 #3 [ffff81011d6bbd90] myfunction2 at ffffffff88186c4e    // 其他栈可以分析，可以结合函数汇编，看看push指令和rsp等寄存器的修改等。
    ffff81011d6bbd98: 0000000000000000 0000000000000000 
    ffff81011d6bbda8: ffff81085d0f0000 ffff81085d099400 
    ffff81011d6bbdb8: 0000000000000000 ffffffff88187845 
 #4 [ffff81011d6bbdc0] myfunction3 ffffffff88187845


x86_64函数参数，大多都通过寄存器传递，寄存器的使用顺序都是固定的，可以自己写个简单函数测试一下，我以前一篇文章也有提到。局部变量一般通过 
mov 0x20（%rsp), rax  这样的居于rsp寄存器栈顶偏移来定位。




（3） mod 命令用来加载调试符号，有时一些结构或者函数的符号信息不在调试版本内核里面，需要用gcc的 -g选项编译自己的模块，然后用mod命令加载里面的调试信息。这样 sym 和whatis 命令就能正确解释我们自己模块里面自定义的结构等信息。



crash> mod
     MODULE       NAME                   SIZE  OBJECT FILE
ffffffff88009e80  ehci_hcd              65741  (not loaded)  [CONFIG_KALLSYMS]
ffffffff88019280  ohci_hcd              54493  (not loaded)  [CONFIG_KALLSYMS]
ffffffff88028e00  uhci_hcd              57433  (not loaded)  [CONFIG_KALLSYMS]
ffffffff88041c00  jbd                   93873  (not loaded)  [CONFIG_KALLSYMS]
ffffffff8806bb00  ext3                 167249  (not loaded)  [CONFIG_KALLSYMS]
ffffffff880a8c00  scsi_mod             188665  (not loaded)  [CONFIG_KALLSYMS]
ffffffff880b8980  sd_mod                56257  (not loaded)  [CONFIG_KALLSYMS]
ffffffff881bab80  pcspkr                36289  (not loaded)  [CONFIG_KALLSYMS]
ffffffff881ca780  edac_mc               60193  (not loaded)  [CONFIG_KALLSYMS]
ffffffff881dd200  shpchp                70765  (not loaded)  [CONFIG_KALLSYMS]
ffffffff881eac00  serio_raw             40517  (not loaded)  [CONFIG_KALLSYMS]
ffffffff881f6280  i5000_edac            42177  (not loaded)  [CONFIG_KALLSYMS]
ffffffff88208f00  sg                    69993  (not loaded)  [CONFIG_KALLSYMS]
ffffffff8821b600  cdrom                 68713  (not loaded)  [CONFIG_KALLSYMS]
ffffffff88229400  sr_mod                50789  (not loaded)  [CONFIG_KALLSYMS]
ffffffff8823cb00  parport               73165  (not loaded)  [CONFIG_KALLSYMS]
ffffffff88249080  lp                    47121  (not loaded)  [CONFIG_KALLSYMS]
ffffffff8825a000  parport_pc            62313  (not loaded)  [CONFIG_KALLSYMS]
ffffffff88265500  ac                    38729  (not loaded)  [CONFIG_KALLSYMS]
ffffffff88270a80  acpi_memhotplug       40133  (not loaded)  [CONFIG_KALLSYMS]
ffffffff8827e480  asus_acpi             50917  (not loaded)  [CONFIG_KALLSYMS]
ffffffff8828a900  battery               43849  (not loaded)  [CONFIG_KALLSYMS]
ffffffff88295c00  button                40545  (not loaded)  [CONFIG_KALLSYMS]
ffffffff882a4900  i2c_core              56129  (not loaded)  [CONFIG_KALLSYMS]
ffffffff882af480  i2c_ec                38593  (not loaded)  [CONFIG_KALLSYMS]
ffffffff882ba980  backlight             39873  (not loaded)  [CONFIG_KALLSYMS]
ffffffff882c8080  sbs                   49921  (not loaded)  [CONFIG_KALLSYMS]
ffffffff882d6d80  video                 53197  (not loaded)  [CONFIG_KALLSYMS]
ffffffff882efa80  dm_mod                99737  (not loaded)  [CONFIG_KALLSYMS]
ffffffff882fec80  dm_multipath          52945  (not loaded)  [CONFIG_KALLSYMS]
ffffffff8830ea80  dm_mirror             60617  (not loaded)  [CONFIG_KALLSYMS]
ffffffff8831dd80  autofs4               57289  (not loaded)  [CONFIG_KALLSYMS]
ffffffff88329280  crypto_api            42177  (not loaded)  [CONFIG_KALLSYMS]
ffffffff88335900  xfrm_nalgo            43845  (not loaded)  [CONFIG_KALLSYMS]
ffffffff88340000  hangcheck_timer       37465  (not loaded)  [CONFIG_KALLSYMS]
crash> 
crash> 
crash> mod -d test_module
crash> mod -s test_module  test_module.ko
     MODULE       NAME                   SIZE  OBJECT FILE
ffffffff88198180  test_module         165128  test_module.ko 




（4) struct 命令把某个地址当成一个结构解析出来，很好用的一个命令，当然前提是你的结构的symbol符号信息已经用mod命令加载或者已经存在。
还有你要知道结构的地址 ^_^ 这个可以通过panic的报告还有分析函数堆栈得到。类似的命令还有p等

crash> struct skb_shared_info  ffff8104ab8e1e80
struct skb_shared_info {
  dataref = {
    counter = 1
  }, 
  nr_frags = 4, 
  gso_size = 0, 
  gso_segs = 0, 
  gso_type = 0, 
  ip6_frag_id = 0, 
  frag_list = 0x0, 
  frags = {{
      page = 0xffff810103eb5a00, 
      page_offset = 0, 
      size = 4096
    }, {
      page = 0xffff810103eb5a38, 
      page_offset = 0, 
      size = 616
    }, {
      page = 0xffff81011b9daca8, 
      page_offset = 0, 
      size = 4096
    }, {


http://people.redhat.com/anderson/crash_whitepaper/help_pages/mod.html
的说明
NAME
  mod - module information and loading of symbols and debugging data

SYNOPSIS
  mod [ -s module [objfile] | -d module | -S [directory] | -D | -r | -o ] 

DESCRIPTION
  With no arguments, this command displays basic information of the currently
  installed modules, consisting of the module address, name, size, the
  object file name (if known), and whether the module was compiled with
  CONFIG_KALLSYMS.
 
  The arguments are concerned with with the loading or deleting of symbolic
  and debugging data from a module's object file.  A modules's object file
  always contains symbolic data (symbol names and addresses), but contains
  debugging data only if the module was compiled with the -g CFLAG.  In
  addition, the module may have compiled with CONFIG_KALLSYMS, which means
  that the module's symbolic data will have been loaded into the kernel's
  address space when it was installed.  If the module was not compiled with
  CONFIG_KALLSYMS, then only the module's exported symbols will be loaded
  into the kernel's address space.  Therefore, for the purpose of this
  command, it should noted that a kernel module may have been compiled in
  one of following manners:

  1. If the module was built without CONFIG_KALLSYMS and without the -g CFLAG,
     then the loading of the module's additional non-exported symbols can
     be accomplished with this command.
  2. If the module was built with CONFIG_KALLSYMS, but without the -g CFLAG,
     then there is no benefit in loading the symbols from the module object
     file, because all of the module's symbols will have been loaded into the
     kernel's address space when it was installed.
  3. If the module was built with CONFIG_KALLSYMS and with the the -g CFLAG,
     then the loading of the module's debugging data can be accomplished
     with this command.
  4. If the module was built without CONFIG_KALLSYMS but with the -g CFLAG,
     then the loading of the both module's symbolic and debugging data can
     be accomplished with this command.
 
  -s module [objfile]  Loads symbolic and debugging data from the object file
                       for the module specified.  If no objfile argument is
                       appended, a search will be made for an object file
                       consisting of the module name with a .o or .ko suffix,
                       starting at the /lib/modules/<release> directory on
                       the host system.  If an objfile argument is appended,
                       then that file will be used.
            -d module  Deletes the symbolic and debugging data of the module
                       specified.
       -S [directory]  Load symbolic and debugging data from the object file
                       for all loaded modules.  For each module, a search
                       will be made for an object file consisting of the
                       module name with a .o or.ko suffix, starting at the
                       /lib/modules/<release> directory of the host system.
                       If a directory argument is appended, then the search
                       will be restricted to that directory.
                   -D  Deletes the symbolic and debugging data of all modules.
                   -r  Reinitialize module data. All currently-loaded symbolic
                       and debugging data will be deleted, and the installed
                       module list will be updated (live system only).
                   -o  Load module symbols with old mechanism.
 
  After symbolic and debugging data have been loaded, backtraces and text
  disassembly will be displayed appropriately.  Depending upon the processor
  architecture, data may also printed symbolically with the "p" command;
  at a minimum, the "rd" command may be used with module data symbols.
 
  If crash can recognize that the set of modules has changed while running a
  session on a live kernel, the module data will be reinitialized the next
  time this command is run; the -r option forces the reinitialization.

====================================================




（5） 其他命令

dmesg  查看内核崩溃时的  log 
rd  读相应的内存等。
可以用help命令查看帮助
dis -l 可以反汇编出错的地址，并且列出源代码的行数


crash32> help dis

NAME
  dis - disassemble

SYNOPSIS
  dis [-r][-l][-u][-b [num]] [address | symbol | (expression)] [count]

DESCRIPTION
  This command disassembles source code instructions starting (or ending) at
  a text address that may be expressed by value, symbol or expression:

            -r  (reverse) displays all instructions from the start of the 
                routine up to and including the designated address.
            -l  displays source code line number data in addition to the 
                disassembly output.
            -u  address is a user virtual address in the current context;
                otherwise the address is assumed to be a kernel virtual address.
                If this option is used, then -r and -l are ignored.
      -b [num]  modify the pre-calculated number of encoded bytes to skip after
                a kernel BUG ("ud2a") instruction; with no argument, displays
                the current number of bytes being skipped. (x86 and x86_64 only)
       address  starting hexadecimal text address.
        symbol  symbol of starting text address.  On ppc64, the symbol

crash32> dis -l c012c033
/BUILD/linux-2.4.9-e.71/polled_io.c: 177
0xc012c033 <polled_io_poll+83>: test   %eax,0xc(%ebx)

```
