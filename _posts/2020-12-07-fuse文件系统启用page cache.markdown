fuse文件系统默认没有启用page cache的，需要配置mount选项
```text
   High-level mount options:
       These following options are not actually passed to the kernel but interpreted  by  libfuse.
       They can only be specified for filesystems that use the high-level libfuse API:


       kernel_cache
              This option disables flushing the cache of the file contents
              on every open(2).  This should only be enabled on filesystems,
              where the file data is never changed externally (not through
              the mounted FUSE filesystem).  Thus it is not suitable for
              network filesystems and other "intermediate" filesystems.

              NOTE: if this option is not specified (and neither direct_io)
              data is still cached after the open(2), so a read(2) system
              call will not always initiate a read operation.

       auto_cache
              This option is an alternative to kernel_cache. Instead of
              unconditionally keeping cached data, the cached data is
              invalidated on open(2) if the modification time or the size of
              the file has changed since it was last opened.
```

看起来一般是不要打开的好，auto_cache 应该比kernel_cache 更安全一些。
另外这个选项看起来只有直接使用libfuse动态库的时候才才支持，fusermount3 fusermount命令是不支持的
和fuse系统调用的 FOPEN_KEEP_CACHE 设置有关吧


参考：
-----
man mount.fuse   
https://www.man7.org/linux/man-pages/man8/fuse.8.html    
https://man7.org/linux/man-pages/man4/fuse.4.html   
https://elixir.bootlin.com/linux/latest/source/fs/fuse/file.c   
https://github.com/libfuse/libfuse/blob/master/lib/fuse.c   

