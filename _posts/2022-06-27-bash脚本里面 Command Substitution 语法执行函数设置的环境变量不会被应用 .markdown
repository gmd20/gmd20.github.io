ouput=$(command)        # 执行命令，把输出赋值给output变量   
output=$(func1 parm1)   # 执行函数，把输出赋值给output变量   


这种 Command Substitution 是在 “ subshell environment‘ 里面执行， 估计是在子进程还是什么环境里面执行的，所以设置的环境变量   
不会被当前环境继承。   只能改成 不用这种方式执行函数了，直接func1 parm1 调用，然后子啊func1里面使用全局变量来传递返回值。   
shell的脚本总是会遇到很奇怪的问题。
