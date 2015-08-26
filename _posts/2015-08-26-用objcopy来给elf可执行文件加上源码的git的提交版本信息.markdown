objcopy 把一个文件全部内容作为elf格式的section插入
=================================================

```
#  git rev-parse HEAD
0af0d9e7fb4cbecd5b22f00f1a136c9db933b26d

# git rev-parse --short HEAD
0af0d9e

# cat /tmp/git_commit_hash
git_commit_hash=0af0d9e7fb4cbecd5b22f00f1a136c9db933b26d

# objcopy  --add-section .git_commit_hash=/tmp/git_commit_hash  /usr/bin/test  /tmp/test

# file /tmp/test
/tmp/test: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), not stripped


/tmp/test:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         001ac282  08048c00  08048c00  00000c00  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .plt          00000170  081f4ea0  081f4ea0  001acea0  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  2 .rodata       000f1970  081f6000  081f6000  001ae000  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .typelink     0000083c  082e7970  082e7970  0029f970  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .gosymtab     00000000  082e81ac  082e81ac  002a01ac  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  5 .gopclntab    0009640a  082e81c0  082e81c0  002a01c0  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  6 .dynsym       000001c0  0837eb60  0837eb60  00336b60  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  7 .rel          00000008  0837e5d0  0837e5d0  003365d0  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  8 .gnu.version  00000038  0837e5e0  0837e5e0  003365e0  2**1
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  9 .gnu.version_r 00000070  0837e620  0837e620  00336620  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 10 .hash         00000090  0837e6a0  0837e6a0  003366a0  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 11 .rel.plt      000000b0  0837e740  0837e740  00336740  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 12 .dynstr       000001a5  0837e9a0  0837e9a0  003369a0  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 13 .got          00000004  0837f000  0837f000  00337000  2**2
                  CONTENTS, ALLOC, LOAD, DATA
 14 .got.plt      00000064  0837f020  0837f020  00337020  2**2
                  CONTENTS, ALLOC, LOAD, DATA
 15 .dynamic      00000098  0837f0a0  0837f0a0  003370a0  2**2
                  CONTENTS, ALLOC, LOAD, DATA
 16 .noptrdata    00015d0c  0837f140  0837f140  00337140  2**5
                  CONTENTS, ALLOC, LOAD, DATA
 17 .data         00003b54  08394e60  08394e60  0034ce60  2**5
                  CONTENTS, ALLOC, LOAD, DATA
 18 .bss          000066c0  083989c0  083989c0  003509b4  2**5
                  ALLOC
 19 .noptrbss     0000d860  0839f080  0839f080  003509b4  2**5
                  ALLOC
 20 .interp       00000013  08048bed  08048bed  00000bed  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 21 .tbss         00000008  00000000  00000000  003509b4  2**2
                  ALLOC, THREAD_LOCAL
 22 .debug_abbrev 000000fd  00000000  00000000  003509b4  2**0
                  CONTENTS, READONLY, DEBUGGING
 23 .debug_line   00028509  00000000  00000000  00350ab1  2**0
                  CONTENTS, READONLY, DEBUGGING
 24 .debug_frame  00025720  00000000  00000000  00378fba  2**0
                  CONTENTS, READONLY, DEBUGGING
 25 .debug_info   00098d4b  00000000  00000000  0039e6da  2**0
                  CONTENTS, READONLY, DEBUGGING
 26 .debug_pubnames 00031ae3  00000000  00000000  00437425  2**0
                  CONTENTS, READONLY, DEBUGGING
 27 .debug_pubtypes 00011d87  00000000  00000000  00468f08  2**0
                  CONTENTS, READONLY, DEBUGGING
 28 .debug_aranges 0000001c  00000000  00000000  0047ac8f  2**0
                  CONTENTS, READONLY, DEBUGGING
 29 .git_commit_hash 00000039  00000000  00000000  0047acab  2**0
                  CONTENTS, READONLY

# objdump -t  /tmp/test |grep git_commit
00000000 l    d  .git_commit_hash	00000000              .git_commit_hash

# readelf -S  /tmp/test
There are 34 section headers, starting at offset 0x47ae24:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        08048c00 000c00 1ac282 00  AX  0   0 16
  [ 2] .plt              PROGBITS        081f4ea0 1acea0 000170 04  AX  0   0  4
  [ 3] .rodata           PROGBITS        081f6000 1ae000 0f1970 00   A  0   0 32
  [ 4] .typelink         PROGBITS        082e7970 29f970 00083c 00   A  0   0  4
  [ 5] .gosymtab         PROGBITS        082e81ac 2a01ac 000000 00   A  0   0  1
  [ 6] .gopclntab        PROGBITS        082e81c0 2a01c0 09640a 00   A  0   0 32
  [ 7] .dynsym           DYNSYM          0837eb60 336b60 0001c0 10   A 13   0  4
  [ 8] .rel              REL             0837e5d0 3365d0 000008 08   A  7   0  4
  [ 9] .gnu.version      VERSYM          0837e5e0 3365e0 000038 02   A  7   0  2
  [10] .gnu.version_r    VERNEED         0837e620 336620 000070 00   A 13   2  4
  [11] .hash             HASH            0837e6a0 3366a0 000090 04   A  7   0  4
  [12] .rel.plt          REL             0837e740 336740 0000b0 08   A  7   2  1
  [13] .dynstr           STRTAB          0837e9a0 3369a0 0001a5 00   A  0   0  1
  [14] .got              PROGBITS        0837f000 337000 000004 04  WA  0   0  4
  [15] .got.plt          PROGBITS        0837f020 337020 000064 04  WA  0   0  4
  [16] .dynamic          DYNAMIC         0837f0a0 3370a0 000098 08  WA 13   0  4
  [17] .noptrdata        PROGBITS        0837f140 337140 015d0c 00  WA  0   0 32
  [18] .data             PROGBITS        08394e60 34ce60 003b54 00  WA  0   0 32
  [19] .bss              NOBITS          083989c0 3509b4 0066c0 00  WA  0   0 32
  [20] .noptrbss         NOBITS          0839f080 3509b4 00d860 00  WA  0   0 32
  [21] .interp           PROGBITS        08048bed 000bed 000013 00   A  0   0  1
  [22] .tbss             NOBITS          00000000 3509b4 000008 00 WAT  0   0  4
  [23] .debug_abbrev     PROGBITS        00000000 3509b4 0000fd 00      0   0  1
  [24] .debug_line       PROGBITS        00000000 350ab1 028509 00      0   0  1
  [25] .debug_frame      PROGBITS        00000000 378fba 025720 00      0   0  1
  [26] .debug_info       PROGBITS        00000000 39e6da 098d4b 00      0   0  1
  [27] .debug_pubnames   PROGBITS        00000000 437425 031ae3 00      0   0  1
  [28] .debug_pubtypes   PROGBITS        00000000 468f08 011d87 00      0   0  1
  [29] .debug_aranges    PROGBITS        00000000 47ac8f 00001c 00      0   0  1
  [30] .git_commit_hash  PROGBITS        00000000 47acab 000039 00      0   0  1
  [31] .shstrtab         STRTAB          00000000 47ace4 000140 00      0   0  1
  [32] .symtab           SYMTAB          00000000 47b374 01bf90 10     33 218  4
  [33] .strtab           STRTAB          00000000 497304 02b58f 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)

# readelf -p .git_commit_hash  /tmp/test
String dump of section '.git_commit_hash':
  [     0]  git_commit_hash=0af0d9e7fb4cbecd5b22f00f1a136c9db933b26d
```

