
```text
http://antirez.com/news/75



对HyperLogLog数据结构做了一些说明，和一些参考文档。

通过扔硬币的正反面来说明这个原理，比如，如果你告诉我你扔硬币看到正面的次数是10次，那么我就可以大概估计你扔的次数是20次。 如果这个数值比较到大，那么是结果是比较准确的。

HyperLogLog 就是把这个element 做哈希，然后统计所有hash得到的数值 高位开始0标志位占整个的比例，就可以大概估计总的数目。这就是整个空间不重复数据的数目。



提到了Google的实现的论文

HyperLogLog in Practice: Algorithmic Engineering of a State of The Art Cardinality Estimation Algorithm

[1] http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf
[2] http://static.googleusercontent.com/media/research.google.com/en//pubs/archive/40671.pdf


我之前也有发过一篇文章了。

基数估计算法的相关资料

http://gmd20.blog.163.com/blog/static/1684392320130523843285/

```
