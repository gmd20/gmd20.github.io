https://github.com/fatih/vim-go

最新的这个版本 vim-go的 保存文件是自动gofmt格式化的功能没起作用，
执行:GoFmt是可以的格式化话，说明gofmt.exe的路径没有问题。

看代码应该是.vim\bundle\vim-go\ftplugin\go.vim 文件里面的
```vim
  autocmd BufWritePre <buffer> call go#auto#fmt_autosave()
```
这一行代码起作用的，但这个好像没有加载。gvim默认集成了C:\Program Files\Vim\vim82\ftplugin\go.vim 了， 好像这个ftplugin之能加载一个文件，
如果把系统自带的这个文件删掉，就会加载 .vim\bundle\vim-go\ftplugin\go.vim ，这个是google官方的https://github.com/google/vim-ft-go 库吧已经
合并到gvim里面去了。


根据https://vim.fandom.com/wiki/File_type_plugins 这里的说法，只要把.vim\bundle\vim-go\ftplugin\go.vim文件最开始did_ftplugin检查注释掉就可以了，想下面这样
```vim
"if exists("b:did_ftplugin")
"  finish
"endif
let b:did_ftplugin = 1
```
改了之后确实可以了。

感觉这个vim-go好像没多用处，语法那些gvim自带的google提供的那些应该足够了。 这个vim-go执行:GoInstallBinaries 会自动 安装一些工具比如gopls吧，会有些:GoInfo之类查看函数声明的用法但感觉用处不大。


