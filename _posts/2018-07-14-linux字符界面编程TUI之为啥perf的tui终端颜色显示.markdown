1.  发现测试机上的perf top显示的没有颜色高亮，看起来很不舒服。
看一下原本的centos7的机器，有颜色显示，看起来就好看了。    



2.  之前了解到Ncurses这个库，专门用来开发字符终端界面的。功能很强大，好像还有一个CDK（develop kit） 做了更多控件包装。感兴趣的可以看一下。   
https://en.wikipedia.org/wiki/Ncurses




3.  不过perf用的不是ncurses， 网上有人说用的是newt，另外一个字符界面编程库。    
https://en.wikipedia.org/wiki/Newt_(programming_library)



4.  不过看perf实际的依赖的动态库，用的是S-Lang。  另外一个辅助字符界面开发的库。   
https://en.wikipedia.org/wiki/S-Lang



5.  为啥这个s-lang的虚拟终端上颜色显示不出来呢？ 之能查看源代码了。      
https://elixir.bootlin.com/linux/latest/source/tools/perf/ui/tui/setup.c
```c

int ui__init(void)
{
	int err;

	SLutf8_enable(-1);
	SLtt_get_terminfo();
	SLtt_get_screen_size();

	err = SLsmg_init_smg();
	if (err < 0)
		goto out;
	err = SLang_init_tty(-1, 0, 0);
	if (err < 0)
		goto out;

	err = SLkp_init();
	if (err < 0) {
		pr_err("TUI initialization failed.\n");
		goto out;
	}

	SLkp_define_keysym((char *)"^(kB)", SL_KEY_UNTAB);

	signal(SIGSEGV, ui__signal_backtrace);
	signal(SIGFPE, ui__signal_backtrace);
	signal(SIGINT, ui__signal);
	signal(SIGQUIT, ui__signal);
	signal(SIGTERM, ui__signal);

	perf_error__register(&perf_tui_eops);

	ui_helpline__init();
	ui_browser__init();
	tui_progress__init();

	hist_browser__init_hpp();
out:
	return err;
}

```



看这个 SLtt_get_terminfo 函数好像是干这个事情的，查看一下文档说明：       
http://www.jedsoft.org/slang/doc/html/cslang-8.html
```text
Of course not all terminals are color terminals. If the S-Lang global variable SLtt_Use_Ansi_Colors is non-zero, the terminal is assumed to be a color terminal. The SLtt_get_terminfo will try to determine whether or not the terminal supports colors and set this variable accordingly. It does this by looking for the capability in the terminfo/termcap database. Unfortunately many Unix databases lack this information and so the SLtt_get_terminfo routine will check whether or not the environment variable COLORTERM exists. If it exists, the terminal will be assumed to support ANSI colors and SLtt_Use_Ansi_Colors will be set to one. Nevertheless, the application should provide some other mechanism to set this variable, e.g., via a command line parameter.

When the SLtt_Use_Ansi_Colors variable is zero, all objects with numbers greater than one will be displayed in inverse video
```


6. 很明显s-lang在找 一个叫做 “terminfo/termcap database.” 的数据库。好吧这个到底是一个什么东西呢？     
http://www.tldp.org/HOWTO/Text-Terminal-HOWTO-16.html      
从说明看提供了一个颜色表等终端配置d的文件吧，执行 “infocmp"” 看看当前的终端使用是哪个配置文件。
```text
[root@localhost]# echo $TERM
xterm
[root@localhost]# infocmp
#	Reconstructed via infocmp from file: /usr/share/terminfo/x/xterm
xterm|xterm terminal emulator (X Window System),

[root@localhost ]# rpm -qf  /usr/share/terminfo/v/vs100
ncurses-base-5.9-14.20130511.el7_4.noarch
```
这个终端类型应该是你的xshell等模拟终端里面设置的吧。     
这个配置文件 /usr/share/terminfo/x/xterm 就是对应终端的配置了吧。       
从一个显示正常的机器把  /usr/share/terminfo/x/xterm  文件复制到测试机去，  
perf top -g 显示出来的TUI有颜色高亮，这下看起来和操作爽多了。     
/usr/share/terminfo/ 下面还有linux，vt100 等各种终端的配置文件，怪不得叫做 数据库。   




