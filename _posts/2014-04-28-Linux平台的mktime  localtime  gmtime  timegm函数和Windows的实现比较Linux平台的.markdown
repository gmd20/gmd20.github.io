```text
    
(gdb) bt
#0  __localtime_r (t=0xb7376880, tp=0xb737684c) at localtime.c:33
#1  0xb74f12db in ranged_convert (convert=<optimized out>, t=<optimized out>, tp=0xb737684c) at mktime.c:233
#2  0xb74f159c in __mktime_internal (tp=0xb73768f4, convert=0xb74f1280 <__localtime_r>, offset=0xb75c0a74) at mktime.c:405
#3  0xb74f1ad6 in *__GI_mktime (tp=0xb73768f4) at mktime.c:518
#4  0xaf8ffa40 in make_utc_time (gmt_tm=0xb73768f4) at /home/bright/inficore/service/adapter/smpp/server_submit_sm.cpp:96



glibc的对应代码
https://github.com/Xilinx/glibc/blob/9a3c6a6ff602c88d7155139a7d7d0000b7b7e946/time/mktime.c

http://code.woboq.org/userspace/glibc/time/localtime.c.html



-------------测试代码--------------------------
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
	return   ((unsigned long long)tp.tv_sec) * kUsecPerSec  + (unsigned long long ) tp.tv_usec;
}

void print_time_diff(unsigned long long t0, unsigned long long t1, int n)
{
	std::cout << "执行次数(n):\t" << n  << std::endl
			  << "总耗时(微秒):\t " << t1 - t0 << std::endl
	          << "平均耗时（微秒）" << (double)(t1 - t0)/(double)n << std::endl;
}

using namespace std;

int main(int, char**)
{
	struct tm time;
	int seconds = 0;
	std::cin >> seconds;
	time.tm_year = 114;
	time.tm_mon = 4;
	time.tm_mday = 29;
	time.tm_hour = 11;
	time.tm_min = 10;
	time.tm_sec = seconds;


	//-----------------------
	stringstream sstream;
	unsigned long long total_counter = 0;
	int kLoopCount = 10000 * 10000;
	int i, j;

	unsigned long long t0, t1;
	//----------------------------------
	t0 = time_stamp_usec();
	for (i = 0; i < kLoopCount; i++) {
		time.tm_sec = (time.tm_sec + 1) % 60;
		time_t t = mktime(&time);
		total_counter += t;
	}
	t1 = time_stamp_usec();
	print_time_diff(t0, t1, kLoopCount);	
	//----------------------------------
	std::cout << total_counter << std::endl;
	int aaa;
	cin >> aaa;
	cout << aaa;
	return 0;
}



----------------------用mktime模拟 timegm或者_mkgmtime 功能的代码--------------------

class TimezoneHelper {
public:
	TimezoneHelper() {
		struct tm * timeinfo;
		time_t secs, local_secs, gmt_secs;
		time(&secs);
		timeinfo = localtime(&secs);
		local_secs = mktime(timeinfo);
		timeinfo = gmtime(&secs);
		gmt_secs = mktime(timeinfo);
		timezone_diff_secs =  local_secs - gmt_secs;
	}
	static long timezone_diff_secs;
};
long TimezoneHelper::timezone_diff_secs = 0;
static TimezoneHelper timezone_helper;
inline time_t make_utc_time(struct tm *gmt_tm)
{
	time_t t = mktime(gmt_tm) + TimezoneHelper::timezone_diff_secs;
	return t;
}

等价于
#ifdef _WIN32
#define make_utc_time _mkgmtime
#else
#define make_utc_time timegm
#endif

-------------windows 7 64bit  vc 2013

执行次数(n):    100000000
总耗时(微秒):    19797514
平均耗时（微秒）0.197975

因为是编译的32位程序，所以用32bit运算模拟的除法要慢一些。
如果定义宏#define _USE_32BIT_TIME_T 来使用32位的time_t
测试结果如下:
执行次数(n):    100000000
总耗时(微秒):    9805745
平均耗时（微秒）0.0980575

要比64bit的time_t 时候要快一倍左右。
----------------windows 平台的_mkgmtime -----------
执行次数(n):    100000000
总耗时(微秒):    6615840
平均耗时（微秒）0.0661584
这个也要比直接用mktime要快。
----------virtualbox 4.3.10  linux -----------
bright@debian01:~/test$ uname -a
Linux debian01 3.11-0.bpo.2-686-pae #1 SMP Debian 3.11.10-1~bpo70+1 (2013-12-17) i686 GNU/Linux

bright@debian01:~/test$ gcc --version
gcc (Debian 4.7.2-5) 4.7.2

bright@debian01:~/test$ make test
g++ -g -std=c++11 -O2 -o mktime_test -lrt main.c


执行次数(n):	100000000
总耗时(微秒):	 97892366
平均耗时（微秒）0.978924

要比Windows的32位time_t慢10倍左右

---------------------------------------------------
如果改用timegm函数（不是posix标准，但glibc的也有实现bsd接口）
参考
http://linux.die.net/man/3/timegm

执行次数(n):	100000000
总耗时(微秒):	 8867971
平均耗时（微秒）0.0886797

timegm要比mktime要快好十倍的样子，比windows的mktime要快一点，但比Windows平台的_mkgmtime 要慢一些。

---------------------linux  mktime的perf top 分析结果--------------------------------
-  33.98%  libc-2.13.so        [.] _IO_vfscanf
   - _IO_vfscanf
      - 99.49% _IO_vsscanf
           sscanf
           __tzset_parse_tz
           __tzfile_compute
           __tz_convert
           __localtime_r
           ranged_convert
           __mktime_internal
           mktime
           main
           __libc_start_main
           _start
      - 0.51% sscanf
           __tzset_parse_tz
           __tzfile_compute
           __tz_convert
           __localtime_r
           ranged_convert
           __mktime_internal
           mktime
           main
           __libc_start_main
           _start
+   7.52%  libc-2.13.so        [.] __offtime
+   5.49%  libc-2.13.so        [.] __mktime_internal

----------------linux  timegm的perf top分析-------------------------------------
-  41.58%  libc-2.13.so        [.] __mktime_internal
     __mktime_internal
-  28.43%  libc-2.13.so        [.] __offtime
   - __offtime
      + 98.65% __tz_convert
      + 1.35% __gmtime_r
-   8.27%  libc-2.13.so        [.] __tz_convert
   - __tz_convert
      + 95.66% __gmtime_r
      + 4.34% ranged_convert
-   7.26%  libc-2.13.so        [.] __tzfile_compute
   + __tzfile_compute
+   3.36%  libc-2.13.so        [.] __i686.get_pc_thunk.bx
------------------------------------------------------------------


总结:
linux 的mktime要比windows的要慢 好几倍倍，
应该是函数开始的地方调用__tzset  函数来更新timezone设置或者scanf解析timezone字符串导致导致的吧?
但timegm函数和windows的 mktime就差不多了。Linux平台的timelocal函数应该等价于mktime，跟mktime一样的慢。
timegm比mktime应该就是少了一个解析timezone文件的操作。

看来可以用timegm的地方尽量不要用mktime吧。但timegm不是标准函数，之前代码里面都是用 mktime来模拟timegm的功能，自己加上localtime_offset的timezone偏移值。看来还是直接用timegm要好一些吧。


类似的有Linux的localtime要比gmtime要慢好多倍
------------windows gmtime----------
执行次数(n):    100000000
总耗时(微秒):    4909623
平均耗时（微秒）0.0490962
------------windows localtime-------
执行次数(n):    100000000
总耗时(微秒):    7635470
平均耗时（微秒）0.0763547
------------linux gmtime----------
执行次数(n):	100000000
总耗时(微秒):	 4274841
平均耗时（微秒）0.0427484
------------linux locatime-----------
执行次数(n):	100000000
总耗时(微秒):	 92113588
平均耗时（微秒）0.921136


类似的Linux平台的gmtime函数要比locatime要快很多啊。如果用的比较多的时候，可以自己获取一次timezone的秒数偏移值。然后以后都用gmtime + timezone_diff_secs来模拟locatime函数吧。像localtime这样每次去获取解析timezone确实没必要。cache一次就可以了。它的实现估计是为了兼容posix的标准，要求每次调用时都去重新获取timezone。但一般来说timezone都不会变的吧。
		struct tm * timeinfo;
		time_t secs, local_secs, gmt_secs;
		time(&secs);
		timeinfo = localtime(&secs);
		local_secs = mktime(timeinfo);
		timeinfo = gmtime(&secs);
		gmt_secs = mktime(timeinfo);
		timezone_diff_secs =  local_secs - gmt_secs;



设置要timezone计算的函数都会慢一些，如果可以的话直接使用UTC时间函数吧，有可能系统内部保存的就是UTC时间，不需要再做timezone的换算了。
```
