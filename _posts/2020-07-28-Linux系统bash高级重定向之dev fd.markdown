```bash
[root@localhost ]# cat   <(echo "test")
test
```
bash里面有这种高级用法，把输出直接通过重定向转给另外一个命令，类似管道| 重定向.  
http://www.gnu.org/software/bash/manual/html_node/Redirections.html 
```text
/dev/fd/fd
If fd is a valid integer, file descriptor fd is duplicated.
```
/dev/fd 其实一个软连接指向/proc/self/fd， 如果不存在可以手工创建一下
```text
ln -s /dev/fd  /proc/self/fd
```
