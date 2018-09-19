```text
程序里面有一个格式化ip地址的地方使用sprintf写的，大概这样

sprintf("%d.%d.%d.%d", b[0], b[1],b[2],b[3])

性能测试的时候发现这句代码，要cpu消耗占整个的%1左右，看上去比较突出。

网上很多也自己写了itoa函数，比如

http://www.jb.man.ac.uk/~slowe/cpp/itoa.html

	
	/**
	 * C++ version 0.4 char* style "itoa":
	 * Written by Lukás Chmela
	 * Released under GPLv3.
	 */
	char* itoa(int value, char* result, int base) {
		// check that the base if valid
		if (base < 2 || base > 36) { *result = '\0'; return result; }
	
		char* ptr = result, *ptr1 = result, tmp_char;
		int tmp_value;
	
		do {
			tmp_value = value;
			value /= base;
			*ptr++ = "zyxwvutsrqponmlkjihgfedcba9876543210123456789abcdefghijklmnopqrstuvwxyz" [35 + (tmp_value - value * base)];
		} while ( value );
	
		// Apply negative sign
		if (tmp_value < 0) *ptr++ = '-';
		*ptr-- = '\0';
		while(ptr1 < ptr) {
			tmp_char = *ptr;
			*ptr--= *ptr1;
			*ptr1++ = tmp_char;
		}
		return result;
	}
	


还有之前一个logger库里面用到的


const char digits[] = "9876543210123456789";
const char* zero = digits + 9;

// Efficient Integer to String Conversions, by Matthew Wilson.
template<typename T>
size_t convert(char buf[], T value)
{
  T i = value;
  char* p = buf;

  do
  {
    int lsd = static_cast<int>(i % 10);
    i /= 10;
    *p++ = zero[lsd];
  } while (i != 0);

  if (value < 0)
  {
    *p++ = '-';
  }
  *p = '\0';
  std::reverse(buf, p);

  return p - buf;
}




上面这个应该来自这篇文章吧。

Flexible C++ #1: Efficient Integer to String Conversions

 http://www.drdobbs.com/flexible-c-1-efficient-integer-to-string/184401596?pgno=1



其实两个实现思路都是一样的，不过一个是c一个是c++代码而已。 itoa那个考虑了多进制的。

这个应该是比较快的itoa的实现了？  使用上面这个convert函数替换掉sprintf的话，用来格式化ip地址大概能快个20倍左右。

但如果自己写一个专门针对ip地址的格式化字符串的itoa （每个字节只有3位，而且都是非负数，  可以用 '0' + index 来取代上面那个digits[] = "9876543210123456789"内存引用，然后可以去掉那个while循环和负数的判断），粗略测试一下可以比之前的sprintf要快40倍左右。



inline size_t format_unsigned_char(char * buf, unsigned char c) 
{
	char *output = buf;
	if ( c >= 100) {
		int d1 = c/100;		
		int d2 = (c%100)/10;
		int d3 = c%10;
		output[0] = '0' + d1;
		output[1] = '0' + d2;
		output[2] = '0' + d3;
		return 3;
	} else if (c >=10) {
		int d2 = c/10;
		int d3 = c%10;
		output[0] = '0' + d2;
		output[1] = '0' + d3;
		return 2;
	} else {
		int d3 = c;
		output[0] = '0' + d3;
		return 1;
	}
}
 
__declspec(noinline) int format_ip_addr (const void *ip_addr, char *ip_str)
{
	const unsigned char *b = (unsigned char *)ip_addr;
	char *output = ip_str;
	output +=format_unsigned_char(output, b[0]);
	*(output++) = '.';
	output +=format_unsigned_char(output, b[1]);
	*(output++) = '.';
	output +=format_unsigned_char(output, b[2]);
	*(output++) = '.';
	output +=format_unsigned_char(output, b[3]);
	*(output++) = '\0';
	return output - (char *)ip_addr;
}

```

2018-09-19 注：
现在狠多人注意到这个格式化的问题了，比如 facebook开源的folly c++库 应该有一个专门的数字和字符串转换的优化的实现。
