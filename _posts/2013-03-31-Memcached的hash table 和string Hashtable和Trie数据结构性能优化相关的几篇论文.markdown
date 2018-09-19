    




  下载LOFTER我的照片书  |
昨天看到微博里面@jametong转的这篇论文
“MemC3: Compact and Concurrent MemCache with Dumber Caching and Smarter Hashing”  Bin Fan, David G. Andersen, Michael Kaminsky   Carnegie Mellon University, Intel Labs

这里面说使用了Optimistic Concurrent Cuckoo Hashing （ cache友好性和乐观锁 （optimistic locking）方面的改进的 Cuckoo Hashing) 让Memcached的性能提高了3倍，而且占用的内存也减少了。
 这个论文的思想主要有两个:
1.    "partial-key cuckoo hashing" 
       通过两个hash函数把key映射到哈希表中的两个位置，每个位置可以存储4个key（根据文档普通的cuckoo hash在容量增加到50%的时候性能就要下降很厉害了。使用4 way的话，可以允许总量增加到hashtable size的90%而容量不会明显下降。cuckoo hash名字来源于杜鹃鸟把蛋下在别的小鸟的窝里面，然后自己孵化出来后就把别人原来小鸟的蛋或者雏鸟顶出窝的行为吧。这个cuckoo也应，他有两个hash函数，当出现碰撞的时候，他就自己霸占旧的位置，让以前的那个存储的建去重新做hash （应该是用另外一个函数函数啦），可能又需要去霸占另外的一个已有的key的位置，然后这过程一直下去，直到找到存储位置。  按照这个文档的选择，如果这个循环次数了500次还没有找到位置的话，就做rehash了，扩展hashtable的 size 才行了。 普通的cuckoo hash 当容量查过50%的时候，这种霸占行为估计要循环多次才能找到位置，所以性能不好。如果使用每个位置 可以保存4个key的 4way方案的话，容量可以允许达到90%吧)
         这个文档的一个优化为了利用cache的特性，另外算出一个1字节大小tag，直接保存在 hashtable 的对应slot里面。这样比较的时候先比较tag是不是匹配了再去比较 key。
      struct  slot {
            char  tag;
            key  * key;
      }
这样避免没必要的 指针取地址的操作，因为很多时候tag比较就不满足了，就没必要再去解析这个key对应的内容了，因为key存储在另外的内存块里面很可能cache不命中，性能就不好了。 特别是碰撞的key又用链表链接在一起的时候，那个要遍历链表，咩个节点都在不同的内存，那就更不好了。下面的论文有说到。
      这个tag是直接存储的，所以不需要地址解析。  这个tag技巧其实上次我也尝试过，比如要比较两个字符串是不是相等的时候，可以把第一字母和字符串长度作为一个tag （这有两个字节只是举例，论文的是一个字节通过hash计算出来的）。如果字符串的第一个字母和长度任何一个不一样，那就不需要再去调用strcmp比较了。大多情况下这种小技巧可以获得很好的效果，这论文里面就是避免访问key所在的内存导致的cache miss了。

2.   version counter的Optimistic Locks 。
      因为大多应用，lookup的操作比update操作要多的多。 这里就是通过引入一个counter来避免读取的时候的锁操作。就是读取之前，先记录一个counter的值，读完之后再获取一个counter的值，看看是不是一样的。如果是一样的那值肯定是对的，如果不一样那要重新再读取一次。  update的操作，修改之前先 把 counter的值加1，修改完之后再加1. 这种更新都是原子操作，避免了锁。
       这个counter不是每个键一个counter，而是为所有的key设置一组counter，数目总共8192个，只能用32KB的内存，然后通过hash函数把这个key都映射到这些counter上面。这样没有一个key一个counter那么消耗内存吧。
      这种version counter的技巧，好像也很常用，比如Linux 内核里面RCU锁好像也有类似的，上次看过。一切无锁设计里面解决ABA问题也是类似的思想？ 
       这个用法应该是观察到大多时候，读取都不需要重新进行的吧。


