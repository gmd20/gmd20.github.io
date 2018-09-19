一个例子 。

 

class  A {

public: 

   virtual  void print () {

        cout  << "This is A" << endl;

    };

};

 

class B: public A {

public: 

   virtual  void print () {

        cout  << "This is B" << endl;

    };

};

 

void print( A & a)

{

      a.print();

}

 

void  test ( A &  a) 

{

      boost::bind(print, a);

}

 

B  b;

B & ref_b = b;

 

test ( ref_b);   ////////////// 这个将打印"This is A"  ， 因为 boost::bind 返回的构造函数会把这种引用参数当作值来对待，所以他这时构建出来的 function对象里面包含个  A 类的对象，你写引用的时候，他按照值来构建一个新的对象的存在函数对象里面的。这个有点奇怪啊，和普通的传引用对比的话，因为我之前用的不多，结果这个地方理解错误了，导致了一个bug，到了最后调用虚函数时不对才发现原来是这样的。

      解决办法，是也简单，改成传指针就可以了，他就不会复制整个对象到函数对象里面去了呃。或者显式的写boost:;ref  来表示引用 ，把上面一句改成   boost::bind(print,   boost::ref( a)); 就可以了。 为什么他要求这种显式的boost::ref  boost::cref 的写法呢？ 我估计是  c++ 模板里面区分不了是不是引用的类型？ 

 

 

boost的 bind 模块对这个文档有说，值注意的是，这个 bind构建出来的函数对象，默认也是按照值来复制的，如果你想得到引用，还是要 显式的使用 boost::ref才行。

 

--------------------------

http://www.boost.org/doc/libs/1_49_0/libs/bind/bind.html

 

a copy of the value of i is stored into the function object. boost::ref and boost::cref can be used to make the function object store a reference to an object, rather than a copy:

 

int i = 5;

 

boost::bind(f, boost::ref(i), _1);

 

boost::bind(f, boost::cref(42), _1);     /// const 引用

 

-------------------------

By default, bind makes a copy of the provided function object. boost::ref and boost::cref can be used to make it store a reference to the function object, rather than a copy. This can be useful when the function object is noncopyable, expensive to copy, or contains state; of course, in this case the programmer is expected to ensure that the function object is not destroyed while it's still being used.

 

struct F2

{

    int s;

 

    typedef void result_type;

    void operator()( int x ) { s += x; }

};

 

F2 f2 = { 0 };

int a[] = { 1, 2, 3 };

 

std::for_each( a, a+3, bind( ref(f2), _1 ) );

 

assert( f2.s == 6 );

Using bind with

------------------------------

 

 

http://www.boost.org/doc/libs/1_49_0/doc/html/ref.html

 

The expression boost::ref(x) returns a boost::reference_wrapper<X>(x) where X is the type of x. Similarly, boost::cref(x) returns a boost::reference_wrapper<X const>(x).
