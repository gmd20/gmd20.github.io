```text
    




  下载LOFTER我的照片书  |
因为只有两个线程，一个线程里面只修改一个head，另外一个修改tail指针，所以没有竞争的问题。x86可以保证写内存操作不会乱序，所以下面的简单时间是没有问题的。
 一开始怀疑代码里面用spinlock+  std::list 做的一个队列是不是锁竞争导致性能问题，不过最后发现耗时都是在网络的read write操作上。这种的无锁的 ringbuffer的实现，在每秒 几千个的情况下根本看不出和spinlock的区别来。
网上抄来的，随便看看吧，以后用的上的时候可以参考一下。
#ifndef RING_BUFFER_H_
#define RING_BUFFER_H_

#include <boost/function.hpp>
#include <boost/smart_ptr/detail/yield_k.hpp>

#ifdef __GNUC__
  #include <unistd.h>
  #define _ReadWriteBarrier  asm volatile ("" : : : "memory");
#else
  #include <intrin.h>
  #pragma intrinsic(_ReadWriteBarrier)
#endif

//a single producer and single consumer lock-free queue
template<typename Element, unsigned int Size>
class RingBuffer
{
public:
	RingBuffer() : tail_(0), head_(0){}
	virtual ~RingBuffer() {}

	bool __Push(Element& item)
	{
		unsigned int next_tail = (tail_ + 1) % Size;
		if(next_tail != head_)
		{
		  array_[tail_] = item;
      _ReadWriteBarrier();
			tail_ = next_tail;
			return true;
		}
		// queue was full
		return false;
	}

	void Push(Element& item) {
		int k = 1;
		while (__Push(item) == false) {
			k <<= 1;
			sched_yield(k);
		}
	}

	bool Pop(Element& item)
	{
		if(head_ == tail_)
			return false;

		item = array_[head_];
    _ReadWriteBarrier();
		head_ = (head_ + 1) % Size;
		return true;
	}

private:

	inline void sched_yield(int k )
	{
		if( k < 256 )
		{
			for (int i=0; i<k; i++) {
				BOOST_SMT_PAUSE
			}
			//__asm  _emit 0xf3;
			//__asm  _emit 0x90;
			//#ifdef __GNUC__
			// __asm__ (".byte 0xf3, 0x90")
			//#endif
		}
		else if( k < 1024 )
		{
#ifdef _WIN32
			Sleep(0);
#else
			usleep(0);
#endif
		}
		else
		{
#ifdef _WIN32
			Sleep(1);
#else
			usleep(500);
#endif
		}
	}

#ifdef _WIN32
  //  allign to cache_line_size(64), avoid false sharing problem
	__declspec (align(64)) volatile unsigned int tail_; // input index
	//char pad1[CACHE_LINE_SIZE - sizeof(int)];;
	__declspec (align(64)) volatile unsigned int head_; // output index
	__declspec (align(64)) Element array_[Size];
#else // c++ 11
	volatile unsigned int tail_  __attribute__ ((aligned (64))); // input index
	volatile unsigned int head_  __attribute__ ((aligned (64))); // output index
	Element array_[Size] __attribute__ ((aligned (64)));
#endif
};

#endif /* RING_BUFFER_H_ */


2015-04-17
注意：   关键字 volatile 是必不可少的，  编译器保证volatile 类型的变量的内存访问不会被重新排序。这样在x86上面cpu也不会对内存的读-读和写-写进行重排序（读操作可能会被重排到写的泄密），程序就能正常工作。
Element  其实也是要定义成volatile，不然编译器可能会把下面这两个内存写操作的顺序调换过来：
   array_[tail_] = item;
   tail_ = next_tail;
 因为如果只是 tail_ 生命了volatile，array_[tail_]  没有声明为array_[tail_] ， 实际测试发现gcc编译器生成代码时就把array_[tail_] = item; 放到 tail_ = next_tail; 指令后面去了，导致程序出问题。只有大压力才会导致程序crash。
  但因为 Element 是c++的模板参数来的，这里直接在前面写volatile  Element array_[Size] __attribute__ ((aligned (64)));，   使用起来很麻烦。如果Element 是复合类型，gcc编译有错误，类型转换时候有麻烦。
  所以只能人为的插入_ReadWriteBarrier  编译器指令，提示编译器不要做内存操作重排，才能比较方便的解决问题。
   
 这个用c++ 11 的 atomic 内存模型来写代码就好看多了，自己会插入内存屏障，禁止编译器优化重排内存访问操作。
参考编译器的问题，微软的msnd里面都提示使用c++ atomic了，不要使用volatile 关键字了。使用volatile 关键字的实现可能依赖编译器特性，好像各个编译器行为还可能不太一样，写起来代码也不好看。
https://msdn.microsoft.com/en-us/library/12a04hfd.aspx
https://gcc.gnu.org/onlinedocs/gcc/Volatiles.html
https://msdn.microsoft.com/en-us/library/f20w0x5e(v=vs.90).aspx

  facebook 开源的c++库folly里面有一个 ProducerConsumerQueue.h   c++ 11标准的风格才是推荐的写法了。   



 2012-06-04   facebook 开源的c++库folly里面有一个 ProducerConsumerQueue.h  实现，也是这个东西来的 
https://github.com/facebook/folly/blob/master/folly/ProducerConsumerQueue.h 


他使用了很多c++的新特性了啊， std::atomic 和 auto变量等，可能跨平台的保证性更好吧。那样写的话。不过他们没考虑false sharing 问题，另外 placement new操作符的使用，在指定内存地址调用构造函数
构建新的对象也是比较好的用法吧。动态申请内存确实太影响性能了。

```
