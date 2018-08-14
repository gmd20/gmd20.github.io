想golang里面的log是很容易位置输出文件的， c程序的话需要重定向标准输出流sstdout和错误输出流到stderr
其实也很容易，关键是 dup2 和 freopen 函数。


```c
        freopen("stdout.txt","w", stdout);
        freopen("stderr.txt","w", stderr);        
```

 freopen("stdout.txt","w", stdout);应该等价于 
 ```c
     int new_fd = open("stdout.txt", O_RDWR);
     dup2(new_fd, fileno(stdout));
```



但很奇怪下面这个写法是不行的 
```c
        // redirect stdout to file
        int origin_stdout = dup(fileno(stdout));   // 如果多了这个 dup 之后，freopen 就不会成功了
        int origin_stderr = dup(fileno(stderr));
        freopen("stdout.txt","w", stdout);
        freopen("stderr.txt","w", stderr);
           
        // revert
        dup2(origin_stdout, fileno(stdout));
        dup2(origin_stderr, fileno(stderr));
        close(origin_stdout);
        close(origin_stderr);
        
```

换成下面这个写法 ， 如果少了 printf ("\n");  也是不行的，好像stdout需要初始化
```c
        // redirect stdout to file
        int origin_stdout = dup(fileno(stdout));    // dup之后，需要写点东西再 dup2 才会成功重定向
        int origin_stderr = dup(fileno(stderr));

        int new_fd = open("/dev/null", O_RDWR);
        // int dev_null = open("11111.txt", O_RDWR|O_CREAT);
        
        // printf("\n");   // 如果少了这一行也不行，非常奇怪，好像这个写来触发stdout的file结构的copy-on-write才行。不然dup2虽然返回成功但实际stdout没有重定向
        fputc(0, stdout); // 如果少了这一行也不行，非常奇怪，好像这个写来触发stdout的file结构的copy-on-write才行。不然dup2虽然返回成功但实际stdout没有重定向
        fputc(0, stderr);

        dup2(new_fd, fileno(stdout));
        dup2(new_fd, fileno(stderr));

           
        // revert
        dup2(origin_stdout, fileno(stdout));
        dup2(origin_stderr, fileno(stderr));
        close(origin_stdout);
        close(origin_stderr);
        close(new_fd);
        printf ("ok\n");


```
