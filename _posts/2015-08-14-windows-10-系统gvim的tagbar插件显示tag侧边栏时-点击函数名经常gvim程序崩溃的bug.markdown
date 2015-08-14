

错误时的调用栈
--------------


是鼠标移上去的时候 显示一个 tooltip，也就是 函数原型的 ballon 提示时，出发了windows 10 里面comctl32 控件的动作导致崩溃了。



看上去是 vim里面的一个  'balloonexpr' 'bexpr'  的特性
http://vimdoc.sourceforge.net/htmldoc/options.html#'balloonexpr'



对应的tagbar的代码

```
    if has('balloon_eval')  检查是否编译了这个特性
        setlocal balloonexpr=TagbarBalloonExpr()
        set ballooneval  他这里 打开了 开关
    endif


" TagbarBalloonExpr() {{{2
function! TagbarBalloonExpr() abort
    let taginfo = s:GetTagInfo(v:beval_lnum, 1)

    if empty(taginfo)
        return ''
    endif

    return taginfo.getPrototype(0)
endfunction
```



这个是很鸡肋的功能， 简单的禁用了这个 ballon eval 功能就可以了。  
可以通过这个来设置，不过默认vim是关闭的，  是tagbar给打开了的。
看vimrc的最后加上这一句是有效的。
```
set noballooneval
```


更好的办法当然是修复vim 源码中的bug， 让他在 windows 10 不会崩溃。 



