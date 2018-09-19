```text
Andrei Alexandrescu "Three Optimization Tips for C++"
网上可以找到这个演示，里面提到了几种优化办法，不过都被墙了，
下面这个页面也有点介绍
https://m.facebook.com/notes/facebook-engineering/three-optimization-tips-for-c/10151361643253920

提到的优化技巧：
Prefer static linking and position-dependent code (as opposed to PIC, position-independent code).
Prefer 64-bit code and 32-bit data.
Prefer array indexing to pointers (this one seems to reverse every ten years).
Prefer regular memory access patterns.
Minimize control flow.            // 减少复杂的跳转，复杂的if else 函数调用次数等
Avoid data dependencies.
避免内存操作，宁愿更多的计算也不要才注意内存。一个cache不命中造成的延时就可能导致cpu流水线中断很久。
浮点数的运算和整型数一样快，浮点数的乘除法甚至更快


// 普通的转换
uint32_t u64ToAsciiClassic(uint64_t value, char* dst) {
	// Write backwards.
	auto start = dst;
	do {
		*dst++ = '0' + (value % 10);
		value /= 10;
	} while (value != 0);
	const uint32_t result = dst - start;
	// Reverse in place.
	for (dst--; dst > start; start++, dst--) {
		std::iter_swap(dst, start);
	}
	return result;
}


uint32_t digits10(uint64_t v) {
    uint32_t result = 0;
    do {
        ++result;
         v /= 10;
    } while (v);
     return result;
}


//优化1 用比较操作取代除法
uint32_t digits10(uint64_t v) {
	uint32_t result = 1;
	for (;;) {
		if (v < 10) return result;
		if (v < 100) return result + 1;
		if (v < 1000) return result + 2;
		if (v < 10000) return result + 3;
		// Skip ahead by 4 orders of magnitude
		v /= 10000U;
		result += 4;
	}
}

//优化2  用计算取代数组内存的读写，避免后面那个std::iter_swap
uint32_t uint64ToAscii(uint64_t v, char *const buffer) {
	auto const result = digits10(v);
	uint32_t pos = result - 1;
	while (v >= 10) {
		auto const q = v / 10;
		auto const r = static_cast<uint32_t>(v % 10); 
		buffer[pos--] = '0' + r;
		v = q;
	} 
	//assert(pos == 0); // Last digit is trivial to handle
	*buffer = static_cast<uint32_t>(v)+ '0';
		return result;
}


const uint64_t P01 = 10;
const uint64_t P02 = 100;
const uint64_t P03 = 1000;
const uint64_t P04 = 10000;
const uint64_t P05 = 100000;
const uint64_t P06 = 1000000;
const uint64_t P07 = 10000000;
const uint64_t P08 = 100000000;
const uint64_t P09 = 1000000000;
const uint64_t P10 = 10000000000;
const uint64_t P11 = 100000000000;
const uint64_t P12 = 1000000000000;

//优化3算法优化  二分查找十进制表示的数字位数
uint32_t digits10(uint64_t v) {
	if (v < P01) return 1;
	if (v < P02) return 2;
	if (v < P03) return 3;
	if (v < P12) {
		if (v < P08) {
			if (v < P06) {
				if (v < P04) return 4;
				return 5 + (v >= P05);
			}
			return 7 + (v >= P07);
		}
		if (v < P10) {
			return 9 + (v >= P09);
		}
		return 11 + (v >= P11);
	}
	return 12 + digits10(v / P12);
}


//优化3 算法优化  每次转换两个数字
unsigned u64ToAsciiTable(uint64_t value, char* dst) {
	static const char digits[201] =
		"0001020304050607080910111213141516171819"
		"2021222324252627282930313233343536373839"
		"4041424344454647484950515253545556575859"
		"6061626364656667686970717273747576777879"
		"8081828384858687888990919293949596979899";
	uint32_t const length = digits10(value);
	uint32_t next = length - 1;
	while (value >= 100) {
		auto const i = (value % 100) * 2;
		value /= 100;
		dst[next] = digits[i + 1];
		dst[next - 1] = digits[i];
		next -= 2;
	}
	// Handle last 1-2 digits
	if (value < 10) {
		dst[next] = '0' + uint32_t(value);
	}
	else {
		auto i = uint32_t(value) * 2;
		dst[next] = digits[i + 1];
		dst[next - 1] = digits[i];
	}
	return length;
}




对这3个数做测试
4557
3452635722
9223372036854775800

------原始版本测试结果--------------
输入要测试的数:4557
执行次数(n):    100000000
总耗时(微秒):    3499944
平均耗时（微秒）0.0349994

输入要测试的数:3452635722
执行次数(n):    100000000
总耗时(微秒):    8764613
平均耗时（微秒）0.0876461

输入要测试的数:9223372036854775800
执行次数(n):    100000000
总耗时(微秒):    17935778
平均耗时（微秒）0.179358

--------优化版本2的测试结果--------
输入要测试的数:4557
执行次数(n):    100000000
总耗时(微秒):    2785353
平均耗时（微秒）0.0278535

输入要测试的数:3452635722
执行次数(n):    100000000
总耗时(微秒):    9338686
平均耗时（微秒）0.0933869

输入要测试的数:922337203685477580
执行次数(n):    100000000
总耗时(微秒):    18537854
平均耗时（微秒）0.185379
--------最后优化版本3的结果--------
输入要测试的数:4557
执行次数(n):    100000000
总耗时(微秒):    2077764
平均耗时（微秒）0.0207776

输入要测试的数:455
执行次数(n):    100000000
总耗时(微秒):    1763224
平均耗时（微秒）0.0176322

输入要测试的数:3452635722
执行次数(n):    100000000
总耗时(微秒):    5109649
平均耗时（微秒）0.0510965

输入要测试的数:9223372036854775800
执行次数(n):    100000000
总耗时(微秒):    12360570
平均耗时（微秒）0.123606
-------------------------------------



在intel i7 上面的测试，好像第2版的优化作用不是很明显的。 第3版稍微好点。可能编译的32位程序，模拟的64位除非运算的影响？


测试代码

//#include <winsock2.h>
//#include <ws2tcpip.h>

#define _USE_32BIT_TIME_T
#include <fstream>
#include <iostream>
#include <map>
#include <sstream>
#include <list>
#include <vector>
#include <random>

//
//#include <boost/shared_ptr.hpp>
//#include <boost/weak_ptr.hpp>

#include <stdlib.h>
#include <stdint.h>
#include <stdio.h>
#include <sys/types.h>
#include <sys/timeb.h>
#include <time.h>
#include "pugixml.hpp"

#ifdef _WIN32
#include <Windows.h>
void my_gettimeofday(struct timeval *tp)
{
	uint64_t  intervals;
	FILETIME  ft;

	GetSystemTimeAsFileTime(&ft);

	/*
	* A file time is a 64-bit value that represents the number
	* of 100-nanosecond intervals that have elapsed since
	* January 1, 1601 12:00 A.M. UTC.
	*
	* Between January 1, 1970 (Epoch) and January 1, 1601 there were
	* 134744 days,
	* 11644473600 seconds or
	* 11644473600,000,000,0 100-nanosecond intervals.
	*
	* See also MSKB Q167296.
	*/

	intervals = ((uint64_t)ft.dwHighDateTime << 32) | ft.dwLowDateTime;
	intervals -= 116444736000000000;

	tp->tv_sec = (long)(intervals / 10000000);
	tp->tv_usec = (long)((intervals % 10000000) / 10);
}
#else
#include <sys/time.h>
#define my_gettimeofday(tp)  (void) gettimeofday(tp, NULL);
#endif

// 返回微秒时间差异
unsigned long long time_stamp_usec()
{
	enum { kUsecPerSec = 1000 * 1000 };
	struct timeval tp;
	my_gettimeofday(&tp);
	return tp.tv_sec * kUsecPerSec  + tp.tv_usec;
}

void print_time_diff(unsigned long long t0, unsigned long long t1, int n)
{
	std::cout << "执行次数(n):\t" << n  << std::endl
			  << "总耗时(微秒):\t " << t1 - t0 << std::endl
	          << "平均耗时（微秒）" << (double)(t1 - t0)/(double)n << std::endl;
}


using namespace std;



//----------------------------------------------
uint32_t digits10(uint64_t v) {
	uint32_t result = 1;
	for (;;) {
		if (v < 10) return result;
		if (v < 100) return result + 1;
		if (v < 1000) return result + 2;
		if (v < 10000) return result + 3;
		// Skip ahead by 4 orders of magnitude
		v /= 10000U;
		result += 4;
	}
}

//优化2  用计算取代数组内存的读写，避免后面那个std::iter_swap
__declspec(noinline) uint32_t uint64ToAscii(uint64_t v, char *const buffer) {
	auto const result = digits10(v);
	uint32_t pos = result - 1;
	while (v >= 10) {
		auto const q = v / 10;
		auto const r = static_cast<uint32_t>(v % 10); 
		buffer[pos--] = '0' + r;
		v = q;
	} 
	//assert(pos == 0); // Last digit is trivial to handle
	*buffer = static_cast<uint32_t>(v)+ '0';
		return result;
}


//-----------------------------------------------



int main(int, char**)
{
	
	//int a = 1;
	//
	//return 0;
again:
	char output[65];
	uint64_t a = 0;
	cout << "输入要测试的数:";
	cin >> a;



	stringstream sstream;
	unsigned long long total_counter = 0;
	int kLoopCount = 10000 * 10000;
	int i, j;
	unsigned long long t0, t1;
	//----------------------------------
	t0 = time_stamp_usec();
	for (i = 0; i < kLoopCount; i++) {
		uint64ToAscii(a, output);
		total_counter += output[i%64];
	}
	t1 = time_stamp_usec();
	print_time_diff(t0, t1, kLoopCount);	
	//----------------------------------
	std::cout << total_counter << std::endl;

	goto again;


	int aaa;
	cin >> aaa;
	cout << aaa;
	return 0;
}
```
