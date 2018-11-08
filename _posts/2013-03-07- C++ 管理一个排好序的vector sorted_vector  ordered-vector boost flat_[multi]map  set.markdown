
打算试用一个排好序的vector来做另外一个 “字符串+ value” 的vector的索引， 其实就是字符串作为key，可以通过fnv32  的hash函数计算key，用于key的比较和索引。

之所以用排好序的vector而不是map，是出于考虑。我觉得vector内存连续比map那种好，可以对排序的vector 做二分查找，跟map的l查找的时间复杂度是一样的（文档说排序的vector查找要比map快一倍。） 应用中的插入和删除操作很少，这样即使排序的vector的插入和删除性能比map要差（O(N) 和 O(log N）的差异)，那也影响不大。

最初是这篇文章比较这两种结构的适用场合和性能差异的，写的比较详细，可以看一下。
Why you shouldn't use set (and what you should use instead)
Matt Austern
 http://lafstern.org/matt/col1.pdf 

Andrei Alexandrescu 在 Modern c++ Design 的一书给出了一个 AssocVector 的实现，底层封装了sorted vector，对外提供和map一样的接口。
http://loki-lib.sourceforge.net/html/a00025.html    需要梯子才能访问

boost里面对应的flat_(multi)map/set associative containers
根据boost的文档http://www.boost.org/doc/libs/1_53_0/doc/html/container/non_standard_containers.html#container.non_standard_containers.flat_xxx
有以下优点
Faster lookup than standard associative containers
Much faster iteration than standard associative containers
Less memory consumption for small objects (and for big objects if shrink_to_fit is used)
Improved cache performance (data is stored in contiguous memory)
Non-stable iterators (iterators are invalidated when inserting and erasing elements)
Non-copyable and non-movable values types can't be stored
Weaker exception safety than standard associative containers (copy/move constructors can throw when shifting values in erasures and insertions)
Slower insertion and erasure than standard associative containers (specially for non-movable types)

facebook 开源的c++库folly里面也有一个实现
 https://github.com/facebook/folly/blob/master/folly/sorted_vector_types.h
根据说明，最好是在元素个数比较少，删除插入操作相对 lookup操作少很多情况下使用比较有优势。

最初是通过http://stackoverflow.com/questions/2710221/is-there-a-sorted-vector-class-which-supports-insert-etc  这里的信息找到上面的资料的
