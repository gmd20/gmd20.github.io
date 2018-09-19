```text
    




  下载LOFTER我的照片书  |
      因为要用到16进制编码的字符串编码，自己 其实也是考虑用stringstream写个简单的格式化函数。然后在网上确实找了一个，然后改了一下。copy起来就是简单啊，不用花太多的时间去敲代码和调试。这就是下面代码里面的hex_encode 和hex_decode函数。但感觉这种方便，生成的临时string对象很多，我那个应用其实是可以原地编解码的，就在网上找了个c风格的，就是下面的hex_encode 和 hex_decode,。从http://code.google.com/p/stringencoders/ 这个stringencoders的base64编码的项目里面分离出来的。

 

      然后用vc 2010 自带的性能分析工具大概比较了一下。  vc2010 的 “performance analysis” 功能还是比较好用的。 可以使用“采样”和“精确统计”，可以和直观的得到每个函数调用时间，占用总时间百分比等信息，可以把代码里面的某一行给标出来，精确定位到 性能热点。用起来的很简单，就不用自己写时间统计功能的统计函数了，可能和intel那些性能分析工具也差不多。想比较linux里面的gcc的性能分析要简单方便一些。这也是微软的一贯风格了！

 

       最后的结果还是让我感觉到比较惊讶的，两种办法相差差不多有500倍，没想到会相差这么多。这c++如果使用时候不注意，很容易引入很多临时对象和对象初始化的代码，在这里例子里面临时的string对象和string striingstream初始化时的内存复制，string的动态增长的申请内存和释放内存都是导致最后这个糟糕结果的重要因素。

 

#include <iomanip>
#include <iostream>
#include <map>
#include <sstream>
#include <stdint.h>
 
using namespace std;
#include <stdio.h>
// dump uint32_t as hex digits
void uint32_array_to_c_hex(const uint32_t* ary, int sz, const char* name)
{
    printf("static const uint32_t %s[%d] = {\n", name, sz);
    int i = 0;
    for (;;) {
        printf("0x%08x", ary[i]);
        ++i;
        if (i == sz) break;
        if (i % 6 == 0) {
            printf(",\n");
        } else {
            printf(", ");
        }
    }
    printf("\n};\n");
}
 
 
 
/**
 
 * prints char array as a c program snippet
 
 */
void char_array_to_c(const char* ary, int sz, const char* name)
{
    printf("static const unsigned char %s[%d] = {\n", name, sz);
    uint8_t tmp;
    int i = 0;
    for (;;) {
        if (ary[i] == 0) {
            printf("'\\0'");
        } else if (ary[i] == '\n') {
            printf("'\\n'");
        } else if (ary[i] == '\t') {
            printf("'\\t'");
        } else if (ary[i] == '\r') {
            printf("'\\r'");
        } else if (ary[i] == '\'') {
            printf("'\\''");
        } else if (ary[i] == '\\') {
            printf("'\\\\'");
        } else if (ary[i] < 32 || ary[i] > 126) {
            tmp = (uint8_t) ary[i];
            printf("0x%02x", tmp);
        } else {
            printf(" '%c'", (char)ary[i]);
        }
        ++i;
        if (i == sz) break;
        if (i % 10 == 0) {
            printf(",\n");
        } else {
            printf(", ");
        }
    }
    printf("\n};\n\n");
}
 
 
/**
 * prints an uint array as a c program snippet
 */
void uint32_array_to_c(const uint32_t* ary, int sz, const char* name)
{
    printf("static const uint32_t %s[%d] = {\n", name, sz);
    int i = 0;
    for (;;) {
        printf("%3d", ary[i]);
        ++i;
        if (i == sz) break;
        if (i % 12 == 0) {
            printf(",\n");
        } else {
            printf(", ");
        }
    }
    printf("\n};\n\n");
}
 
 
void hexencodemap()
{
    static const char sHexChars[] = "0123456789ABCDEF";
    int i;
    char hexEncode1[256];
    char hexEncode2[256];
    for (i = 0; i < 256; ++i) {
        hexEncode1[i] = sHexChars[i >> 4];
        hexEncode2[i] = sHexChars[i & 0x0f];
    }
 
    char_array_to_c(hexEncode1, 256, "gsHexEncodeMap1");
    char_array_to_c(hexEncode2, 256, "gsHexEncodeMap2");
}
 
 
 
void hexdecodemap()
{
    uint32_t i;
    uint32_t map[256];
    for (i = 0; i <= 255; ++i) {
        map[i] = 256;
    }
 
    // digits
    for (i = '0'; i <= '9'; ++i) {
        map[i] = i - '0';
    }
 
    // upper
    for (i = 'A'; i <= 'F'; ++i) {
        map[i] = i - 'A' + 10;
    }
 
    // lower
    for (i = 'a'; i <= 'f'; ++i) {
        map[i] = i - 'a' + 10;
    }
 
 uint32_t map1[256];
 for (i = 0; i < 256; ++i) {
  map1[i] = map[i] << 4;
 }
    uint32_array_to_c(map1, 256,  "gsHexDecodeMap1");
 uint32_array_to_c(map, 256, "gsHexDecodeMap2");
}
 



static const uint32_t hexDecodeMap1[256] = {
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,    0,   16,
  32,   48,   64,   80,   96,  112,  128,  144, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 160,   176,  192,  208,  224,
 240, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096,  160,  176,  192,
 208,  224,  240, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096, 4096,
4096, 4096, 4096, 4096, 4096, 4096
};

static const uint32_t hexDecodeMap2[256] = {
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 0,     1,
  2,   3,   4,   5,   6,   7,   8,   9, 256, 256,
256, 256, 256, 256, 256,  10,  11,  12,  13,  14,
 15, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256,  10,  11,  12,
 13,  14,  15, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256, 256, 256, 256, 256,
256, 256, 256, 256, 256, 256
};


static const unsigned char hexEncodeMap1[256] = {
 '0',  '0',  '0',  '0',  '0',  '0',  '0',  '0',  '0',  '0',
 '0',  '0',  '0',  '0',  '0',  '0',  '1',  '1',  '1',  '1',
 '1',  '1',  '1',  '1',  '1',  '1',  '1',  '1',  '1',  '1',
 '1',  '1',  '2',  '2',  '2',  '2',  '2',  '2',  '2',  '2',
 '2',  '2',  '2',  '2',  '2',  '2',  '2',  '2',  '3',  '3',
 '3',  '3',  '3',  '3',  '3',  '3',  '3',  '3',  '3',  '3',
 '3',  '3',  '3',  '3',  '4',  '4',  '4',  '4',  '4',  '4',
 '4',  '4',  '4',  '4',  '4',  '4',  '4',  '4',  '4',  '4',
 '5',  '5',  '5',  '5',  '5',  '5',  '5',  '5',  '5',  '5',
 '5',  '5',  '5',  '5',  '5',  '5',  '6',  '6',  '6',  '6',
 '6',  '6',  '6',  '6',  '6',  '6',  '6',  '6',  '6',  '6',
 '6',  '6',  '7',  '7',  '7',  '7',  '7',  '7',  '7',  '7',
 '7',  '7',  '7',  '7',  '7',  '7',  '7',  '7',  '8',  '8',
 '8',  '8',  '8',  '8',  '8',  '8',  '8',  '8',  '8',  '8',
 '8',  '8',  '8',  '8',  '9',  '9',  '9',  '9',  '9',  '9',
 '9',  '9',  '9',  '9',  '9',  '9',  '9',  '9',  '9',  '9',
 'A',  'A',  'A',  'A',  'A',  'A',  'A',  'A',  'A',  'A',
 'A',  'A',  'A',  'A',  'A',  'A',  'B',  'B',  'B',  'B',
 'B',  'B',  'B',  'B',  'B',  'B',  'B',  'B',  'B',  'B',
 'B',  'B',  'C',  'C',  'C',  'C',  'C',  'C',  'C',  'C',
 'C',  'C',  'C',  'C',  'C',  'C',  'C',  'C',  'D',  'D',
 'D',  'D',  'D',  'D',  'D',  'D',  'D',  'D',  'D',  'D',
 'D',  'D',  'D',  'D',  'E',  'E',  'E',  'E',  'E',  'E',
 'E',  'E',  'E',  'E',  'E',  'E',  'E',  'E',  'E',  'E',
 'F',  'F',  'F',  'F',  'F',  'F',  'F',  'F',  'F',  'F',
 'F',  'F',  'F',  'F',  'F',  'F'
};

static const unsigned char hexEncodeMap2[256] = {
 '0',  '1',  '2',  '3',  '4',  '5',  '6',  '7',  '8',  '9',
 'A',  'B',  'C',  'D',  'E',  'F',  '0',  '1',  '2',  '3',
 '4',  '5',  '6',  '7',  '8',  '9',  'A',  'B',  'C',  'D',
 'E',  'F',  '0',  '1',  '2',  '3',  '4',  '5',  '6',  '7',
 '8',  '9',  'A',  'B',  'C',  'D',  'E',  'F',  '0',  '1',
 '2',  '3',  '4',  '5',  '6',  '7',  '8',  '9',  'A',  'B',
 'C',  'D',  'E',  'F',  '0',  '1',  '2',  '3',  '4',  '5',
 '6',  '7',  '8',  '9',  'A',  'B',  'C',  'D',  'E',  'F',
 '0',  '1',  '2',  '3',  '4',  '5',  '6',  '7',  '8',  '9',
 'A',  'B',  'C',  'D',  'E',  'F',  '0',  '1',  '2',  '3',
 '4',  '5',  '6',  '7',  '8',  '9',  'A',  'B',  'C',  'D',
 'E',  'F',  '0',  '1',  '2',  '3',  '4',  '5',  '6',  '7',
 '8',  '9',  'A',  'B',  'C',  'D',  'E',  'F',  '0',  '1',
 '2',  '3',  '4',  '5',  '6',  '7',  '8',  '9',  'A',  'B',
 'C',  'D',  'E',  'F',  '0',  '1',  '2',  '3',  '4',  '5',
 '6',  '7',  '8',  '9',  'A',  'B',  'C',  'D',  'E',  'F',
 '0',  '1',  '2',  '3',  '4',  '5',  '6',  '7',  '8',  '9',
 'A',  'B',  'C',  'D',  'E',  'F',  '0',  '1',  '2',  '3',
 '4',  '5',  '6',  '7',  '8',  '9',  'A',  'B',  'C',  'D',
 'E',  'F',  '0',  '1',  '2',  '3',  '4',  '5',  '6',  '7',
 '8',  '9',  'A',  'B',  'C',  'D',  'E',  'F',  '0',  '1',
 '2',  '3',  '4',  '5',  '6',  '7',  '8',  '9',  'A',  'B',
 'C',  'D',  'E',  'F',  '0',  '1',  '2',  '3',  '4',  '5',
 '6',  '7',  '8',  '9',  'A',  'B',  'C',  'D',  'E',  'F',
 '0',  '1',  '2',  '3',  '4',  '5',  '6',  '7',  '8',  '9',
 'A',  'B',  'C',  'D',  'E',  'F'
};


int hex_encode(char* dest, const char* str, int len)
{
#ifdef WORDS_BIGENDIAN
	unsigned char *src_ = (unsigned char *) src;
	for (int i =0; i < len; i++) {
		unsigned char v = src_[i];
		dst[i*2] = hextable[v>>4];
		dst[i*2+1] = hextable[v&0x0f];
	}
	dst[len *2] = '\0';
	return len * 2;
#else

    const int buckets = len >> 2; // i.e. i / 4
    const int leftover = len & 0x03; // i.e. i % 4

    uint8_t* p = (uint8_t*) dest;
    uint32_t* srcInt = (uint32_t*) str;
    
	for (int i = 0; i < buckets; ++i) {
		uint32_t x = *srcInt++;
		*p++ = hexEncodeMap1[x & 0xff];
		*p++ = hexEncodeMap2[x & 0xff];
		x >>= 8;
		*p++ = hexEncodeMap1[x & 0xff];
		*p++ = hexEncodeMap2[x & 0xff];
		x >>= 8;
		*p++ = hexEncodeMap1[x & 0xff];
		*p++ = hexEncodeMap2[x & 0xff];
		x >>= 8;
		*p++ = hexEncodeMap1[x];
		*p++ = hexEncodeMap2[x];
	}

	uint8_t * s = (uint8_t *) srcInt;
	uint8_t v;
	switch (leftover) {
	case 3:
		v = *s++;
		*p++ = hexEncodeMap1[v];
		*p++ = hexEncodeMap2[v];
	case 2:
		v = *s++;
		*p++ = hexEncodeMap1[v];
		*p++ = hexEncodeMap2[v];
	case 1:
		v = *s++;
		*p++ = hexEncodeMap1[v];
		*p++ = hexEncodeMap2[v];
	case 0:
	default:
		;
	}

    *p = '\0';
	return len * 2;
#endif
}


int hex_decode(char* dest, const char* str, int len)
{
    int i;

    uint32_t val1, val2;
    uint8_t* p = (uint8_t*) dest;
    uint8_t* s = (uint8_t*) str;

    const int buckets = len >> 2;    // i.e. len / 4
    const int leftover = len & 0x03; // i.e. len % 4
    if (leftover & 0x01) { // i.e if leftover is odd,
                           // leftover==1 || leftover == 3
        return -1;
    }

    // read 4 bytes, output 2.
    // Note on PPC G4, GCC 4.0, it's quite a bit faster to
    // NOT use t0,t1,t2,t3, and just put the *s++ in the gsHexDecodeMap
    // lookup
    uint8_t t0,t1,t2,t3;
    for (i = 0; i < buckets; ++i) {
        t0 = *s++; t1= *s++; t2 = *s++; t3 = *s++;
        val1 = hexDecodeMap1[t0] | hexDecodeMap2[t1];
        val2 = hexDecodeMap1[t2] | hexDecodeMap2[t3];
        if (val1 > 0xff || val2 > 0xff) return -1;
        *p++ = (uint8_t) val1;
        *p++ = (uint8_t) val2;
    }

    if (leftover == 2) {
        val1 = hexDecodeMap1[s[0]] | hexDecodeMap2[s[1]];
        if (val1 > 0xff) return -1;
        *p++ = (uint8_t) val1;
    }

    return (int)(p - (uint8_t*)dest);
}



std::string hex_encode2(std::string &src)
{
 std::stringstream out;
 for ( std::string::size_type i = 0; i < src.size(); i++ ) {
  out << std::hex << std::setfill('0') << std::setw(2)
   << static_cast<unsigned int>(src[i]);
  if ( i < src.size() - 1 )
   out <<' ';
 }
 return out.str();
}
 
std::string hex_decode2(std::string & src)
{
 std::stringstream in (src);
 std::string ret;
 int byte;
 
 while ( in>> std::hex >> std::setfill('0') >> std::setw(2)>> byte )
  ret += static_cast<char>( byte );
 
 return ret;
}
 
 
 
void main()
{
    hexdecodemap();
    hexencodemap();
 
 int a = 100;
 
 char str1[64] = "a-==00~098da,3455.";
 char str2[256] ="";
 
 for (int i =0; i< 1000;i++ )
 {
 hex_encode(str2,str1,18);
 hex_decode(str1,str2,18*2);
 //string  str (str1);
 //cout << str << endl;
 //hex_encode(str2,str1,str.length());
 //str = str2;
 //cout << str << endl;
 
 //string blank ("                      ");
 //copy(blank.begin(),blank.end(),str1);
 //hex_decode(str1,str2,str.length());
 //str = str1;
 //cout << str << endl;
    }
 
 string  str;
 for (int i =0; i< 1000;i++ )
 {
 str = str1;
 //cout << str << endl;
 str = hex_encode2(str);
 copy(str.begin(),str.end(),str2);
 //cout << str << endl;
   
 str = str2;
 str = hex_decode2(str);
 copy(str.begin(),str.end(),str1);
 //cout << str << endl;
 }
 
 
 
  //cout  << a << endl;
 
 //cin >>  a;
 

}







C++ 十六进制字符编码hex encode/decode 和vc2010的性能分析工具（performance analysis）的使用 - widebright - widebright的个人空间
 
C++ 十六进制字符编码hex encode/decode 和vc2010的性能分析工具（performance analysis）的使用 - widebright - widebright的个人空间
 
C++ 十六进制字符编码hex encode/decode 和vc2010的性能分析工具（performance analysis）的使用 - widebright - widebright的个人空间
 
C++ 十六进制字符编码hex encode/decode 和vc2010的性能分析工具（performance analysis）的使用 - widebright - widebright的个人空间
 
C++ 十六进制字符编码hex encode/decode 和vc2010的性能分析工具（performance analysis）的使用 - widebright - widebright的个人空间
 
C++ 十六进制字符编码hex encode/decode 和vc2010的性能分析工具（performance analysis）的使用 - widebright - widebright的个人空间
 
C++ 十六进制字符编码hex encode/decode 和vc2010的性能分析工具（performance analysis）的使用 - widebright - widebright的个人空间
 上面这个图是release版的，其实都是debug版的
 ```