把调试符号信息抽取到独立文件
============================
```
# objcopy --only-keep-debug /tmp/test  /tmp/test.debuginfo

# ls -lh /tmp
-rw-r--r-- 1 root root   57  8月 26 11:50 git_commit_hash
-rwxr-xr-x 1 root root 4.8M  8月 26 13:28 test
-rwxr-xr-x 1 root root 1.5M  8月 26 13:44 test.debuginfo
```

生成不包含调试信息的文件
========================
```
# objcopy --strip-debug /tmp/test /tmp/test.nodebuginfo
```

把调试信息合并到可执行文件
==========================
```
# objcopy --add-gnu-debuglink=/tmp/test.debuginfo /tmp/test.nodebuginfo
```




把一个文件作为二进制object文件，然后链接到可执行文件使用
=========================================================
```
# objcopy -I binary -O elf32-i386 -B i386 /tmp/git_commit_hash  /tmp/git_commit_hash.o

#  file /tmp/git_commit_hash.o
/tmp/git_commit_hash.o: ELF 32-bit LSB relocatable, Intel 80386, version 1 (SYSV), not stripped

# readelf -S  /tmp/git_commit_hash.o
There are 5 section headers, starting at offset 0x90:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .data             PROGBITS        00000000 000034 000039 00  WA  0   0  1
  [ 2] .shstrtab         STRTAB          00000000 00006d 000021 00      0   0  1
  [ 3] .symtab           SYMTAB          00000000 000158 000050 10      4   2  4
  [ 4] .strtab           STRTAB          00000000 0001a8 000067 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)


# nm   /tmp/git_commit_hash.o |grep git
00000039 D _binary__tmp_git_commit_hash_end
00000039 A _binary__tmp_git_commit_hash_size
00000000 D _binary__tmp_git_commit_hash_start
```

