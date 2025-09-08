运行虚拟机时，提示获取不到显示器的分辨率。
```text
unsupported resolution for screen shot: 0x0 (screen 0). 
返回 
代码: ns_error_invalid_arg (0x80070057) 
组件: displaywrap 
界面: idisplay {14fd6676-ee6b-441a-988b-c83025ab693a}

```
看网上介绍，需要卸载kvm驱动才行
```bash
modprobe -r kvm_intel kvm
```
