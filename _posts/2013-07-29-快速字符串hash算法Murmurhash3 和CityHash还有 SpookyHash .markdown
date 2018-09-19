```text
Murmurhash3 按它自己的测试说已经比fnv要快5倍了，我一直以为fnv是最快了的，vc 2010里面的std默认用的就是fnv。

CityHash 是后来Google受了Murmurhash3的启发，再改进了的？ Google内部的hash_map<string, int> 都是用这个算法了。



http://blog.reverberate.org/2012/01/state-of-hash-functions-2012.html

这篇文章比较了上面3个算法的特点，但好像说的不是很对，或者后来CityHash这些又更新了。

我自己也看了一下：

Murmurhash3代码相对来说简单很多。

SpookyHash 这个只有128位的输出的。

CityHash这个有32位 64位 128 258输出等版本，还有使用SSE4.2  CRC32指令的版本。32位的版本如果长度小于24的话，应该直接调用Murmurhash3来的？大于24个字节长度的话，很复杂，利用了很多展开的技巧，估计是为了最大化利用cpu的流水线。64位的输出的话利用了64位整数运算等。这些优化可能和cpu有关，在比较现代的cpu上更好？有些代码居于Murmurhash3来的吧。

    反正如果只要32位的hash，而且要做hash的字节长度又比较小（小于24个字节长度）的，用Murmurhash3就好了，代码简单，性能应该也是最好的。如果长度又比较长，为了最大化性能可以考虑CityHash，可能在比较新的cpu上表现更好，要求64位 128位输出那些也可以考虑CityHash。

SpookyHash如果要128位输出可以考虑一下吧。不知道跟CityHash想比较怎么样。


http://burtleburtle.net/bob/hash/spooky.html  SpookyHash的网站有实现代码

https://code.google.com/p/cityhash/   cityhash的开源代码

https://code.google.com/p/smhasher/wiki/MurmurHash3   murmurhash3算法的介绍

http://zh.wikipedia.org/wiki/Murmur%E5%93%88%E5%B8%8C

http://en.wikipedia.org/wiki/CityHash

http://blog.csdn.net/yfkiss/article/details/7337382



这样的话我也觉得在32位的机器可以考虑使用murmurhash3，用来替换fnv32，肯定是比fnv32快的了，代码也不复杂。

https://github.com/dcjones/hat-trie/blob/master/src/murmurhash3.c 这里有一个实现代码，摘录一下呵呵

/* This is MurmurHash3. The original C++ code was placed in the public domain
 * by its author, Austin Appleby. */

#include "murmurhash3.h"

static inline uint32_t fmix(uint32_t h)
{
    h ^= h >> 16;
    h *= 0x85ebca6b;
    h ^= h >> 13;
    h *= 0xc2b2ae35;
    h ^= h >> 16;

    return h;
}


static inline uint32_t rotl32(uint32_t x, int8_t r)
{
    return (x << r) | (x >> (32 - r));
}


uint32_t hash(const char* data, size_t len_)
{
    const int len = (int) len_;
    const int nblocks = len / 4;

    uint32_t h1 = 0xc062fb4a;

    uint32_t c1 = 0xcc9e2d51;
    uint32_t c2 = 0x1b873593;

    //----------
    // body

    const uint32_t * blocks = (const uint32_t*) (data + nblocks * 4);

    int i;
    for(i = -nblocks; i; i++)
    {
        uint32_t k1 = blocks[i];

        k1 *= c1;
        k1 = rotl32(k1, 15);
        k1 *= c2;

        h1 ^= k1;
        h1 = rotl32(h1, 13);
        h1 = h1*5+0xe6546b64;
    }

    //----------
    // tail

    const uint8_t * tail = (const uint8_t*)(data + nblocks*4);

    uint32_t k1 = 0;

    switch(len & 3)
    {
        case 3: k1 ^= tail[2] << 16;
        case 2: k1 ^= tail[1] << 8;
        case 1: k1 ^= tail[0];
              k1 *= c1; k1 = rotl32(k1,15); k1 *= c2; h1 ^= k1;
    }

    //----------
    // finalization

    h1 ^= len;

    h1 = fmix(h1);

    return h1;
}



在我自己的电脑上（Intel E6550 的cpu, vc 2010）对 cityhash32 和 murmurhash3 和fnx32做了下简单的测试，结果如下

输入使用key长度: 5
CityHash32  :执行 200000 次， 耗时 2766 微秒
murmurhash3 :执行 200000 次， 耗时 2621 微秒
fnv32       :执行 200000 次， 耗时 1596 微秒
------------------------------
输入使用key长度: 8
CityHash32  :执行 200000 次， 耗时 2781 微秒
murmurhash3 :执行 200000 次， 耗时 2909 微秒
fnv32       :执行 200000 次， 耗时 2249 微秒
------------------------------
输入使用key长度: 10
CityHash32  :执行 200000 次， 耗时 2967 微秒
murmurhash3 :执行 200000 次， 耗时 3171 微秒
fnv32       :执行 200000 次， 耗时 2945 微秒
------------------------------
输入使用key长度: 16
CityHash32  :执行 200000 次， 耗时 3510 微秒
murmurhash3 :执行 200000 次， 耗时 3541 微秒
fnv32       :执行 200000 次， 耗时 4863 微秒
------------------------------
输入使用key长度: 20
CityHash32  :执行 200000 次， 耗时 3551 微秒
murmurhash3 :执行 200000 次， 耗时 3897 微秒
fnv32       :执行 200000 次， 耗时 6076 微秒
------------------------------
输入使用key长度: 23
CityHash32  :执行 200000 次， 耗时 3603 微秒
murmurhash3 :执行 200000 次， 耗时 4250 微秒
fnv32       :执行 200000 次， 耗时 6355 微秒
------------------------------
输入使用key长度: 24
CityHash32  :执行 200000 次， 耗时 3513 微秒
murmurhash3 :执行 200000 次， 耗时 4294 微秒
fnv32       :执行 200000 次， 耗时 6473 微秒
------------------------------
输入使用key长度: 64
CityHash32  :执行 200000 次， 耗时 7880 微秒
murmurhash3 :执行 200000 次， 耗时 7919 微秒
fnv32       :执行 200000 次， 耗时 20033 微秒
------------------------------
输入使用key长度: 128
CityHash32  :执行 200000 次， 耗时 11844 微秒
murmurhash3 :执行 200000 次， 耗时 13743 微秒
fnv32       :执行 200000 次， 耗时 45484 微秒
------------------------------
输入使用key长度: 256
CityHash32  :执行 200000 次， 耗时 20260 微秒
murmurhash3 :执行 200000 次， 耗时 25730 微秒
fnv32       :执行 200000 次， 耗时 90331 微秒
------------------------------
可以看到如果长度大于10开始，fnv32就开始慢于cityhash和murmurhash3了，最多要3到4倍左右。
CityHash32 比murmurhash3 稍微快那么一点点。

测试代码如下

//#include <winsock2.h>
//#include <ws2tcpip.h>

#include <iomanip>
#include <fstream>
#include <iostream>
#include <map>
#include <sstream>
#include <list>
#include <vector>
//
//#include <boost/shared_ptr.hpp>
//#include <boost/weak_ptr.hpp>

#include <stdlib.h>
#include <stdint.h>
#include <stdio.h>
#include <sys/types.h>
#include <sys/timeb.h>
#include <time.h>

#include <Windows.h>

using namespace std;










//-------------------------------------

static inline uint32_t fmix(uint32_t h)
{
    h ^= h >> 16;
    h *= 0x85ebca6b;
    h ^= h >> 13;
    h *= 0xc2b2ae35;
    h ^= h >> 16;

    return h;
}


static inline uint32_t rotl32(uint32_t x, int8_t r)
{
    return (x << r) | (x >> (32 - r));
}


uint32_t murmurhash3(const char* data, size_t len_)
{
    const int len = (int) len_;
    const int nblocks = len / 4;

    uint32_t h1 = 0xc062fb4a;

    uint32_t c1 = 0xcc9e2d51;
    uint32_t c2 = 0x1b873593;

    //----------
    // body

    const uint32_t * blocks = (const uint32_t*) (data + nblocks * 4);

    int i;
    for(i = -nblocks; i; i++)
    {
        uint32_t k1 = blocks[i];

        k1 *= c1;
        k1 = rotl32(k1, 15);
        k1 *= c2;

        h1 ^= k1;
        h1 = rotl32(h1, 13);
        h1 = h1*5+0xe6546b64;
    }

    //----------
    // tail

    const uint8_t * tail = (const uint8_t*)(data + nblocks*4);

    uint32_t k1 = 0;

    switch(len & 3)
    {
        case 3: k1 ^= tail[2] << 16;
        case 2: k1 ^= tail[1] << 8;
        case 1: k1 ^= tail[0];
              k1 *= c1; k1 = rotl32(k1,15); k1 *= c2; h1 ^= k1;
    }

    //----------
    // finalization

    h1 ^= len;

    h1 = fmix(h1);

    return h1;
}

//------------------------------------------------------------
const uint32_t FNV_32_HASH_START = 216613626UL;
const uint64_t FNV_64_HASH_START = 14695981039346656037ULL;

inline uint32_t fnv32(const char* s,
                      uint32_t hash = FNV_32_HASH_START) {
  for (; *s; ++s) {
    hash += (hash << 1) + (hash << 4) + (hash << 7) +
            (hash << 8) + (hash << 24);
    hash ^= *s;
  }
  return hash;
}

inline uint32_t fnv32_buf(const void* buf,
                          int n,
                          uint32_t hash = FNV_32_HASH_START) {
  const char* char_buf = reinterpret_cast<const char*>(buf);

  for (int i = 0; i < n; ++i) {
    hash += (hash << 1) + (hash << 4) + (hash << 7) +
            (hash << 8) + (hash << 24);
    hash ^= char_buf[i];
  }

  return hash;
}

//-----------------------------------------------------


#include "city.h"









int main (int, char**)
{
	
	//int bbbb;
	//std::cin >>bbbb;
	//return 0;

	stringstream sstream;
	LARGE_INTEGER freq, t0, t1;
	QueryPerformanceFrequency(&freq);
	size_t number = 100000000; 
	unsigned long long total_counter = 0;
	int i, j;
	unsigned long time;

	char * hash_key = new char [256];	
	memcpy(hash_key, "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
  "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag" 
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"		
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
  "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag" 
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"	
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
  "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag" 
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"
 "abd3;456,88002dfadfd80234------13gag"			
		, 256);

	int length = 10;
repeat_test:
	cout << "输入使用key长度: " ;
	cin >> length;

	//----------------------------------
	QueryPerformanceCounter(&t0);
	for (i=0; i< 200000; i++) {
		total_counter += CityHash32(hash_key,length);
	}
	QueryPerformanceCounter(&t1);
	time = (((t1.QuadPart-t0.QuadPart)*1000000)/freq.QuadPart);
	std::cout  << "CityHash32  :执行 " << i <<" 次， 耗时 " << time <<  " 微秒" << std::endl;
	//----------------------------------

	QueryPerformanceCounter(&t0);
	for (i=0; i< 200000; i++) {
		total_counter += murmurhash3(hash_key,length);
	}
	QueryPerformanceCounter(&t1);
	time = (((t1.QuadPart-t0.QuadPart)*1000000)/freq.QuadPart);
	std::cout  << "murmurhash3 :执行 " << i <<" 次， 耗时 " << time <<  " 微秒" << std::endl;

	//------------------------------------
	QueryPerformanceCounter(&t0);
	for (i=0; i< 200000; i++) {
		total_counter += fnv32_buf(hash_key,length);
	}
	QueryPerformanceCounter(&t1);
	time = (((t1.QuadPart-t0.QuadPart)*1000000)/freq.QuadPart);
	std::cout  << "fnv32       :执行 " << i <<" 次， 耗时 " << time <<  " 微秒" << std::endl;
	//--------------------------------------

	cout << "------------------------------" << endl;

	if (length < 256){
goto repeat_test; 
	}

	int aaa;
	cin >> aaa;
	cout << aaa; 

	cout<< total_counter << endl;
	return 0;
}






```
