4.1x版本的内核，很多人都观察到系统启动时会有个卡顿现象。    
比如这篇文章：   
“快杰云主机 SSH 登录缓慢的排查和解决”  http://blog.ucloud.cn/archives/4310

这个老版本内核是没有这个现象的，大概是某版本里面引入的吧，就是之前有人报告内核的
随机数生成器（cryptographically-secure pseudo-random number generator (CSPRNG) ）不够“随机”（entropy熵不够），
所以改进的内核在开机初始化阶段，会有一个等待entropy pool初始化完成的过程，  
不带GRND_NONBLOCK标记的getrandom()这个系统调用会在内核里面阻塞等到内核的随机数生成器的初始化完成才会完成，这个要差不多一分钟吧，
所以很多依赖内核随机数生成器的应用开机阶段都会阻塞差不多1分钟左右。
systemd的这篇文章解释的很清楚：
```text
Linux Kernel Entropy Pool
Today’s computer systems require random number generators for numerous cryptographic and other purposes. 
On Linux systems, the kernel’s entropy pool is typically used as high-quality source of random numbers. 
The kernel’s entropy pool combines various entropy inputs together, mixes them and provides an API to 
userspace as well as to internal kernel subsystems to retrieve it. This entropy pool needs to be initialized
with a minimal level of entropy before it can provide high quality, cryptographic random numbers to applications.
Until the entropy pool is fully initialized application requests for high-quality random numbers cannot be fulfilled.

The Linux kernel provides three relevant userspace APIs to request random data from the kernel’s entropy pool:

The getrandom() system call with its flags parameter set to 0. If invoked the calling program will synchronously 
block until the random pool is fully initialized and the requested bytes can be provided.

The getrandom() system call with its flags parameter set to GRND_NONBLOCK. If invoked the request for random bytes
will fail if the pool is not initialized yet.

Reading from the /dev/urandom pseudo-device will always return random bytes immediately, even if the pool is not
initialized. The provided random bytes will be of low quality in this case however. Moreover the kernel will log
about all programs using this interface in this state, and which thus potentially rely on an uninitialized entropy pool.

(Strictly speaking there are more APIs, for example /dev/random, but these should not be used by almost any 
application and hence aren’t mentioned here.)

Note that the time it takes to initialize the random pool may differ between systems. If local hardware random
number generators are available, initialization is likely quick, but particularly in embedded and virtualized
environments available entropy is small and thus random pool initialization might take a long time (up to tens of minutes!).

Modern hardware tends to come with a number of hardware random number generators (hwrng), that may be used
to relatively quickly fill up the entropy pool. Specifically:

All recent Intel and AMD CPUs provide the CPU opcode RDRAND to acquire random bytes. Linux includes random
bytes generated this way in its entropy pool, but didn’t use to credit entropy for it (i.e. data from this
source wasn’t considered good enough to consider the entropy pool properly filled even though it was used). 
This has changed recently however, and most big distributions have turned on the CONFIG_RANDOM_TRUST_CPU=y
kernel compile time option. This means systems with CPUs supporting this opcode will be able to very quickly
reach the “pool filled” state.

The TPM security chip that is available on all modern desktop systems has a hwrng. It is also fed into the
entropy pool, but generally not credited entropy. You may use rng_core.default_quality=1000 on the kernel
command line to change that, but note that this is a global setting affect all hwrngs. (Yeah, that’s weird.)

Many Intel and AMD chipsets have hwrng chips. Their Linux drivers usually don’t credit entropy. (But there’s
rng_core.default_quality=1000, see above.)

Various embedded boards have hwrng chips. Some drivers automatically credit entropy, others do not. Some WiFi
chips appear to have hwrng sources too, and they usually do not credit entropy for them.

virtio-rng is used in virtualized environments and retrieves random data from the VM host. It credits full entropy.

The EFI firmware typically provides a RNG API. When transitioning from UEFI to kernel mode Linux will query
some random data through it, and feed it into the pool, but not credit entropy to it. What kind of random source
is behind the EFI RNG API is often not entirely clear, but it hopefully is some kind of hardware source.

If neither of these are available (in fact, even if they are), Linux generates entropy from various non-hwrng
sources in various subsystems, all of which ultimately are rooted in IRQ noise, a very “slow” source of entropy, 
in particular in virtualized environments.
```

这个最简单的应该就是“信任”硬件的随机源，编译内核时，打开“CONFIG_RANDOM_TRUST_CPU=y”这个参数就可以了。
但很多平台比如CentOS8默认都是没有打开这个的。在5.3里面又内核的开发者来尝试解决这个问题吧，参见下面的LWN的两篇文章
“Really fixing getrandom”和“Fixing getrandom()”，通过一个引入“jitter-entropy”机制来引入让随机数的初始化更快吧。
之前好像在哪里看过国内的人写的介绍的文章，忘记在哪里看到的了。具体看参见最新的内核源码drivers/char/random.c

看了源码其实不需要设置CONFIG_RANDOM_TRUST_CPU=y重新编译内核，其实linux内核有一个启动参数random.trust_cpu配置的。
```text
        random.trust_cpu={on,off}
                        [KNL] Enable or disable trusting the use of the
                        CPU's random number generator (if available) to
                        fully seed the kernel's CRNG. Default is controlled
                        by CONFIG_RANDOM_TRUST_CPU.
```
试了一下打开这个选择，果然启动时没有这个阻塞了，这个random.trust_cpu加上那个“jitter-entropy”的补丁默认一般也是够安全的吧，
不知道那些发行版什么时候会默认打开。

未打开时的内核日志，应用在系统开机未初始化完就调用这个接口会有警告：
```text
[    0.001000] random: get_random_bytes called from start_kernel+0x36b/0x55b with crng_init=0
[    1.053467] random: systemd: uninitialized urandom read (16 bytes read)
[    1.053607] random: systemd: uninitialized urandom read (16 bytes read)
[    1.053769] random: systemd: uninitialized urandom read (16 bytes read)
[    2.843114] random: crng init done
```
打开后会有这个：
```text
[    0.001000] random: crng done (trusting CPU's manufacturer)
```


这个东西其实影响蛮广的，比如golang的里面库也会针对这个情况打印警告消息。
```text
func warnBlocked() {
	println("crypto/rand: blocked for 60 seconds waiting to read random data from the kernel")
}

func (r *devReader) Read(b []byte) (n int, err error) {
	if atomic.CompareAndSwapInt32(&r.used, 0, 1) {
		// First use of randomness. Start timer to warn about
		// being blocked on entropy not being available.
		t := time.AfterFunc(60*time.Second, warnBlocked)
		defer t.Stop()
	}
	if altGetRandom != nil && r.name == urandomDevice && altGetRandom(b) {
		return len(b), nil
	}
	r.mu.Lock()
	defer r.mu.Unlock()
	if r.f == nil {
		f, err := os.Open(r.name)
		if f == nil {
			return 0, err
		}
		if runtime.GOOS == "plan9" {
			r.f = f
		} else {
			r.f = bufio.NewReader(hideAgainReader{f})
		}
	}
	return r.f.Read(b)
}
```


参考：
“Fixing getrandom()”  https://lwn.net/Articles/800509/
“Really fixing getrandom()” https://lwn.net/Articles/802360/
“Random Seeds” https://systemd.io/RANDOM_SEEDS/
“Understanding the Red Hat Enterprise Linux random number generator interface” https://www.redhat.com/en/blog/understanding-red-hat-enterprise-linux-random-number-generator-interface


