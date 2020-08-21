发现swoole的一个服务，用普通的kill总是杀不死，最后还是留一个进程在那里非要用 kill -9 才能结束掉。
看了一下，进程是卡死在 grpc shutdown的信号量上面了。
```text
#0  0x00007f533454d48c in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
#1  0x00007f53301ada52 in gpr_cv_wait () from /lib64/libgrpc.so.11
#2  0x00007f533015745b in ?? () from /lib64/libgrpc.so.11
#3  0x00007f533018e060 in grpc_shutdown_internal_locked() () from /lib64/libgrpc.so.11
#4  0x00007f533018e34c in grpc_shutdown_blocking () from /lib64/libgrpc.so.11
#5  0x00007f53304f2665 in zm_shutdown_grpc () from  grpc.so
#6  0x00000000007011db in module_destructor ()
#7  0x00000000006fb86c in module_destructor_zval ()
#8  0x000000000070bbd4 in zend_hash_graceful_reverse_destroy ()
#9  0x00000000006fc6a3 in zend_shutdown ()
#10 0x000000000069f21a in php_module_shutdown ()
#11 0x0000000000469ac4 in main ()
```

https://github.com/grpc/grpc/issues/18833  这个理由提到 要改php.ini模式启动fork模式的支持。
```text
grpc.enable_fork_support = 1
grpc.poll_strategy = epoll1
```

但这么改之后，启动的时候就死锁了，
```text
#  pstack 7009
#0  0x00007f48bc31e48c in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
#1  0x00007f48b701ba52 in gpr_cv_wait () from /lib64/libgrpc.so.11
#2  0x00007f48b6fc545b in ?? () from /lib64/libgrpc.so.11
#3  0x00007f48b6ffc060 in grpc_shutdown_internal_locked() () from /lib64/libgrpc.so.11
#4  0x00007f48b6ffc34c in grpc_shutdown_blocking () from /lib64/libgrpc.so.11
#5  0x00007f48b7360841 in postfork_child () from grpc.so
#6  0x00007f48bc639b10 in __run_fork_handlers () from /lib64/libc.so.6
#7  0x00007f48bc5f838c in fork () from /lib64/libc.so.6
#8  0x00007f48b812530d in swoole_fork () from swoole.so
#9  0x00007f48b815897f in swManager_start(swServer*) () from swoole.so
#10 0x00007f48b8160e83 in swFactoryProcess_start(swFactory*) () from swoole.so
#11 0x00007f48b815cc0b in swServer_start(swServer*) () from swoole.so
#12 0x00007f48b81cedd0 in zim_swoole_server_start(_zend_execute_data*, _zval_struct*) () from swoole.so
#13 0x000000000077d7c8 in execute_ex ()
#14 0x000000000077e323 in zend_execute ()
#15 0x00000000006fcb24 in zend_execute_scripts ()
#16 0x000000000069f520 in php_execute_script ()
#17 0x00000000007803b3 in do_cli ()
#18 0x0000000000469a68 in main ()
```
还是卡在 postfork_child 后面的 grpc_shutdown_blocking函数上面，也就是这个fork模式grpc的工作有些问题。


搜索了一下，这个grpc的fork模式确实很多问题：
https://github.com/swoole/swoole-src/issues/2604
gRPC 跨进程使用引发的问题 https://zhuanlan.zhihu.com/p/136619485
https://github.com/grpc/grpc/blob/master/doc/fork_support.md
https://github.com/grpc/grpc/issues/15334
https://github.com/grpc/grpc//src/php/README.md
https://github.com/grpc/grpc/blob/master/src/php/README.md
https://github.com/grpc/grpc/issues/13412
https://github.com/grpc/grpc/issues/20250
https://github.com/grpc/grpc/pull/22774

因为swoole本身就是fork子进程的，所以刚好跟grpc有冲突吗。这个看代码，像是grpc fork后，有些条件变量状态不对了，shutdown流程死锁了。看堆栈看出来是 
grpc_shutdown_internal_locked 调用的哪个子函数调用这个gpr_cv_wait。  

简单的把grpc_shutdown_blocking函数给注释掉了，是不会卡死了，但不知道有没有什么负面作用。看python那边postfork的逻辑好像也没有shutdown的代码，只是删除channel避免
子进程里面使用父进程的channel吧。不知道这么改grpc在子进程里面还能正常工作不，还有待测试。这个正确的做法是找到这个信号量然后看grpc的代码逻辑来修复，但工作量太大了。
```c
void postfork_child() {
  TSRMLS_FETCH();

  // loop through persistent list and destroy all underlying grpc_channel objs
  destroy_grpc_channels();

  release_persistent_locks();
  
  // clean all channels in the persistent list
  php_grpc_clean_persistent_list(TSRMLS_C);

  // clear completion queue
  grpc_php_shutdown_completion_queue(TSRMLS_C);

  // clean-up grpc_core
  /*
  grpc_shutdown_blocking();
  if (grpc_is_initialized() > 0) {
    zend_throw_exception(spl_ce_UnexpectedValueException,
                         "Oops, failed to shutdown gRPC Core after fork()",
                         1 TSRMLS_CC);
  }
   */
  // restart grpc_core
  grpc_init();
  grpc_php_init_completion_queue(TSRMLS_C);
}
```


20202-08-21补充：  有人开了一个帖子了，应该是同一个问题
https://github.com/grpc/grpc/issues/23833

