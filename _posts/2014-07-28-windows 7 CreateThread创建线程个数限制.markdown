程序采用CreateThread 函数创建线程
http://msdn.microsoft.com/en-us/library/windows/desktop/ms682453(v=vs.85).aspx

createthread函数失败，调用 GetLastError获取错误，是内存不足。
根据msdn的说明，这个线程个数只受可用内存限制。还有系统默认stacksize的大小有关。系统默认的stacksize是1M，所以32位系统最多可以创建2G/1M  = 2000个左右的线程。  这个默认stacksize 是程序编译时的linker选项指定。
CreateThread函数 dwStackSize  参数指定了这个线程的栈的大小，0就采用系统默认1M的大小。

注意的一点是进程的stack 占用的虚拟空间大小，在任务管理器里面看不出来。需要使用vmmap工具查看才行。所以看起来任务管理器里面内存占用还是很少，但Createthread就失败了，有点奇怪。
windows 7 CreateThread创建线程个数限制 - widebright - widebright的个人空间
 

写个检查的测试程序：
```text
DWORD WINAPI ThreadProc(
	_In_  LPVOID lpParameter
	)
{
	Sleep(1000000);
	return 0;
}

int main(int, char**)
{	
	DWORD ThreadId;
	for (int num = 0; num < 10000; num++) {
		HANDLE _thread = CreateThread(NULL, 0, ThreadProc, NULL, 0, &ThreadId);
		if (_thread == NULL) break;
		std::cout << num << std::endl;
	}	

	int ddd;
	cin >> ddd;
	return ddd;
	}
  ```
windows 7 64位机器上面测试
32位程序，大概能创建1500个线程。（2G的用户空间，还是需要用作其他用途的，如果其他部分占用了比较多的内存，可以创建的就更少了）。
64位程序，可以创建1万个线程左右。
其实程序不应该建立这么多的线程的，跟cpu个数一样多就好了吧。这个出问题的程序是采用第3方的库，采用每个http连接一个线程的方式，很是不好的做法。
