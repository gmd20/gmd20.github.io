```text
html和xml里面是有类似这样的escape string的编码转换的。
比如xml的
/**
 * & --> &amp;
 * < --> &lt;
 * > --> &gt;
 * " --> &quot;
 * ' --> &apos;
 */

这个转换，很容易成为程序的性能热点，现在的一个模块就是这样。
之前刚好看到类似问题相关的文章，escape string的转换相关函数优化之后，响应时间好很多。
When the database is fast enough  http://blog.iconfinder.com/when-the-database-is-fast-enough/
Github.com blog    的Escape Velocity  https://github.com/blog/1475-escape-velocity
github为了解决这个转换问题而建立的开源库houdini  https://github.com/vmg/houdini/

看了一下代码，还是encode的转换，主要是通过lookup table的查表方式来优化的。decode是使用gperf来生成 “完美哈希函数”来实现。 可以参考一下代码的实现，不错！

GNU gperf is a perfect hash function generator
http://www.gnu.org/software/gperf/
这个可以为指定的预知字符串列表生成所谓的“完美哈希函数”，自动生成查找函数，不会有冲突键的存在。第一次了解。在某些场合下面使用应该是挺合适的。 
下面这个文章有点介绍
http://blogs.ejb.cc/archives/5997/gperf-and-perfect-hash-function

就是根据 https://github.com/vmg/houdini/blob/master/html_unescape.gperf 这样的配置，自动生成c函数代码了，和flex和bison那些有点类似用法。

我自己做一下小小的修改：


class small_buffer
{
public:
	small_buffer():size_(0),extra_buf_((char*) &data_)
	{
	
	}
	~small_buffer()
	{
		if(extra_buf_ != (char *) &data_) {
			delete [] extra_buf_;
		}			
	}
	void buf_grow(unsigned int size)
	{
		if (size > 1024) {
			extra_buf_ = new char[size];
		}
	}
	void put(const void * b, unsigned int len)
	{
		memcpy(extra_buf_ + size_, b, len);
		size_ += len;
	}
	void putc(char c)
	{
		extra_buf_[size_++] = c; 
	}
	void puts(const char * string)
	{
		put(string, strlen(string));
	}
	char * data() 
	{
		return extra_buf_;
	}
	unsigned int size()
	{
		return size_;	
	}
private:
	char data_[1024];
	unsigned int size_;
	char *extra_buf_;
};



bool
unescape_xml(small_buffer &ob, const char *src, unsigned int size)
{
	unsigned int i = 0;
	unsigned int start = 0;
	while (i < size) {
		if (src[i] != '&') {
			i++;
			continue;
		} else {
			goto REAL_WORK;
		}
	}
	return false;

REAL_WORK:
	ob.buf_grow(size);

MATCH_LOOP:

	char code = 0;
	unsigned int len = 0;
	do {
		if (i + 3 < size) {
			unsigned int key =  *(unsigned int *) &src[i];
			if (key ==  ('&' | 'l' << 8 | 't' << 16 | ';' << 24)) {
				code = '<';
				len = 4;
				break;
			} 
			if (key == ('&' | 'g' << 8 | 't' << 16 | ';' << 24)) {
				code = '>';
				len = 4;
				break;
			}
		} else {
			break;
		} 

		if (i + 4 < size) {
			unsigned int key =  *(unsigned int *) &src[i+1];
			if (key ==  ('a' | 'm' << 8 | 'p' << 16 | ';' << 24) ) {
				code = '&';
				len = 5;
				break;
			}
		} else {
			break;
		}

		if (i + 5 < size && src[i+5] == ';') {
			unsigned int key =  *(unsigned int *) &src[i+1];
			if (key == ('q' | 'u' << 8 | 'o' << 16 | 't' << 24)) {
				code = '"';
				len = 6;
				break;
			}
			if (key == ('a' | 'p' << 8 | 'o' << 16 | 's' << 24)) {
				code = '\'';
				len = 6;
				break;
			}
		} else {
			break;
		}
	} while (0);

	if (code) {
		if (i > start) {
			ob.put(&src[start], i - start);
		}
		ob.putc(code);
		i += len;
		start = i;
	} else {
		i++;
	}

	while (i < size) {
		if (src[i] != '&') {
			i++;
			continue;
		} else {
			goto MATCH_LOOP;
		}
	}

	if (i > start) {
		ob.put(&src[start], i - start);
	}

	return true;
}


/**
 * & --> &amp;
 * < --> &lt;
 * > --> &gt;
 * " --> &quot;
 * ' --> &apos;
 */
static const char *LOOKUP_CODES[] = {
	"", /* reserved: use literal single character */
	"", /* unused */
	"", /* reserved: 2 character UTF-8 */
	"", /* reserved: 3 character UTF-8 */
	"", /* reserved: 4 character UTF-8 */
	"?", /* invalid UTF-8 character */
	"&quot;",
	"&amp;",
	"&apos;",
	"&lt;",
	"&gt;"
};

static const unsigned char CODE_INVALID = 5;

static const unsigned char XML_LOOKUP_TABLE[] = {
	/* ASCII: 0xxxxxxx */
	5, 5, 5, 5, 5, 5, 5, 5, 5, 0, 0, 5, 5, 0, 5, 5,
	5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5,
	0, 0, 6, 0, 0, 0, 7, 8, 0, 0, 0, 0, 0, 0, 0, 0,
	0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 9, 0,10, 0,
	0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
	0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
	0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
	0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,

	/* Invalid UTF-8 char start: 10xxxxxx */
	5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5,
	5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5,
	5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5,
	5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5,

	/* Multibyte UTF-8 */

	/* 2 bytes: 110xxxxx */
	2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2,
	2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2,

	/* 3 bytes: 1110xxxx */
	3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3,

	/* 4 bytes: 11110xxx */
	4, 4, 4, 4, 4, 4, 4, 4,

	/* Invalid UTF-8: 11111xxx */
	5, 5, 5, 5, 5, 5, 5, 5,
};



// ---------------------------------------
// if escape string was found in the sr buffer, ob will be set  and  return true 
// else ob will not set and return false
//----------------------------------------
bool
escape_xml(small_buffer &ob, const unsigned char *src, unsigned int size)
{
	unsigned int i = 0;
	unsigned char code = 0;
	
	// most of the time no replacement is needed,  optimize for the fast path
	while (i < size) {
		unsigned int byte;
		byte = src[i++];
		code = XML_LOOKUP_TABLE[byte];
		if (code) {
			goto REAL_WORK;
		}
	}
	return false;

REAL_WORK:
	i--;
	ob.buf_grow(size * 8);
	if (i > 0) {
		ob.put(src,i);
	}

	while (i < size) {
		unsigned int start, end;
		start = end = i;

		while (i < size) {
			unsigned int byte;

			byte = src[i++];
			code = XML_LOOKUP_TABLE[byte];

			if (!code) {
				/* single character used literally */
			} else if (code >= CODE_INVALID) {
				break; /* insert lookup code string */
			} else if (code > size - end) {
				code = CODE_INVALID; /* truncated UTF-8 character */
				break;
			} else {
				unsigned int chr = byte & (0xff >> code);

				while (--code) {
					byte = src[i++];
					if ((byte & 0xc0) != 0x80) {
						code = CODE_INVALID;
						break;
					}
					chr = (chr << 6) + (byte & 0x3f);
				}

				switch (i - end) {
					case 2:
						if (chr < 0x80)
							code = CODE_INVALID;
						break;
					case 3:
						if (chr < 0x800 ||
							(chr > 0xd7ff && chr < 0xe000) ||
							chr > 0xfffd)
							code = CODE_INVALID;
						break;
					case 4:
						if (chr < 0x10000 || chr > 0x10ffff)
							code = CODE_INVALID;
						break;
					default:
						break;
				}
				if (code == CODE_INVALID)
					break;
			}

			end = i;
		}

		if (end > start)
			ob.put(src + start, end - start);

		/* escaping */
		if (end >= size)
			break;

		ob.puts(LOOKUP_CODES[code]);
	}

	return true;
}


使用例子
int main (int, char**)
{
	int a = 1;
restart:
	small_buffer buf;
	small_buffer buf2;
	std::string s;
	std::cin >> s;
	
	if (escape_xml(buf, (unsigned char *)s.c_str(),s.size())) {
		s.assign(buf.data(),buf.size());
	}
	std::cout << s << std::endl;

	if (unescape_xml(buf2,s.c_str(),s.size())) {
		s.assign(buf2.data(),buf2.size());
	}
	std::cout << s << std::endl;


	goto restart;
	return 0;

}

```
