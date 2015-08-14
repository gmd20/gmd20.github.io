错误时的调用栈
--------------

```
	comctl32.dll!CToolTipsMgr::GetToolAtPoint(struct HWND__ *,int,int,int,int)	Unknown
	comctl32.dll!CToolTipsMgr::ToolAtMessagePos(void)	Unknown
	comctl32.dll!CToolTipsMgr::ShowVirtualBubble(void)	Unknown
	comctl32.dll!CToolTipsMgr::HandleRelayedMessage(struct HWND__ *,unsigned int,unsigned int,long,unsigned int)	Unknown
	comctl32.dll!CToolTipsMgr::ToolTipsWndProc(struct HWND__ *,unsigned int,unsigned int,long)	Unknown
	comctl32.dll!CToolTipsMgr::s_ToolTipsWndProc(struct HWND__ *,unsigned int,unsigned int,long)	Unknown
	user32.dll!__InternalCallWinProc@20()	Unknown
	user32.dll!_UserCallWinProcCheckWow@36()	Unknown
	user32.dll!_SendMessageWorker@24()	Unknown
  user32.dll!_SendMessageW@16()	Unknown
	comctl32.dll!_CallNextSubclassProc@20()	Unknown
	comctl32.dll!_MasterSubclassProc@16()	Unknown
	user32.dll!__InternalCallWinProc@20()	Unknown
	user32.dll!_UserCallWinProcCheckWow@36()	Unknown
	user32.dll!_DispatchMessageWorker@8()	Unknown
	user32.dll!_DispatchMessageW@4()	Unknown
	gvim.exe!00464eeb()	Unknown
```

是鼠标移上去的时候 显示一个 tooltip，也就是 函数原型的 ballon 提示时，出发了windows 10 里面comctl32 控件的动作导致崩溃了。

看上去是 vim里面的一个  'balloonexpr' 'bexpr'  的特性
http://vimdoc.sourceforge.net/htmldoc/options.html#'balloonexpr'

对应的tagbar的代码

```
    if has('balloon_eval')
        setlocal balloonexpr=TagbarBalloonExpr()
        set ballooneval
    endif

function! TagbarBalloonExpr() abort
    let taginfo = s:GetTagInfo(v:beval_lnum, 1)

    if empty(taginfo)
        return ''
    endif

    return taginfo.getPrototype(0)
endfunction
```

这个是很鸡肋的功能， 简单的禁用了这个 ballon eval 功能就可以了。
可以通过这个来设置，不过默认vim是关闭的， 是tagbar 用set ballooneval给打开了的。
看vimrc的最后加上这一句是有效的。
```
set noballooneval
```

更好的办法当然是修复vim 源码中的bug， 让他在 windows 10 不会崩溃。
