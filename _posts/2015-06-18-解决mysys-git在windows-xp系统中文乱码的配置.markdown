修改mingw64  git的全局配置
-------------------
* c:\git\mingw64\etc\gitconfig
* 或者 c:\users\<username>\.gitconfig

```
[core]
  quotepath = off
  editor = gvim

[gui]
    #代码库统一用urf-8
    encoding = utf-8
[i18n]
    #设置 commit log 提交时使用 utf-8 编码致
    commitencoding = utf-8
    #使得在 $ git log 时将 utf-8 编码转换成 gbk 编码，解决Msys bash中git log乱码
    logoutputencoding = gbk
```

修改msys  bash的配置
-------------------
* c:\git\etc\profile
```bash
#ls能够正常显示中文
alias ls='ls --show-control-chars --color=auto'
```
