```text
    




  下载LOFTER我的照片书  |
BM 算法最坏的情况可能比较字符的次数没有KMP算法好，但他最好的时间复杂度可以达到O（n/m) 要比KMP好，在字符集比较大的应用场合一般情况下好像是KMP性能的3到5倍？ 说是是这个BM已经是字符匹配算法性能比较的基准了，算比较的实现了吧，很多文本编辑器的“查找”功能都是使用的这个。


他算法从字符串的末尾向前面开始比较：

abcdaddddddddddddddddd 
abcde
    ^
比如像上面这个字符串，先比较最后一个字符e 和 a，然后不匹配了，怎么去调整指针呢？

这个算法定义了两种规则

第一种叫做 “坏字符规则”

就是不匹配了，然后从 s 串的这个不匹配的字符a去找在 p模式串中的a最后一个位置，然后然后对齐两个比较就可以了。
所以下一步对齐就是

abcdaddddddddddddddddd 
    abcde
    ^
 
这个很容易理解吧，如果这个字符都没有出现过，那直接跳过整个字符串了，
    
abcdaddddddddddddddddd 
bbcde 
    ^
失配的字符a没有在 bbcde 出现过 所有下一次比较就是

abcdaddddddddddddddddd 
     bbcde 
     ^

预先计算模式串的“坏字符规则”数组是很简单的吧，假设他为 F[n]


那么对bbcde   这个串有
F[‘b'] =1   b 出现了两次，保存最后面那个位置
F[‘c'] =2
F[‘d'] =3
F[‘e'] =4
F[‘f'] = -1   字母f没有在bbcde 里面出现过，定义为 -1

大家都知道这个数组怎么计算的了吧！

void prepare_badcharacter_heuristic(const char *str, size_t size, int result[ALPHABET_SIZE])
{
        size_t i;
 
        for (i = 0; i < ALPHABET_SIZE; i++)
                result[i] = -1;
 
        for (i = 0; i < size; i++)
                result[(size_t) str[i]] = i;
}


则是下面这种情况用到。

第二种规则" 最佳后缀" 数组

bbbbadaaa
adadeda
     ^^
    ^
   
    这次查找的时候，a 和 d都匹配了，比较到 'a' 和 ‘e’的时候失配了。这时怎么办呢，如果按照前面的“坏字符规则”，，那个数组里面保存的a 出现的最后一个位置  adadeda里面最后一个出现的a还在 e的后面啊，如果按照前面的规则，那么这样移动指针，指针后退移动了
        
  bbbbadaaa
adadeda
      ^
      
这个指针为负的往后退的用法明显不给力阿，所以这时需要第二个规则 "最佳后缀" 数组

bbbbadaaaaaaaaaaaaa
adadeda
     ^^
    ^
    
这个比较失败后，应该去查找那个已经完全匹配的字串da是不是 在前面出现过了，然后对齐指针，就是这样
   
bbbbadaaaaaaaaaaaaa
    adadeda
     ^^
                
调整后再继续开始比较。 值得注意的是，这次比较里面，da匹配的部分是不是就重复比较了呢，上一次已经比较过了的，可能这就是BM算法子啊坏的情况下比KMP比较次数多的原因吧。


怎么把这个信息计算和保存到一个数组里面，其实是和KMP的跳转数组的计算一样的原理的了，就是计算字符串自身的最大匹配的前缀和后缀。可以参考一下我前面KMP算法里面理解，如果那个理解了，这个应该也是很容易理解的。

还有一种情况需要注意一下，比如 这个bab这个匹配了，但这个在abbab的前面没有出现了，但bab的子串ab 出现了，所以就是把这个子串ab对齐，有点类似 KMP算法里面的全匹配串搜索失败后继续尝试第二个最长的匹配串。
aabababacba
abbab                        
   abbab            
 
整个算法参考文章1里面说的比较清楚吧，如果你理解了KMP里面的跳转数据的计算，那理解这第二个规则 "好的后缀" 也就比较容易了，第一个规则就更简单了。 参考1 和wiki上面的参考2里面有提供有这个算法的实现了，可以去看一下，很容易理解的吧。


===============================
另外很多人觉的某种应用场合（模式串比较短，字符集比较大）BM算法规则二实现比较复杂而且用处不大，他们就省略了这个规则二，再改进了一下规则一，说是获得了比BM更好的性能，感兴趣的可以自己看一下参考文档 3和4，原理和实现非常简单的。

比如一个叫做 D.M. Sunday的人提出的Sunday algorithm（Quick Search algorithm）算法（D.M. Sunday: A Very Fast Substring Search Algorithm. Communications of the ACM, 33, 8, 132-142 (1990)）是这样的，


abcabdaacba        s【m】
bcaab              p 【n】
      bcaab           调整后
^    ^^   

算法是这样工作的，可以按照任何顺序比较两个字符串（那个位置出现概率小，更容易引起失配，算法效果就越好阿），比如从第一个字符开始往后比较，   s[0] = 'a' != p[0] ='b'

那 么算法采用类似BM算法中的第一个“坏字符规则”，查找s串中p长度的后面一个字符，因为不管怎么移动指针，这个字符最后都要落在p里面的，现在就可以看 他是否在 p出现，然后用调整他们对齐的办法来移动指针。比如 s[5] = 'd' 不再 “bcaab” 中出现，那么可以调整指针让 p串对齐到
d的下一个开始的地方。

abcabcaacba        
bcaab            
^    ^
如果是上图，那么s串最后一个字符的下一个是c ，p串中的c 的最后一个出现位置是2，所以调整后是这样的

abcabcaacba        
    bcaab 
     ^
     
真个算法是不是非常简单呢，参考文章3 和4都给出了代码

比如3中给出的代码：

Preprocessing

The occurrence function occ required for the bad-character heuristics is computed in the same way as in the Boyer-Moore algorithm.

Given a pattern p, the following function sundayInitocc computes the occurrence function; it is identical to the function bmInitocc.

void sundayInitocc()
{
    int j;
    char a;

    for (a=0; a<alphabetsize; a++)
        occ[a]=-1;

    for (j=0; j<m; j++)
    {
        a=p[j];
        occ[a]=j;
    }
}

 
Searching algorithm

Using a function matchesAt that compares the pattern with the text window in a certain manner depending on the implementation, the searching algorithm looks as follows:

void sundaySearch()
{
    int i=0;
    while (i<=n-m)
    {
        if (matchesAt(i)) report(i);
        i+=m;
        if (i<n) i-=occ[t[i]];
    }
}

After statement i+=m, it is necessary to check if the value of i is at most n-1, since subsequently t[i] is accessed. 




参考文档
1. Boyer-Moore algorithm   http://www.inf.fh-flensburg.de/lang/algorithmen/pattern/bmen.htm 我把他转载到这个空间了
                        http://www.inf.fh-flensburg.de/lang/algorithmen/pattern/bmen.htm
2. Boyer–Moore string search algorithm http://en.wikipedia.org/wiki/Boyer%E2%80%93Moore_string_search_algorithm

3. Sunday algorithm  http://www.inf.fh-flensburg.de/lang/algorithmen/pattern/sundayen.htm

4. Quick Search algorithm http://www-igm.univ-mlv.fr/~lecroq/string/node19.html#SECTION00190

5. http://www-igm.univ-mlv.fr/~lecroq/string/   这个网页列出了各种各样的字符串的匹配算法，并且分析算法时间复杂度等比较，还给出了各种算法实现代码，非常全面。
```