可以看到符号是用文件名加 start/end来表示开始和结尾的

```
# strings /tmp/git_commit_hash.o
git_commit_hash=0af0d9e7fb4cbecd5b22f00f1a136c9db933b26d
```

链接使用这个object文件
```
g++ -g -std=c++11 -O2 -o clock_gettime_test  /tmp/git_commit_hash.o  main.c -lrt
```

这样生成的elf格式没有新section 但是有这个符号
```
# nm clock_gettime_test
08049d84 d _DYNAMIC
080489b0 t _GLOBAL__sub_I_main
08048b90 T __libc_csu_init
         U __libc_start_main@@GLIBC_2.0
08049f25 D _binary__tmp_git_commit_hash_end
00000039 A _binary__tmp_git_commit_hash_size
08049eec D _binary__tmp_git_commit_hash_start
```

可以看到 _binary__tmp_git_commit_hash_start 和 _binary__tmp_git_commit_hash_end 符号有正确的添加进可执行文件。
在c代码里面可以用extern 来引用这个几个符号使用。

```cpp
int main(void)
{
  extern char  _binary__tmp_git_commit_hash_start;
  extern char  _binary__tmp_git_commit_hash_end;
  char * p = &_binary__tmp_git_commit_hash_start;
  while (p != & _binary__tmp_git_commit_hash_end) {
    std::cout << *p++;
  }
  std::cout << std::endl;
  return 0;
}
```

测试一下
```
# g++ -g -std=c++11 -O2 -o clock_gettime_test  /tmp/git_commit_hash.o  main.c -lrt
# ./clock_gettime_test 
git_commit_hash=0af0d9e7fb4cbecd5b22f00f1a136c9db933b26d
```

cmake
=====
可以在cmake里面使用 execute_process 命令的OUTPUT_VARIABLE OUTPUT_FILE 等把上述
的objcopy 命令集成的编译过程里面去。  让生成elef 自动携带 git commit log 的信息。

source file 的 property 设置 EXTERNAL_OBJECT， 告诉cmake这个一个object文件不是普通的源码文件


总结一下
========

1.  可以用objcopy 插入一个 自定义的elf section。 这个可以在编译完成之后才进行。
2.  可以用objcopy 可以把文件先编译成为object文件，然后编译出来的elf可以通过 符号访问这个文件内容。
3.  可以自动生成一个c源码文件， 把git commit 的信息字符 赋值给 static char[] 数组。这样每个模块再
    链接这个c文件就可以了。这个也是通用性最好的办法了。
    还可以以通过编译环境里面宏里面来来来赋值：
    volatile const char git_commit_hash[]= GIT_COMMIT_HASH;
    gcc -DGIT_COMMIT_HASH="git_commit_hash=xxxxx"


