```text
现在认为Murmurhash3 和CityHash 还有SpookyHash都比fnv这个算法要快吧。参考http://gmd20.blog.163.com/blog/static/16843923201362985128582/





Trie 就是字典树

http://en.wikipedia.org/wiki/Trie

里面有个rie和 red black tree （红黑树）和hash table的性能比较的图，也有一些特点对比的介绍



Gnu  c++ std 的实现里面的容器

http://gcc.gnu.org/onlinedocs/libstdc++/ext/pb_ds/trie_based_containers.html

------------------------------------------

Radix tree

http://en.wikipedia.org/wiki/Radix_tree



Trie 的空间优化版本，这里又有和hash table的时间复杂度比较。

------------------------------------------------

HAT-trie:

 A Cache-conscious Trie-based Data Structure for Strings

http://crpit.com/confpapers/CRPITV62Askitis.pdf



wiki提到是 Radix tree的变异，一个cache 友好的Trie。按照论文文档，速度和空间占有和hash table不相上下。



https://github.com/chris-vaszauskas/hat-trie/tree/master/src

这里有个开源std容器风格的C++ 模板实现，是GPL协议的。

--------------------------------------------------------

hash table



c++ 库里面的hash map 或者 std::unordered_map 

hash map不是标准的std，各个厂家自己的实现，std::unordered_map 是后来标准引入的。

相比较map基于红黑树的实现，std::unordered_map就是使用hash函数来做的了。因为红黑树里面节点是有序的，所以这个无序的hash的就叫做nordered_map了。如果忽略碰撞等因素hash的查找应该比红黑树的性能要好点的。



hash_map Class

http://msdn.microsoft.com/en-us/library/0d462wfh(v=vs.71)

unordered_map Class

http://msdn.microsoft.com/en-us/library/bb982522.aspx

unordered_map

http://www.cplusplus.com/reference/stl/unordered_map/



facebook发布的folly c++库（https://github.com/facebook/folly/tree/master/folly）里面也有

AtomicHashMap.h

AtomicHashArray.h

等相关的，这个应该是支持多线程的无锁实现来的。



字符串 hash 函数

https://github.com/facebook/folly/blob/master/folly/Hash.h  这里里面有也有fnv32  fnv32_buf这个字符串哈希函数的实现代码。 这个说是最好的字符串哈希函数了。





根据网上 这篇文章 "http://programmers.stackexchange.com/questions/49550 ”（靠，百度弱智的过滤，标题和链接发布出来，google一下吧   完整的链接是后面这个地址，把“和 ”替换为英文，  http://programmers.stackexchange.com/questions/49550/which-hashing-algorithm-is-best-for-uniqueness-和-speed    ）的评测，根据随机性（碰撞少的）和速度比较，最好的哈希函数是  FNV-1a 

“Fowler–Noll–Vo hash function”

http://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function

文章后来又更新了，认为最好的是Murmurhash3，这个要比fnv要快。

--------------------------------------------



vc2010中使用字符串哈希函数，

 std::hash_map 

 std::tr1::unorder_map

 std::unorder_map 

使用的应该都是这个函数来的 在  <xfunctional> 头文件里面定义

template<>

class hash<_STD string>

: public unary_function<_STD string, size_t>

{// hash functor

public:

typedef _STD string _Kty;



size_t operator()(const _Kty& _Keyval) const

{// hash _Keyval to size_t value by pseudorandomizing transform

size_t _Val = 2166136261U;

size_t _First = 0;

size_t _Last = _Keyval.size();

size_t _Stride = 1 + _Last / 10;



for(; _First < _Last; _First += _Stride)

_Val = 16777619U * _Val ^ (size_t)_Keyval[_First];

return (_Val);

}

};



微软的这个算法用的也是fnv-1 来的。参考下面说明。

---------------------------------------------------------------------------



这个face,book中开源的folly库里面的实现





/*

 * Fowler / Noll / Vo (FNV) Hash

 *     http://www.isthe.com/chongo/tech/comp/fnv/

 */



//根据上面的网页说明，下面代码里面的 FNV_32_HASH_START的值应该为2166136261UL 才对吧？？不知道facebook代码里面是写错了还是怎么的。

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



inline uint32_t fnv32(const std::string& str,

                      uint64_t hash = FNV_32_HASH_START) {

  return fnv32_buf(str.data(), str.size(), hash);

}



inline uint64_t fnv64(const char* s,

                      uint64_t hash = FNV_64_HASH_START) {

  for (; *s; ++s) {

    hash += (hash << 1) + (hash << 4) + (hash << 5) + (hash << 7) +

      (hash << 8) + (hash << 40);

    hash ^= *s;

  }

  return hash;

}



inline uint64_t fnv64_buf(const void* buf,

                          int n,

                          uint64_t hash = FNV_64_HASH_START) {

  const char* char_buf = reinterpret_cast<const char*>(buf);



  for (int i = 0; i < n; ++i) {

    hash += (hash << 1) + (hash << 4) + (hash << 5) + (hash << 7) +

      (hash << 8) + (hash << 40);

    hash ^= char_buf[i];

  }

  return hash;

}



inline uint64_t fnv64(const std::string& str,

                      uint64_t hash = FNV_64_HASH_START) {

  return fnv64_buf(str.data(), str.size(), hash);

}







===============================

综合考虑一下，我的 16个长度的字符串的查找，直接用标准库里面的 unorder_map应该就可以了吧。 比trie的实现要简单一些，大多评测也都说hash的办法要比trie的性能要好。但trie的hat-tire也有它的优势了，平均性能比较稳定，可顺序遍历等。有时应该可以考虑结合两种办法，比如Linux内核里面的路由表的查找，就结合了hash 和trie两种办法了。



===============================

上面 微软的hash 函数也是 FNV-1 来的，

    hash += (hash << 1) + (hash << 4) + (hash << 7) +

            (hash << 8) + (hash << 24);

  是

 hash  =  hash  * 16777619U 的优化写法，

因为这个质数16777619U = (0x01000000  +256+128+16+2+1 ，所以展开变成移位就是上面那个。



关于FNV-1  和 FNV-1a的原理和各种实现代码参考这个网页，包含gcc优化，汇编优化等版本

http://www.isthe.com/chongo/tech/comp/fnv/index.html



伪码如下

----------fnv-1 ----------------

hash = offset_basis

for each octet_of_data to be hashed

 hash = hash * FNV_prime

 hash = hash xor octet_of_data

return hash



----------fnv-1a-----------------

hash = offset_basis

for each octet_of_data to be hashed

 hash = hash xor octet_of_data

 hash = hash * FNV_prime

return hash

----------------------------------

offset_basis 和FNV_prime 参数选择参考 http://www.isthe.com/chongo/tech/comp/fnv/index.html#FNV-param

两者只是 异或操作和 乘法的顺序不一样，说是fnv-1a要快一些，推荐使用fnv-1a

```
