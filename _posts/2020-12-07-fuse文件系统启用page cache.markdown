fuse文件系统默认没有启用page cache的，需要配置mount选项
```text
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

看起来一般是不要打开的好，auto_cache 应该比kernel_cache 更安全一些
