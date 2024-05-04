参考：   
=====
https://www.chessprogramming.org/BMI2   
https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=pext&ig_expand=5088   
https://gcc.gnu.org/onlinedocs/gcc/x86-Built-in-Functions.html   



pext 指令
========
是 “Parallel Bits Extract” 的意思，  作用就是根据你提供的一个mask掩码整数，如果mask的某一位k是1就这个 目标整数dst的第k位抽出来 作为新整数的一位。


intel提供的伪代码是下面这样的
============================
```text
Synopsis
unsigned __int64 _pext_u64 (unsigned __int64 a, unsigned __int64 mask)
#include <immintrin.h>
Instruction: pext r64, r64, r64
CPUID Flags: BMI2

Description
Extract bits from unsigned 64-bit integer a at the corresponding bit locations specified by mask to contiguous low bits in dst; the remaining upper bits in dst are set to zero.

Operation
tmp := a
dst := 0
m := 0
k := 0
DO WHILE m < 64
	IF mask[m] == 1
		dst[k] := tmp[m]
		k := k + 1
	FI
	m := m + 1
OD

```

象棋百科提供的C语言的代码
===========================
看起来没有intel的容易理解
```c
U64 _pext_u64(U64 val, U64 mask) {
  U64 res = 0;
  for (U64 bb = 1; mask; bb += bb) {
    if ( val & mask & -mask )
      res |= bb;
    mask &= mask - 1;
  }
  return res;
} 
```

象棋百科提供的示意图
====================
就是说pext运算后的解释，把s60 和 s7 s5 s2这些位抽出来组成的DEST就是指令返回的结果。
```text
SRC1   ┌───┬───┬───┬───┬───┐    ┌───┬───┬───┬───┬───┬───┬───┬───┐
       │S63│S62│S61│S60│S59│....│ S7│ S6│ S5│ S4│ S3│ S2│ S1│ S0│ 
       └───┴───┴───┴───┴───┘    └───┴───┴───┴───┴───┴───┴───┴───┘

SRC2   ┌───┬───┬───┬───┬───┐    ┌───┬───┬───┬───┬───┬───┬───┬───┐
(mask) │ 0 │ 0 │ 0 │ 1 │ 0 │0...│ 1 │ 0 │ 1 │ 0 │ 0 │ 1 │ 0 │ 0 │  (f.i. 4 bits set)
       └───┴───┴───┴───┴───┘    └───┴───┴───┴───┴───┴───┴───┴───┘

DEST   ┌───┬───┬───┬───┬───┐    ┌───┬───┬───┬───┬───┬───┬───┬───┐
       │ 0 │ 0 │ 0 │ 0 │ 0 │0...│ 0 │ 0 │ 0 │ 0 │S60│ S7│ S5│ S2│ 
       └───┴───┴───┴───┴───┘    └───┴───┴───┴───┴───┴───┴───┴───┘
```

pdep 叫做 "Parallel Bits Deposit"， 是pext的反操作吧



gcc 提供的build-in函数
======================
使用  -mbmi2 编译选项就会生成这些指令吧。 这个bmi2 指令集在intel Haswell 架构以后的cpu应该都支持，  linux系统里面cat /proc/cpuinfo  的flags 里面会带有 bmi2 字样

```c
unsigned int _bzhi_u32 (unsigned int, unsigned int);
unsigned int _pdep_u32 (unsigned int, unsigned int);
unsigned int _pext_u32 (unsigned int, unsigned int);
unsigned long long _bzhi_u64 (unsigned long long, unsigned long long);
unsigned long long _pdep_u64 (unsigned long long, unsigned long long);
unsigned long long _pext_u64 (unsigned long long, unsigned long long);
```


应用
====
这个pext指令的应用，有一个用途是把大整数转为小整数的索引，比如用来实现哈希表的完美哈希函数。比如这个https://github.com/boost-ext/mph库的用法：

```python
def hash[kv: array, unknown: typeof(kv[0][0])](key : any):
  # 0. find mask which uniquely identifies all keys [compile-time]
  mask = ~typeof(kv[0][0]) # 0b111111...

  for i in range(nbits(mask)):
    masked = []
    mask.unset(i)

    for k, v in kv:
      masked.append(k & mask)

    if not unique(masked):
      mask.set(i)

  assert unique(masked)
  assert mask != ~typeof(kv[0][0])

  lookup = array(typeof(kv[0]), 2^popcount(mask)) # static constexpr + alignment
  for k, v in kv:
    lookup[pext(k, mask)] = (k, v)

  # 1. lookup [run-time] / if key is a string convert to u32 or u64 first (memcpy)
    # word: 00101011
    # mask: 11100001
    #    &: 000____1
    # pext: ____0001 # intel/intrinsics-guide/index.html#text=pext
    def pext(a : uN, mask : uN):
      dst, m, k = ([], 0, 0)

      while m < nbits(a):
        if mask[m] == 1:
          dst.append(a[m])
          k += 1
        m += 1

      return uN(dst)

  k, v = lookup[pext(key, mask)]

  if k == key: # policies (conditional, branchless, ...)
    return v
  else:
    return unknown
```

在象棋引擎里面，主要用来实现 “bitboard位棋盘” 的 棋子攻击表的 完美哈希函数， “Fancy Magic Bitboards” “Fancy PEXT Bitboards ”，把 棋盘的掩码转换为小的哈希表索引。   
可以参考Stockfish引擎的 init_magics函数
https://github.com/official-stockfish/Stockfish/blob/master/src/bitboard.cpp

