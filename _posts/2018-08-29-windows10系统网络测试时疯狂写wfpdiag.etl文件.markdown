做网络测试时，发现windows10在疯狂的写这个磁盘，每秒20MB左右，这个文件只有128kb
C:\ProgramData\Microsoft\Windows\wfp\wfpdiag.etl

搜了一下应该是防火墙在记录日志，我测试时虚拟机用了很多IP，好像是这样触发他什么规则记录日志了。不过也太疯狂了，
通过下面这个命令可以禁止它。
netsh wfp set options netevents = off
