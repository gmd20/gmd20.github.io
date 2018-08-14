想golang里面的log是很容易位置输出文件的， c程序的话需要重定向标准输出流sstdout和错误输出流到stderr
其实也很容易，关键是 dup2 和 freopen 函数。

```c
        // redirect stdout to file
        int origin_stdout = dup(fileno(stdout));
        int origin_stderr = dup(fileno(stderr));
        freopen("stdout.txt","w", stdout);
        freopen("stderr.txt","w", stderr);
        
        // revert
        dup2(origin_stdout, fileno(stdout));
        dup2(origin_stderr, fileno(stderr));
        fclose(origin_stdout);
        fclose(origin_stderr);
        
```
