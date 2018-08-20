bash的叫脚本写起来真是不习惯。

```bash
args="param1 param2 param3"


command1 "$args"  # command 看到1个参数
command1 $args  #   command1 命令里面会看到3个参数 “param1” 、“param2” 、“param3”。
```
看上面这个例子， 后面这个args字符串没有引号的就被展开，分成3个参数传给 command1这个命令或者函数了。



这个在bash脚本里面叫做
[Shell-Expansions](https://www.gnu.org/software/bash/manual/html_node/Shell-Expansions.html#Shell-Expansions),



字符串的分割也是可以通过IFS内置变量来控制怎么分割的，这个叫做
[Word Splitting](https://www.gnu.org/software/bash/manual/html_node/Word-Splitting.html)



比如下面通过设置IFS来吧一个字符分为4个参数展开传个whiptail的用法
```bash
menu_list="1 - : 11111111111:2 - : 2222222222"
IFS=:
whiptail --title "Menu example" --menu "Choose an option" 25 78 16 \
$menu_list
unset IFS
```
另外IFS的分割字符的设置，  应该还会影响 read 读取变量分割，和for $word in $word_list等字符串的分割吧。
反正这个bash很多技巧，用起来真是不熟。