-----------------------------------------------------------------------------
字符hash table 和 trie内存cache使用的优化。
Cache-Conscious Collision Resolution in String Hash Tables   Nikolas Askitis and Justin Zobel
HAT-trie: A Cache-conscious Trie-based Data Structure for Strings   Nikolas Askitis Ranjan Sinha
Fast and Compact Hash Tables for Integer Keys     Nikolas Askitis

这个说的就是hash table里面，如果有冲突的时候，标准的链表实现是把所有的链在一个链表里面。 这个论文呢，是把所有string的key和value保存在同一个快的内存里面。这样就能够避免遍历链表操作的cache miss吧。看看下面这个原文图示
Memcached的hash table 和string  Hashtable和Trie数据结构性能优化相关的几篇论文 - widebright - widebright的个人空间
 
可以看看下面他的array方式的存储和standard的方式存储的区别。

根据论文的测试，这种array的方式性能很稳定。如果每个slot要保存10个 key的时候，array的方式要比standard和compat的方式要快97%。  但我觉得这种hash table size只是 总数据集数目的十分之一的情况很极端，这样就退化成链表的遍历和数组遍历的比较了。 如果这种情况下大多需要扩展hash tabe的大小，然后rehash了吧。 如果  hashtable size 保证每个slot值保存你一个键值的密度，这个array的方式和standard 链表的方式都是一样的了，性能一样的好了。甚至 标准链表的方式跟快一些。 所以hash table的size还是非常重要的一个参数来的，如果实现用的标准链表的方法来实现的话。  不过这也再次提醒遍历list节点对 cache的影响。

Google的工程师Stochastic Geometry写了一篇文章 “Cache conscious hash tables” 上面有实现相关的代码和这个结构的解释，可以看一下。 http://www.stochasticgeometry.ie/2008/03/29/cache-concious-hash-tables/

stackoverflow的一个帖子提到了HAT-trie 的c语言实现，在github有好几，感兴趣的可以试试
http://stackoverflow.com/questions/3339535/hat-trie-in-ansi-c-implementation

----------------------------------------------------------------------------------------------------

前段时间做过 c++ 里面stirng作为键值的 hashtable 或者map的测试。
       实际应用是  <string> -> <int> 这种key -value对吧。用来做命名计数的，比如统计某个调用用了多长毫秒之类的，访问和更新很频繁。不过总的数目很少，估计很少有超过20个的样子。
        以前的代码有直接用std：：map的，还有就是直接保存到数组里面，然后查找的时候用for循环+strcmp查找的。
我尝试修改使用hash和binary search的方式，不过都没什么效果。  就这种数据是十几个的时候，for循环比较比其他的要快好像。
因为我们接口使用用字符串常量const char *这样的键值来查找，其他的存储数据接口改起来也和你麻烦。我是用hashmap给他给数组加了个index。不过都没看出什么效果。   
        1.  std::map <string, int>  这个的话，因为我们的输入都是const char * 这个估计是map->find的时候需要创建临时string对象，导致性能不好吧。除非修改接口为直接用string来访问，那应该可以避免临时对象才比较好。    
        2. 计算字符串hash值的代码蛮大的。因为元素总数比较小，这个没看出for循环比较有什么的优势。如果是用
对象之类自定义对象作为key的话，还可以缓存一下计算出来的hash值。如果用const char * 这种每次都要计算，在元素数量比小时相对说代价比较大的。

        3. 额外的计算hash值作为tag，或者使用第一个字符+ 字符串长度作为tag，建立索引，然后用二分查找的方式也没什么明显的优势。 
         最后还是保留for循环直接比较，然后strcmp之前先比较一个 第一字符
               for  
                      if  s[0] == s2[i]  && strcmp(s,s2) 
     
       没发现什么更好的方式，也许可以向前面的arary hash table 一样，把这所有的for循环的元素都保存在一个连续的内存块里面，
[string len] [string key] [value ] [string len 2] [string key 2] [value2 ] .....
这样，然后查找的时候，直接可以根据stringlen 跳过比匹配的字符串，跟上面论文提到的方法一样。不知道在 字符粗数目 20个左右的时候相比较其他方法有没有什么优势，有时间测试一下。
