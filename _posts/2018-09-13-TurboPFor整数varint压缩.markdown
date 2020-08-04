修改makefile 增加一行 把MARCH改为 MARCH=-march=x86-64  把改为x86-64通用平台
```text
gcc -O2 -DNDEBUG -s -w -Wall -std=gnu99 -DUSE_THREADS  -fstrict-aliasing  -DNSIMD -march=x86-64  bitutil.o bitpack.o bitunpack.o   bitunpack_sse.o bitpack_sse.o fp.o vint.o vp4c.o vp4d.o  vp4c_sse.o vp4d_sse.o vsimple.o test.c -lm


ar -r libTurboPFor.a  bitutil.o bitpack.o bitpack_sse.o bitunpack.o bitunpack_sse.o fp.o vint.o vp4c.o vp4c_sse.o vp4d.o vp4d_sse.o vsimple.o

gcc -O2 -DNDEBUG -s -w -Wall -std=gnu99 -DUSE_THREADS  -fstrict-aliasing -DNSIMD -march=x86-64 -lm  test.c libTurboPFor.a

```
```c
#include <stdio.h>
#include <stdint.h>
#include <stddef.h>

#include "fp.h"

void main(void)
{
	uint32_t a[50] = { 31, 10, 23, 30, 35, 10, 12, 15, 16, 10,
		      21, 20, 20, 38, 35, 10, 12, 15, 16, 10,
		      33, 11, 10, 3, 35, 10, 12, 15, 16, 10,
		      35, 15, 30, 31, 35, 10, 12, 15, 17, 11,
		      38, 10, 20, 28, 35, 10, 12, 15, 16, 13,
		 } ;

/*
	uint32_t a[50] = { 5631, 7810, 1323, 4530, 2435, 810, 19312, 1315, 34616, 1310,
		      21, 8920, 1320, 1338, 2335, 23410, 23412, 2615, 4516, 410,
		      4333, 411, 410, 4573, 835, 9910, 9912, 2315, 3316, 33310,
		      35, 215, 330, 431, 435, 3310, 44412, 33315, 3317, 611,
		      938, 910, 9920, 928, 935, 810, 812, 7715, 8716, 7813,
		 } ;

	uint32_t a[50] = {
		      1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 
		      1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 
		      1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 
		      1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 
		      1, 1, 1, 1, 1, 1, 1, 1, 1, 1
		 } ;
*/

	uint32_t b[50] = { 0 };

	uint8_t out[256];

//	size_t len = bvzenc32(a, 50, out, 0);  // large, 
//	size_t len2 = bvzdec32(out, 50, b, 0);
//	size_t len = fpgenc32(a, 50, out, 0); // good
//	size_t len2 = fpgdec32(out, 50, b, 0);
//	size_t len = p4nenc32(a, 50, out);  // best
//	size_t len2 = p4ndec32(out, 50, b);
	size_t len = p4nenc128v32(a, 50, out);  // best
	size_t len2 = p4ndec128v32(out, 50, b);

	// size_t len = p4nzenc128v32(a, 50, out);  // mid, error
	// size_t len2 = p4nzdec128v32(out, 50, b); 
	// size_t len = p4nzenc32(a, 50, out);  // mid, error
	// size_t len2 = p4nzdec32(out, 50, b);

	printf("%d %d\n", len, len2);

//	uint8_t * len = vsenc32(a, 50, out); // good, error
//	uint8_t * len2 = vsdec32(out, 50, b);
//	uint8_t * len = vbenc32(a, 50, out);  // large, bad not RLE
//	uint8_t * len2 = vbdec32(out, 50, b);

//	printf("%d %d\n", len - out, len2 - out);


	for (int i = 0; i < 50 ; i++) {
		if (a[i] != b[i]) {
			printf ("error %d %d %d\n", a[i], b[i], i);
		}
	}
}

```

默认编译的出来libic.a 文件特别大，链接时gcc好像也不会选择某个函数来链接进去。 可以把 fp.c文件复制一份改一下，把没有用到的 函数都删掉，只编译自己用到的函数，这样编译出来的.a文件就特别小了。

```text
查看默认的编译选项
[root@localhost TurboPFor-Integer-Compression]# make fp.o
gcc -O3 -march=corei7-avx -mtune=corei7-avx  -w -Wall  -fstrict-aliasing -falign-loops  -Iext   fp.c -c -o fp.o

gcc -O3 -march=corei7-avx -mtune=corei7-avx  -w -Wall  -fstrict-aliasing -falign-loops  -Iext my_fp.c -c -o my_fp.o
rm -f libTurboPFor.a
ar -r libTurboPFor.a my_fp.o

测试程序
gcc -O2 -lm test.c libTurboPFor.a
```
