数据库事务系统的一个优化结构，在某些场合比B-tree要好吧。

google 出品的高性能key-value数据库levelDB应该是就是根据这个思想实现的。

 

其实就是一个多级的B-tree吧, 通过延时的“磁盘写和归并排序”来减少磁盘写操作，进而提高系统的整体性能。他把多个随机的磁盘写操作，变成一个批量的连续的磁盘的写操作。

适合于那种插入很频繁，但查找不是很频繁的应用场合吧（后台的归并整理之前的查找应该是有些性能损失的。）。

 

网上可以找到这个pdf文档，文档讲的很清楚，什么原理看上去并不复杂，感兴趣的通许可以去参观一下吧

 

“The Log-Structured Merge-Tree (LSM-Tree)”

Patrick O'Neil1, Edward Cheng2

Dieter Gawlick3, Elizabeth O'Neil1

To be published: Acta Informatica

 

 

摘要部分摘录：

ABSTRACT. High-performance transaction system applications typically insert rows in a

History table to provide an activity trace; at the same time the transaction system generates log

records for purposes of system recovery. Both types of generated information can benefit from

efficient indexing. An example in a well-known setting is the TPC-A benchmark application,

modified to support efficient queries on the History for account activity for specific accounts.

This requires an index by account-id on the fast-growing History table. Unfortunately, standard

disk-based index structures such as the B-tree will effectively double the I/O cost of the

transaction to maintain an index such as this in real time, increasing the total system cost up to

fifty percent. Clearly a method for maintaining a real-time index at low cost is desirable. The

Log-Structured Merge-tree (LSM-tree) is a disk-based data structure designed to provide

low-cost indexing for a file experiencing a high rate of record inserts (and deletes) over an

extended period. The LSM-tree uses an algorithm that defers and batches index changes, cascading

the changes from a memory-based component through one or more disk components in an

efficient manner reminiscent of merge sort. During this process all index values are continuously

accessible to retrievals (aside from very short locking periods), either through the

memory component or one of the disk components. The algorithm has greatly reduced disk arm

movements compared to a traditional access methods such as B-trees, and will improve costperformance

in domains where disk arm costs for inserts with traditional access methods

overwhelm storage media costs. The LSM-tree approach also generalizes to operations other

than insert and delete. However, indexed finds requiring immediate response will lose I/O efficiency

in some cases, so the LSM-tree is most useful in applications where index inserts are

more common than finds that retrieve the entries. This seems to be a common property for

History tables and log files, for example. The conclusions of Section 6 compare the hybrid use

of memory and disk components in the LSM-tree access method with the commonly understood

advantage of the hybrid method to buffer disk pages in memory.
