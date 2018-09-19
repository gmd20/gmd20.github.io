```text
在网上大概搜了一下，两个比较好的例子是

nginx 里面 spinlock，另外一个是 boost的smartptr内部实现（boost_1_48_0\boost\smart_ptr\detail\spinlock.hpp)  也有一个比较不错的，支持vc和gcc那些。好像高版本的vc 还几个spinlock的宏，不知道怎么样！ 其实都差不多，基本原理都是这样的：

1）  用原子操作来进行锁的获取。

2）  忙等待的时候，可以让cpu 执行 pause ，这个可以让cpu知道是死循环空等，省点电吧C++ 的 spinlock 实现 - widebright - widebright的个人空间

3） 指数级的状态监测，先等2秒，那是拿不到锁，就等4秒，8秒，16秒之类的，实在不行，最后就切换进程，休眠等吧。

 

nginx的代码，有网友分离了出来  ，直接看这个吧, 我记得是在core目录下的spinlock文件里面，也可以直接去看原版啦

http://bollaxu.iteye.com/blog/870656

void ngx_spinlock(ngx_atomic_t *lock, ngx_atomic_int_t value, ngx_uint_t spin)  
{  
#if (NGX_HAVE_ATOMIC_OPS)  
    ngx_uint_t  i, n;  
    for ( ;; ) {  
        //__sync_bool_compare_and_swap(lock, old, set)  
        //这是原子操作  
        //尝试把lock的值设置成value，如果不为0就说明已经有其他进程获取了该锁  
        if (*lock == 0 && ngx_atomic_cmp_set(lock, 0, value)) {  
            //获取锁成功  
            return;  
        }  
  
        if (ngx_ncpu > 1) {  
            //spin=1024，每次shift一位  
            for (n = 1; n < spin; n <<= 1) {  
                //n=1，2，4，8，16，32，64，128 ...  
                //每次等待时间增加一倍  
                for (i = 0; i < n; i++) {  
                    //__asm__(".byte 0xf3, 0x90")  
                    //__asm__("pause")  
                    //等待一段时间  
                    ngx_cpu_pause();  
                }  
                //等待一段时间再去尝试获取锁  
                if (*lock == 0 && ngx_atomic_cmp_set(lock, 0, value)) {  
                    //获取锁成功  
                    return;  
                }  
            }  
        }  
        //试了这么多次，还是没有获取锁，那就休息一下吧  
        //sched_yield()  
        //usleep(1)  
        ngx_sched_yield();  
    }  
#else  
#if (NGX_THREADS)  
#error ngx_spinlock() or ngx_atomic_cmp_set() are not defined !  
#endif  
#endif  
} 


boost 里面的 windows版的，有网友整理了一下 
http://topic.csdn.net/u/20110521/00/22cf64c6-ffdb-469a-9963-4a9e324d5343.html


void yield(unsigned k)
{
    if (k < 4)
    {
    }
    else if (k < 16)
    {
        _mm_pause();
    }
    else if (k < 32)
    {
        ::Sleep(0);
    }
    else
    {
        ::Sleep(1);
    }
}

class spinlock
{
public:
     spinlock()
        : v_(0)     //网友写的这样初始化也可以？感觉用原子操作来赋值为0才行， boost中是statci变量初始化时直接赋值为0所以不用考虑操作是不是原子的问题。 但好像asio中也有这样的用法，我感觉不够安全啊，还是用原子操作来的安全一点。

    { 

          InterlockedExchange( &v_,0)  ;   // 动态初始化应该使用原子操作 ,像上面那样直接赋值初始化应该有问题吧

    }


    bool try_lock()
    {
        long r = InterlockedExchange(&v_, 1);
        _ReadWriteBarrier();                    // 这个起嘛作用？
        return r == 0;
    }

    void lock()
    {
        for (unsigned k = 0; !try_lock(); ++k)
        {
            yield(k);
        }
    }

    void unlock()
    {
        // 这里没有使用Interlocked系列函数，why?
        _ReadWriteBarrier();                      
        *const_cast<long volatile*>(&v_) = 0;  // const_cast和volatile？ 
    }

private:
    long v_;
};

```
