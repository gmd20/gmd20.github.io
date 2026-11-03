1. 生成 MOK 密钥对
==================
# 生成私钥 + 自签证书（证书先临时输出为 PEM）
```bash
openssl req -nodes -new -x509 -newkey rsa:2048 \
  -keyout MOK_private.pem -out MOK_public.pem \
  -days 36500 -subj "/CN=test/"
```

# 拼出合并文件（私钥在前、证书在后）
```bash
cat  MOK_private.pem  MOK_public.pem   > signing_key.pem
```

# 导出 DER 格式公钥,  mokutil 要求 DER 格式
```bash
openssl x509 -in  MOK_public.pem  -outform DER -out  MOK_public.der
```

# 收紧权限
```bash
chmod 600 MOK_private.pem  signing_key.pem
chmod 644 MOK_public.pem 
```

# 内核配置里面相关参数， debian 的 CONFIG_MODULE_SIG_FORCE 是没有设置的，
# 不设置 CONFIG_MODULE_SIG_FORCE，内核就会处于“宽松模式”（permissive），从而跳过对未签名或签名无法验证模块的强制性拦截
# 即使内核编译时未设置 CONFIG_MODULE_SIG_FORCE，如果系统以 Secure Boot 模式启动并因此触发了内核的 Lockdown（锁定）模式，
# 那么“宽松模式”就会被强制覆盖。
```text
CONFIG_MODULE_SIG=y          # 启用模块签名支持 
CONFIG_MODULE_SIG_FORCE=y    # 强制签名（Secure Boot 必需，lockdown 模式下自动生效）
CONFIG_MODULE_SIG_ALL=y      # make modules_install 时自动签名
CONFIG_MODULE_SIG_SHA256=y   # 选择哈希算法
CONFIG_MODULE_SIG_KEY="certs/signing_key.pemv"   # 签名密钥路径
CONFIG_SYSTEM_TRUSTED_KEYS="certs/MOK_public.pem"   # 内核信任的额外证书 ，只含证书不含私钥 
CONFIG_SECURITY_LOCKDOWN_LSM=y
CONFIG_LOCK_DOWN_KERNEL_FORCE_INTEGRITY=y    # Secure Boot 模式下通常自动启用
```

# CONFIG_MODULE_SIG_KEY 的默认值是 "certs/signing_key.pem"。如果你保持这个默认值不变，内核在编译时会自动生成一对临时的私钥和证书用于签名, 
# 密钥格式：CONFIG_MODULE_SIG_KEY 指向的文件必须同时包含私钥和对应的证书。  secureboot 启动时需要把这证书安装到 EFI 的 MOK 里面去。
# CONFIG_SYSTEM_TRUSTED_KEYS="certs/MOK_public.pem"   内核信任的证书列表（PEM，只含证书不含私钥），但用自己的 vmlinux  同样要给 vmlinuz签名。
```bash
$ scripts/config --set-str MODULE_SIG_KEY /root/secureboot/signing_key.pem  # 自己的证书签名内核模块
```



2. 用 sign-file 签名内核模块
============================
KBUILD_DIR=/kernel-build/linux-6.12.113/   # 按实际版本调整
```bash
$KBUILD_DIR/scripts/sign-file sha256 signing_key.pem signing_key.pem  ./test.ko
$KBUILD_DIR/scripts/sign-file sha256 MOK_private.pem MOK_public.pem  ../test.ko    # 单独的公钥私钥文件也可以
```

3. 注册 MOK 到 Secure Boot
=========================

检查 Secure Boot 状态
```bash
sudo mokutil --sb-state          # 应输出：SecureBoot enabled
```

注册密钥
```bash
sudo mokutil --import MOK_public.pem
# 会提示输入 8-16 位一次性密码，记住它
```bash

```bash
sudo mokutil --list-new          # 确认密钥已排队等待注册
sudo reboot
```
重启时固件会启动蓝紫色的 MokManager 界面，按提示操作：

看到 “Press any key to perform MOK management” 时按键；
选择 Enroll MOK → View key 0（核对指纹）→ Continue；
选择 Yes，输入刚才设置的一次性密码；
选择 Reboot。


回到系统后验证：

sudo mokutil --test-key  MOK_public.der
```
# 输出： MOK_public.der  is already enrolled

```bash
sudo dmesg | grep -i cert        # 应能看到密钥已加载进系统信任环
```

测试加载模块
```bash
sudo modinfo my_module           # 应看到 sig_id: PKCS#7、signer、sig_hashalgo 等字段
sudo insmod my_module            # 或 modprobe my_module
modinfo 中能看到 signer: My Custom MOK 即表示签名成功；如果出现 Key was rejected by service 或 Required key not available，说明密钥未被信任或签名参数不匹配
```

4.  如果是自己编译的内核, 还要签名内核
=====================================
```bash
sudo apt install sbsigntool
sudo sbsign --key /var/lib/shim-signed/mok/MOK_private.pem \
            --cert /var/lib/shim-signed/mok/MOK_public.pem \
            /boot/vmlinuz-$(uname -r) \
            --output /boot/vmlinuz-$(uname -r).tmp
sudo mv /boot/vmlinuz-$(uname -r).tmp /boot/vmlinuz-$(uname -r)
```


5. Secureboot 证书链
===================
shim -> grub -> linux 发行版
