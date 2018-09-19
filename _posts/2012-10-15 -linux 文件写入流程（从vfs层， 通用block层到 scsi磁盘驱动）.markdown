```text
之前对scsi层和vfs层有点大概的了解。想学习一下通用block层的 page cache 和 那些电梯算法之类的。但还是没什么时间认真去看啊，那个东西也算比较复杂的。大概看下面这个书和简单浏览了一下源码。这书确实够经典啊，以前就全部大概翻了一下，但我读书一般也是很粗略的过一边，没什么印象。现在再去看，很多东西其实讲的还是很清楚的。后来在chinaunix论坛的 内核源码 模块也看网友发的帖子，也列了详细的文件读取和写入调用过程了。我还是自己参考找了一遍，关键是想了解不同的数据结构是怎么关联起来的。

读书笔记

这个书果然是经典啊，这几章内容都是相关的

Understanding the Linux Kernel, 3rd Edition

Chapter 14. Block Device Drivers

Chapter 15. The Page Cache

Chapter 16. Accessing Files



http://git.kernel.org/?p=linux/kernel/git/torvalds/linux.git;a=blob;f=Documentation/block/biodoc.txt

https://www.kernel.org/doc/htmldocs/kernel-api/blkdev.html 





后来在网上看到高手写的这两个帖子，源码调用列出来，比较详细啊，自己看代码时也参考了一下流程。

http://bbs.chinaunix.net/thread-3774478-1-1.html

http://bbs.chinaunix.net/thread-3772486-1-1.html







----------------------相关的数据街哦股---------------------------

file ， inode                                // 有文件关联到  vfs层。 inode那里好像有个  block  device的 指针的。

block_device    vfs  operation     // 由磁盘驱动创建 ， 

address_space       // 管理 page cache   的radix tree 查找，

        ->address_space_operations   //每个文件系统去定义

file  -> iovec->kiocb    // 把用户空闲的读写，描述为 offset 和  pos的 通用管理结构。

address_space_operations   // 文件系统注册的  真正的 读写page的操作，文件系统也使用 block层的很懂辅助函数来完成工作。

page , bufffer_head       //  表示 page cache 缓存在 每个page上面的 关系。  用于page cache层/

bio         // 由 buffer head 生成，可以多个合并在 一起成为 request，    page cache 层的 buffer head  和 磁盘request 之间的 转换结构吧。

gendisk       磁盘驱动发现 存储设备的时候创建这个对象，表示一个磁盘。 同时每个都有对应的request queue 。

request queue

request       //  通用的 block层 ，磁盘请求。  由 bio 生成，

scsi command            // scsi 层使用





--------------------------

vfs的 inode的初始化时，指定 address_space_operations



http://lxr.free-electrons.com/source/fs/ext4/inode.c

static const struct address_space_operations ext4_aops = {

        .readpage               = ext4_readpage,

        .readpages              = ext4_readpages,

        .writepage              = ext4_writepage,  写脏页到磁盘

        .writepages             = ext4_writepages,  写脏页到磁盘

        .write_begin            = ext4_write_begin,  ///generic_perform_write 调用到，准备好需要操作的 page cache对应的page

        .write_end              = ext4_write_end,

        .bmap                   = ext4_bmap,

        .invalidatepage         = ext4_invalidatepage,

        .releasepage            = ext4_releasepage,

        .direct_IO              = ext4_direct_IO,

        .migratepage            = buffer_migrate_page,

        .is_partially_uptodate  = block_is_partially_uptodate,

        .error_remove_page      = generic_error_remove_page,

};


----------------------------

block device  -> inode

                    -> struct  hd_struct  *  hd_part  分区

                    -> struct gendisk* hd_disk   磁盘





系统所有的block device 都在全局链表里面 all_bdevs 





驱动

.4.2.1. Defining a custom driver descriptor

.4.2.2. Initializing the custom descriptor    register_blkdev 函数

.4.2.3. Initializing the gendisk descriptor

.4.2.4. Initializing the table of block device methods

.4.2.5. Allocating and initializing a request queue      blk_init_queue函数

.4.2.6. Setting up the interrupt handler    request_irq函数

.4.2.7. Registering the disk    add_disk  函数， 注册 sys 文件系统 kobject，扫描 分区，初始化gendisk的分



区数组





default block device file operations

read generic_file_read( )

write blkdev_file_write( )

aio_read generic_file_aio_read( )

aio_write blkdev_file_aio_write( )

----------------------------

发现硬盘  -> alloc_disk( )->



gendisk-> request queue 

           -> struct hd_struct



bio_alloc

bio-> struct block_device * bi_bdev





Request Queue  ->struct request * last_merge

                         ->elevator_t * elevator

                         -> request_fn_proc * request_fn

                         ->spinlock_t * queue_lock

                         ->unsigned short max_hw_sectors

                         ->unsigned short max_phys_segments



-----------------------

Page Cache



If the owner of a page in the page cache is a file, the address_space object is embedded in the

i_data field of a VFS inode object. The i_mapping field of the inode always points to the

address_space object of the owner of the pages containing the inode's data. The host field of the

address_space object points to the inode object in which the descriptor is embedded.





Thus, if a page belongs to a file that is stored in an Ext3 filesystem , the owner of the page is the

inode of the file and the corresponding address_space object is stored in the i_data field of the

VFS inode object. The i_mapping field of the inode points to the i_data field of the same inode,

and the host field of the address_space object points to the same inode.



The methods of the address_space object

writepage Write operation (from the page to the owner's disk image)

sync_page Start the I/O data transfer of already scheduled operations on owner's pages

set_page_dirty Set an owner's page as dirty

The most important methods are readpage, writepage, prepare_write, and commit_write.



find_get_page(

add_to_page_cache

remove_from_page_cache

read_cache_page





buffer_head 结构  管理  buffer在 page的什么地方



buffer_head -> b_state  标志，是不是 dirty  ，异步等

                   -> page   这个block在哪个 page上面

                   -> b_size      大小

                   ->  b_data  字啊page里面的偏移





alloc_buffer_head( ) and free_buffer_head( )



.2.4. Allocating Block Device Buffer Pages

.2.7. Submitting Buffer Heads to the Generic Block Layer

.2.7.1. The submit_bh( ) function







.3. Writing Dirty Pages to Disk

.3.1. The pdflush Kernel Threads

Earlier



sync ()    把进程的所有脏页写到磁盘

fsyncs()   把进程某个文件的脏页写到磁盘

fdatasync()    同fsync 但不包含文件的inode block





The service routine sys_sync( ) of the sync( ) system call invokes a series of auxiliary functions:

wakeup_bdflush(0);

sync_inodes(0);

sync_supers( );

sync_filesystems(0);

sync_filesystems(1);

sync_inodes(1);





================================

Chapter 16. Accessing Files



Reading a file is page-based: the kernel always transfers whole pages of data at once. If a

process issues a read( ) system call to get a few bytes, and that data is not already in RAM, the

kernel allocates a new page frame, fills the page with the suitable portion of the file, adds the

page to the page cache, and finally copies the requested bytes into the process address space.

For most filesystems, reading a page of data from a file is just a matter of finding what blocks on

disk contain the requested data. Once this is done, the kernel fills the pages by submitting the

proper I/O operations to the generic block layer. In practice, the read method of all disk-based

filesystems is implemented by a common function named generic_file_read( ).

Write operations on disk-based files are slightly more complicated to handle, because the file size

could increase, and therefore the kernel might allocate some physical blocks on the disk. Of

course, how this is precisely done depends on the filesystem type. However, many disk-based

filesystems implement their write methods by means of a common function named

generic_file_write( ). Examples of such filesystems are Ext2, System V /Coherent /Xenix , and

MINIX . On the other hand, several other filesystems, such as journaling and network filesystems

, implement the write method by means of custom functions.













generic_file_read -> do_generic_file_read 函数  

      flip  操作的文件文件对象

       查找address_space 这这里filp->f_mapping.     找出page cache 对应的page buffer

      page_cache_readahead     调用read_page去实际读文件





// 需要实际从磁盘读出相应页

int ext3_readpage(struct file *file, struct page *page)

{

return mpage_readpage(page, ext3_get_block);

}

// 直接操作 块设备的读 

}int blkdev_readpage(struct file * file, struct * page page)

{

return block_read_full_page(page, blkdev_get_block);

}











generic_file_write

    file object  , buffer   转换到  iovec

    找到 inode 



    init_sync_kiocb   kiocb.



_ _generic_file_aio_write_nolock

   ext4_write_begin

           grab_cache_page_write_begin

           __block_write_begin

  ext4_generic_write_end

          block_write_end

         __block_commit_write



drivers. The block layer make_request function builds up a request structure,
places it on the queue and invokes the drivers request_fn. The driver makes
use of block layer helper routine elv_next_request to pull the next request
off the queue. Control or diagnostic functions might bypass block and directly
invoke underlying driver entry points passing in a specially constructed
request structure.



代码阅读



http://lxr.linux.no/linux+v3.6.2/fs/read_write.c#L479
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
size_t, count)
{
 struct file *file;
 ssize_t ret = -EBADF;
 int fput_needed;

 file = fget_light(fd, &fput_needed);
 if (file) {
  loff_t pos = file_pos_read(file);
  ret = vfs_write(file, buf, count, &pos);
  file_pos_write(file, pos);
  fput_light(file, fput_needed);
 }

 return ret;
}


ssize_t vfs_write(struct file *file, const char __user *buf, size_t count, loff_t *pos)
{
       ssize_t ret;

       if (!(file->f_mode & FMODE_WRITE))
               return -EBADF;
       if (!file->f_op || (!file->f_op->write && !file->f_op->aio_write))
               return -EINVAL;
       if (unlikely(!access_ok(VERIFY_READ, buf, count)))
               return -EFAULT;

       ret = rw_verify_area(WRITE, file, pos, count);
       if (ret >= 0) {
               count = ret;
               if (file->f_op->write)
                       ret = file->f_op->write(file, buf, count, pos);
               else
                       ret = do_sync_write(file, buf, count, pos);
               if (ret > 0) {
                       fsnotify_modify(file);
                       add_wchar(current, ret);
               }
               inc_syscw(current);
       }

       return ret;
}

EXPORT_SYMBOL(vfs_write);

ssize_t do_sync_write(struct file *filp, const char __user *buf, size_t len, loff_t *ppos)
{
       struct iovec iov = { .iov_base = (void __user *)buf, .iov_len = len };
       struct kiocb kiocb;
       ssize_t ret;

       init_sync_kiocb(&kiocb, filp);
       kiocb.ki_pos = *ppos;
       kiocb.ki_left = len;
       kiocb.ki_nbytes = len;

       for (;;) {
               ret = filp->f_op->aio_write(&kiocb, &iov, 1, kiocb.ki_pos);  // 通过异步来做同步的写的
               if (ret != -EIOCBRETRY)
                       break;
               wait_on_retry_sync_kiocb(&kiocb); //这个就是同步操作的等待？？？
       }

       if (-EIOCBQUEUED == ret)
               ret = wait_on_sync_kiocb(&kiocb);  //这个就是同步操作的等待？？？
       *ppos = kiocb.ki_pos;
       return ret;
}

EXPORT_SYMBOL(do_sync_write);

http://lxr.linux.no/linux+v3.6.2/fs/ext4/file.c#L307
const struct file_operations ext4_file_operations = {
       .llseek         = ext4_llseek,
       .read           = do_sync_read,
       .write          = do_sync_write,
       .aio_read       = generic_file_aio_read,
       .aio_write      = ext4_file_write,     ////异步操作对应这个
       .unlocked_ioctl = ext4_ioctl,
#ifdef CONFIG_COMPAT
       .compat_ioctl   = ext4_compat_ioctl,
#endif
       .mmap           = ext4_file_mmap,
       .open           = ext4_file_open,
       .release        = ext4_release_file,
       .fsync          = ext4_sync_file,
       .splice_read    = generic_file_splice_read,
       .splice_write   = generic_file_splice_write,
       .fallocate      = ext4_fallocate,


http://lxr.linux.no/linux+v3.6.2/fs/ext4/file.c#L174
static ssize_t
ext4_file_write(struct kiocb *iocb, const struct iovec *iov,
               unsigned long nr_segs, loff_t pos)
{
       struct inode *inode = iocb->ki_filp->f_path.dentry->d_inode;
       ssize_t ret;

       /*
        * If we have encountered a bitmap-format file, the size limit
        * is smaller than s_maxbytes, which is for extent-mapped files.
        */

       if (!(ext4_test_inode_flag(inode, EXT4_INODE_EXTENTS))) {
               struct ext4_sb_info *sbi = EXT4_SB(inode->i_sb);
               size_t length = iov_length(iov, nr_segs);

               if ((pos > sbi->s_bitmap_maxbytes ||
                   (pos == sbi->s_bitmap_maxbytes && length > 0)))
                       return -EFBIG;

               if (pos + length > sbi->s_bitmap_maxbytes) {
                       nr_segs = iov_shorten((struct iovec *)iov, nr_segs,
                                             sbi->s_bitmap_maxbytes - pos);
               }
       }

       if (unlikely(iocb->ki_filp->f_flags & O_DIRECT))
               ret = ext4_file_dio_write(iocb, iov, nr_segs, pos);
       else
               ret = generic_file_aio_write(iocb, iov, nr_segs, pos);      ////非 direct io

       return ret;
}


ssize_t __generic_file_aio_write(struct kiocb *iocb, const struct iovec *iov,
                             unsigned long nr_segs, loff_t *ppos)



ssize_t
generic_file_buffered_write(struct kiocb *iocb, const struct iovec *iov,
               unsigned long nr_segs, loff_t pos, loff_t *ppos,
               size_t count, ssize_t written)
{
       struct file *file = iocb->ki_filp;
       ssize_t status;
       struct iov_iter i;
       iov_iter_init(&i, iov, nr_segs, count, written);
       status = generic_perform_write(file, &i, pos);
       if (likely(status >= 0)) {
               written += status;
               *ppos = pos + status;
       }
       
       return written ? written : status;
}
EXPORT_SYMBOL(generic_file_buffered_write);

static ssize_t generic_perform_write(struct file *file,
                               struct iov_iter *i, loff_t pos)
{
       struct address_space *mapping = file->f_mapping;      ///file 得到address space对象
       const struct address_space_operations *a_ops = mapping->a_ops;  // 文件系统注册的操作
       long status = 0;
       ssize_t written = 0;
       unsigned int flags = 0;

       /*
        * Copies from kernel address space cannot fail (NFSD is a big user).
        */
       if (segment_eq(get_fs(), KERNEL_DS))
               flags |= AOP_FLAG_UNINTERRUPTIBLE;

       do {
               struct page *page;
               unsigned long offset;   /* Offset into pagecache page */
               unsigned long bytes;    /* Bytes to write to page */
               size_t copied;          /* Bytes copied from user */
               void *fsdata;

               offset = (pos & (PAGE_CACHE_SIZE - 1));
               bytes = min_t(unsigned long, PAGE_CACHE_SIZE - offset,
                                               iov_iter_count(i));

again:
               /*
                * Bring in the user page that we will copy from _first_.
                * Otherwise there's a nasty deadlock on copying from the
                * same page as we're writing to, without it being marked
                * up-to-date.
                *
                * Not only is this an optimisation, but it is also required
                * to check that the address is actually valid, when atomic
                * usercopies are used, below.
                */
               if (unlikely(iov_iter_fault_in_readable(i, bytes))) {
                       status = -EFAULT;
                       break;
               }

               status = a_ops->write_begin(file, mapping, pos, bytes, flags,
                                               &page, &fsdata);       ///写开始，应该是写到 page cache层次。
               if (unlikely(status))
                       break;

               if (mapping_writably_mapped(mapping))
                       flush_dcache_page(page);

               pagefault_disable();
               copied = iov_iter_copy_from_user_atomic(page, i, offset, bytes);
               pagefault_enable();
               flush_dcache_page(page);

               mark_page_accessed(page);
               status = a_ops->write_end(file, mapping, pos, bytes, copied,
                                               page, fsdata);     ///写结束
               if (unlikely(status < 0))
                       break;
               copied = status;

               cond_resched();

               iov_iter_advance(i, copied);
               if (unlikely(copied == 0)) {
                       /*
                        * If we were unable to copy any data at all, we must
                        * fall back to a single segment length write.
                        *
                        * If we didn't fallback here, we could livelock
                        * because not all segments in the iov can be copied at
                        * once without a pagefault.
                        */
                       bytes = min_t(unsigned long, PAGE_CACHE_SIZE - offset,
                                               iov_iter_single_seg_count(i));
                       goto again;
               }
               pos += copied;
               written += copied;

               balance_dirty_pages_ratelimited(mapping);
               if (fatal_signal_pending(current)) {
                       status = -EINTR;
                       break;
               }
       } while (iov_iter_count(i));

       return written ? written : status;
}


static int ext4_write_begin(struct file *file, struct address_space *mapping,
                           loff_t pos, unsigned len, unsigned flags,
                           struct page **pagep, void **fsdata)
{
       struct inode *inode = mapping->host;
       int ret, needed_blocks;
       handle_t *handle;
       int retries = 0;
       struct page *page;
       pgoff_t index;
       unsigned from, to;

       trace_ext4_write_begin(inode, pos, len, flags);
       /*
        * Reserve one block more for addition to orphan list in case
        * we allocate blocks but write fails for some reason
        */
       needed_blocks = ext4_writepage_trans_blocks(inode) + 1;
       index = pos >> PAGE_CACHE_SHIFT;      // 这个是radix tree的索引？ 
       from = pos & (PAGE_CACHE_SIZE - 1);
       to = from + len;

retry:
       handle = ext4_journal_start(inode, needed_blocks);
       if (IS_ERR(handle)) {
               ret = PTR_ERR(handle);
               goto out;
       }

       /* We cannot recurse into the filesystem as the transaction is already
        * started */
       flags |= AOP_FLAG_NOFS;

       page = grab_cache_page_write_begin(mapping, index, flags);     //在page cache准备好需要写进入的page ？
       if (!page) {
               ext4_journal_stop(handle);
               ret = -ENOMEM;
               goto out;
       }
       *pagep = page;

       if (ext4_should_dioread_nolock(inode))
               ret = __block_write_begin(page, pos, len, ext4_get_block_write);
       else
               ret = __block_write_begin(page, pos, len, ext4_get_block);

       if (!ret && ext4_should_journal_data(inode)) {
               ret = walk_page_buffers(handle, page_buffers(page),
                               from, to, NULL, do_journal_get_write_access);
       }

       if (ret) {
               unlock_page(page);
               page_cache_release(page);
               /*
                * __block_write_begin may have instantiated a few blocks
                * outside i_size.  Trim these off again. Don't need
                * i_size_read because we hold i_mutex.
                *
                * Add inode to orphan list in case we crash before
                * truncate finishes
                */
               if (pos + len > inode->i_size && ext4_can_truncate(inode))
                       ext4_orphan_add(handle, inode);

               ext4_journal_stop(handle);
               if (pos + len > inode->i_size) {
                       ext4_truncate_failed_write(inode);
                       /*
                        * If truncate failed early the inode might
                        * still be on the orphan list; we need to
                        * make sure the inode is removed from the
                        * orphan list in that case.
                        */
                       if (inode->i_nlink)
                               ext4_orphan_del(NULL, inode);
               }
       }

       if (ret == -ENOSPC && ext4_should_retry_alloc(inode->i_sb, &retries))
               goto retry;
out:
       return ret;
}


/*
* Find or create a page at the given pagecache position. Return the locked
* page. This function is specifically for buffered writes.
*/
struct page *grab_cache_page_write_begin(struct address_space *mapping,
                                       pgoff_t index, unsigned flags)
{
       int status;
       gfp_t gfp_mask;
       struct page *page;
       gfp_t gfp_notmask = 0;

       gfp_mask = mapping_gfp_mask(mapping);
       if (mapping_cap_account_dirty(mapping))
               gfp_mask |= __GFP_WRITE;
       if (flags & AOP_FLAG_NOFS)
               gfp_notmask = __GFP_FS;
repeat:
       page = find_lock_page(mapping, index);         //从 radix tree里面找到对应的page
       if (page)
               goto found;

       page = __page_cache_alloc(gfp_mask & ~gfp_notmask);
       if (!page)
               return NULL;
       status = add_to_page_cache_lru(page, mapping, index,
                                               GFP_KERNEL & ~gfp_notmask);
       if (unlikely(status)) {
               page_cache_release(page);
               if (status == -EEXIST)
                       goto repeat;
               return NULL;
       }
found:
       wait_on_page_writeback(page);    //等待上一次写操作完成？？？？
       return page;
}
EXPORT_SYMBOL(grab_cache_page_write_begin);



http://lxr.linux.no/linux+v3.6.2/fs/buffer.c#L1792
int __block_write_begin(struct page *page, loff_t pos, unsigned len,
               get_block_t *get_block)
{
       unsigned from = pos & (PAGE_CACHE_SIZE - 1);
       unsigned to = from + len;
       struct inode *inode = page->mapping->host;     ///page 和 inode的关系这样的啊
       unsigned block_start, block_end;
       sector_t block;      //扇区 ？？
       int err = 0;
       unsigned blocksize, bbits;
       struct buffer_head *bh, *head, *wait[2], **wait_bh=wait;    ///////看来这个函数是准备buffer_head 的

       BUG_ON(!PageLocked(page));
       BUG_ON(from > PAGE_CACHE_SIZE);
       BUG_ON(to > PAGE_CACHE_SIZE);
       BUG_ON(from > to);

       blocksize = 1 << inode->i_blkbits;
       if (!page_has_buffers(page))
               create_empty_buffers(page, blocksize, 0);   ///为 page 创建相应的 空的buffer_head
       head = page_buffers(page);     // page对应的  buffer_head ，

       bbits = inode->i_blkbits;
       block = (sector_t)page->index << (PAGE_CACHE_SHIFT - bbits);

       for(bh = head, block_start = 0; bh != head || !block_start;
           block++, block_start=block_end, bh = bh->b_this_page) {
               block_end = block_start + blocksize;
               if (block_end <= from || block_start >= to) {
                       if (PageUptodate(page)) {
                               if (!buffer_uptodate(bh))
                                       set_buffer_uptodate(bh);
                       }
                       continue;
               }
               if (buffer_new(bh))
                       clear_buffer_new(bh);
               if (!buffer_mapped(bh)) {
                       WARN_ON(bh->b_size != blocksize);
                       err = get_block(inode, block, bh, 1);      //准备好buffer_head 对应的 block的内存映射？？  这里也会设置buffer_head的 block_device 为对应的 inode的 block_device. 这样后面处理的buffer_head的时候就知道 是那个设备上去的了。bio  和 request就可以发到 对应的设备上去
                       if (err)
                               break;
                       if (buffer_new(bh)) {
                               unmap_underlying_metadata(bh->b_bdev,
                                                       bh->b_blocknr);
                               if (PageUptodate(page)) {
                                       clear_buffer_new(bh);
                                       set_buffer_uptodate(bh);
                                       mark_buffer_dirty(bh);
                                       continue;
                               }
                               if (block_end > to || block_start < from)
                                       zero_user_segments(page,
                                               to, block_end,
                                               block_start, from);
                               continue;
                       }
               }
               if (PageUptodate(page)) {
                       if (!buffer_uptodate(bh))
                               set_buffer_uptodate(bh);
                       continue; 
               }
               if (!buffer_uptodate(bh) && !buffer_delay(bh) &&
                   !buffer_unwritten(bh) &&
                    (block_start < from || block_end > to)) {
                       ll_rw_block(READ, 1, &bh);              /// 请求从磁盘加载数据来填充对应的page cache 的buffer_head 对应的那些内存映射结构。  
                       *wait_bh++=bh;
               }
       }
       /*
        * If we issued read requests - let them complete.
        */
       while(wait_bh > wait) {           /// 这里等待数据加载？？？？
               wait_on_buffer(*--wait_bh);
               if (!buffer_uptodate(*wait_bh))
                       err = -EIO;
       }
       if (unlikely(err))
               page_zero_new_buffers(page, from, to);
       return err;
}
EXPORT_SYMBOL(__block_write_begin);
 
__block_write_begin 只是发起了读的bio到底层了，并没有触发 写bio操作。那个应该是等待 page cache的策略去触发或者  block_write_end 里面再去出来




static int _ext4_get_block(struct inode *inode, sector_t iblock,
                           struct buffer_head *bh, int flags)
{
        handle_t *handle = ext4_journal_current_handle();
        struct ext4_map_blocks map;
        int ret = 0, started = 0;
        int dio_credits;

        map.m_lblk = iblock;
        map.m_len = bh->b_size >> inode->i_blkbits;

        if (flags && !handle) {
                /* Direct IO write... */
                if (map.m_len > DIO_MAX_BLOCKS)
                        map.m_len = DIO_MAX_BLOCKS;
                dio_credits = ext4_chunk_trans_blocks(inode, map.m_len);
                handle = ext4_journal_start(inode, dio_credits);
                if (IS_ERR(handle)) {
                        ret = PTR_ERR(handle);
                        return ret;
                }
                started = 1;
        }
/*
 * The ext4_map_blocks() function tries to look up the requested blocks,
 * and returns if the blocks are already mapped.
     */
        ret = ext4_map_blocks(handle, inode, &map, flags);
        if (ret > 0) {
                map_bh(bh, inode->i_sb, map.m_pblk);   //设置  buffer_head的 block_device 为 inode的device
                bh->b_state = (bh->b_state & ~EXT4_MAP_FLAGS) | map.m_flags;
                bh->b_size = inode->i_sb->s_blocksize * map.m_len;
                ret = 0;
        }
        if (started)
                ext4_journal_stop(handle);
        return ret;
}

int ext4_get_block(struct inode *inode, sector_t iblock,
                   struct buffer_head *bh, int create)
{
        return _ext4_get_block(inode, iblock, bh,
                               create ? EXT4_GET_BLOCKS_CREATE : 0);
}



http://lxr.linux.no/linux+v3.6.2/fs/buffer.c#L2939
/**
* ll_rw_block: low-level access to block devices (DEPRECATED)
* @rw: whether to %READ or %WRITE or maybe %READA (readahead)
* @nr: number of &struct buffer_heads in the array
* @bhs: array of pointers to &struct buffer_head
*
* ll_rw_block() takes an array of pointers to &struct buffer_heads, and
* requests an I/O operation on them, either a %READ or a %WRITE.  The third
* %READA option is described in the documentation for generic_make_request()
* which ll_rw_block() calls.
*
* This function drops any buffer that it cannot get a lock on (with the
* BH_Lock state bit), any buffer that appears to be clean when doing a write
* request, and any buffer that appears to be up-to-date when doing read
* request.  Further it marks as clean buffers that are processed for
* writing (the buffer cache won't assume that they are actually clean
* until the buffer gets unlocked).
*
* ll_rw_block sets b_end_io to simple completion handler that marks
* the buffer up-to-date (if approriate), unlocks the buffer and wakes
* any waiters. 
*
* All of the buffers must be for the same device, and must also be a
* multiple of the current approved size for the device.
*/
void ll_rw_block(int rw, int nr, struct buffer_head *bhs[])
{
       int i;

       for (i = 0; i < nr; i++) {
               struct buffer_head *bh = bhs[i];

               if (!trylock_buffer(bh))
                       continue;
               if (rw == WRITE) {
                       if (test_clear_buffer_dirty(bh)) {
                               bh->b_end_io = end_buffer_write_sync;
                               get_bh(bh);
                               submit_bh(WRITE, bh);
                               continue;
                       }
               } else {
                       if (!buffer_uptodate(bh)) {
                               bh->b_end_io = end_buffer_read_sync;
                               get_bh(bh);    
                               submit_bh(rw, bh);   ///根据 buffer_head 数组提交 bio ？？？
                               continue;
                       }
               }
               unlock_buffer(bh);
       }
}
EXPORT_SYMBOL(ll_rw_block);


http://lxr.linux.no/linux+v3.6.2/fs/buffer.c#L2867

int submit_bh(int rw, struct buffer_head * bh)
{
       struct bio *bio;
       int ret = 0;

       BUG_ON(!buffer_locked(bh));
       BUG_ON(!buffer_mapped(bh));
       BUG_ON(!bh->b_end_io);
       BUG_ON(buffer_delay(bh));
       BUG_ON(buffer_unwritten(bh));

       /*
        * Only clear out a write error when rewriting
        */
       if (test_set_buffer_req(bh) && (rw & WRITE))
               clear_buffer_write_io_error(bh);

       /*
        * from here on down, it's all bio -- do the initial mapping,
        * submit_bio -> generic_make_request may further map this bio around
        */
       bio = bio_alloc(GFP_NOIO, 1);     ///申请一个 bio ，bio原来从这里出来的啊

       bio->bi_sector = bh->b_blocknr * (bh->b_size >> 9);
       bio->bi_bdev = bh->b_bdev;     // bio 是  block device的bio，关联起来。 
       bio->bi_io_vec[0].bv_page = bh->b_page;
       bio->bi_io_vec[0].bv_len = bh->b_size;
       bio->bi_io_vec[0].bv_offset = bh_offset(bh);

       bio->bi_vcnt = 1;
       bio->bi_idx = 0;
       bio->bi_size = bh->b_size;

       bio->bi_end_io = end_bio_bh_io_sync;
       bio->bi_private = bh;      //////这里

       bio_get(bio);
       submit_bio(rw, bio);  ///提交bio 

       if (bio_flagged(bio, BIO_EOPNOTSUPP))
               ret = -EOPNOTSUPP;

       bio_put(bio);
       return ret;
}
EXPORT_SYMBOL(submit_bh);



http://lxr.linux.no/linux+v3.6.2/block/blk-core.c#L1813
/**
* submit_bio - submit a bio to the block device layer for I/O
* @rw: whether to %READ or %WRITE, or maybe to %READA (read ahead)
* @bio: The &struct bio which describes the I/O
*
* submit_bio() is very similar in purpose to generic_make_request(), and
* uses that function to do most of the work. Both are fairly rough
* interfaces; @bio must be presetup and ready for I/O.
*
*/
void submit_bio(int rw, struct bio *bio)
{
       int count = bio_sectors(bio);

       bio->bi_rw |= rw;

       /*
        * If it's a regular read/write or a barrier with data attached,
        * go through the normal accounting stuff before submission.
        */
       if (bio_has_data(bio) && !(rw & REQ_DISCARD)) {
               if (rw & WRITE) {
                       count_vm_events(PGPGOUT, count);
               } else {
                       task_io_account_read(bio->bi_size);
                       count_vm_events(PGPGIN, count);
               }

               if (unlikely(block_dump)) {
                       char b[BDEVNAME_SIZE];
                       printk(KERN_DEBUG "%s(%d): %s block %Lu on %s (%u sectors)\n",
                       current->comm, task_pid_nr(current),
                               (rw & WRITE) ? "WRITE" : "READ",
                               (unsigned long long)bio->bi_sector,
                               bdevname(bio->bi_bdev, b),
                               count);
               }
       }

       generic_make_request(bio);
}
EXPORT_SYMBOL(submit_bio);


/**
* generic_make_request - hand a buffer to its device driver for I/O
* @bio:  The bio describing the location in memory and on the device.
*
* generic_make_request() is used to make I/O requests of block
* devices. It is passed a &struct bio, which describes the I/O that needs
* to be done.
*
* generic_make_request() does not return any status.  The
* success/failure status of the request, along with notification of
* completion, is delivered asynchronously through the bio->bi_end_io
* function described (one day) else where.
*
* The caller of generic_make_request must make sure that bi_io_vec
* are set to describe the memory buffer, and that bi_dev and bi_sector are
* set to describe the device address, and the
* bi_end_io and optionally bi_private are set to describe how
* completion notification should be signaled.
*
* generic_make_request and the drivers it calls may use bi_next if this
* bio happens to be merged with someone else, and may resubmit the bio to
* a lower device by calling into generic_make_request recursively, which
* means the bio should NOT be touched after the call to ->make_request_fn.
*/
void generic_make_request(struct bio *bio)
{
       struct bio_list bio_list_on_stack;

       if (!generic_make_request_checks(bio))
               return;

       /*
        * We only want one ->make_request_fn to be active at a time, else
        * stack usage with stacked devices could be a problem.  So use
        * current->bio_list to keep a list of requests submited by a
        * make_request_fn function.  current->bio_list is also used as a
        * flag to say if generic_make_request is currently active in this
        * task or not.  If it is NULL, then no make_request is active.  If
        * it is non-NULL, then a make_request is active, and new requests
        * should be added at the tail
        */
       if (current->bio_list) {
               bio_list_add(current->bio_list, bio);
               return;
       }

       /* following loop may be a bit non-obvious, and so deserves some
        * explanation.
        * Before entering the loop, bio->bi_next is NULL (as all callers
        * ensure that) so we have a list with a single bio.
        * We pretend that we have just taken it off a longer list, so
        * we assign bio_list to a pointer to the bio_list_on_stack,
        * thus initialising the bio_list of new bios to be
        * added.  ->make_request() may indeed add some more bios
        * through a recursive call to generic_make_request.  If it
        * did, we find a non-NULL value in bio_list and re-enter the loop
        * from the top.  In this case we really did just take the bio
        * of the top of the list (no pretending) and so remove it from
        * bio_list, and call into ->make_request() again.
        */
       BUG_ON(bio->bi_next);
       bio_list_init(&bio_list_on_stack);
       current->bio_list = &bio_list_on_stack;
       do {
               struct request_queue *q = bdev_get_queue(bio->bi_bdev);   ///关联起来，从 bio 的 block_device 得到request queue， 创建的时候， request queue 和 scsi device 应该就是一一对应的了

               q->make_request_fn(q, bio);   //调用队列的 make_request_fn

               bio = bio_list_pop(current->bio_list);
       } while (bio);
       current->bio_list = NULL; /* deactivate */
}
EXPORT_SYMBOL(generic_make_request);


struct request_queue *scsi_alloc_queue(struct scsi_device *sdev)
{
        struct request_queue *q;


        q = __scsi_alloc_queue(sdev->host, scsi_request_fn);    ////  注册的 scsi_request_fn 函数
        if (!q)
                return NULL;

        blk_queue_prep_rq(q, scsi_prep_fn);
        blk_queue_softirq_done(q, scsi_softirq_done);
        blk_queue_rq_timed_out(q, scsi_times_out);
        blk_queue_lld_busy(q, scsi_lld_busy);
        return q;
}


/**
 * blk_init_queue  - prepare a request queue for use with a block device
 * @rfn:  The function to be called to process requests that have been
 *        placed on the queue.
 * @lock: Request queue spin lock
 *
 * Description:
 *    If a block device wishes to use the standard request handling procedures,
 *    which sorts requests and coalesces adjacent requests, then it must
 *    call blk_init_queue().  The function @rfn will be called when there
 *    are requests on the queue that need to be processed.  If the device
 *    supports plugging, then @rfn may not be called immediately when requests
 *    are available on the queue, but may be called at some time later instead.
 *    Plugged queues are generally unplugged when a buffer belonging to one
 *    of the requests on the queue is needed, or due to memory pressure.
 *
 *    @rfn is not required, or even expected, to remove all requests off the
 *    queue, but only as many as it can handle at a time.  If it does leave
 *    requests on the queue, it is responsible for arranging that the requests
 *    get dealt with eventually.
 *
 *    The queue spin lock must be held while manipulating the requests on the
 *    request queue; this lock will be taken also from interrupt context, so irq
 *    disabling is needed for it.
 *
 *    Function returns a pointer to the initialized request queue, or %NULL if
 *    it didn't succeed.
 *
 * Note:
 *    blk_init_queue() must be paired with a blk_cleanup_queue() call
 *    when the block device is deactivated (such as at module unload).
 **/

struct request_queue *blk_init_queue(request_fn_proc *rfn, spinlock_t *lock)
{
        return blk_init_queue_node(rfn, lock, -1);
}
EXPORT_SYMBOL(blk_init_queue);


scsi_alloc_queue->__scsi_alloc_queue -> blk_init_queue -> blk_init_allocated_queue-> blk_queue_make_request
把 queue的 make_request_fn 设置为 blk_queue_bio  

scsi每个设备应该都有一个它自己的 request queue



void blk_queue_bio(struct request_queue *q, struct bio *bio)
{
       const bool sync = !!(bio->bi_rw & REQ_SYNC);
       struct blk_plug *plug;
       int el_ret, rw_flags, where = ELEVATOR_INSERT_SORT;
       struct request *req;
       unsigned int request_count = 0;

       /*
        * low level driver can indicate that it wants pages above a
        * certain limit bounced to low memory (ie for highmem, or even
        * ISA dma in theory)
        */
       blk_queue_bounce(q, &bio);

       if (bio->bi_rw & (REQ_FLUSH | REQ_FUA)) {
               spin_lock_irq(q->queue_lock);
               where = ELEVATOR_INSERT_FLUSH;
               goto get_rq;
       }

       /*
        * Check if we can merge with the plugged list before grabbing
        * any locks.
        */
       if (attempt_plug_merge(q, bio, &request_count))
               return;

       spin_lock_irq(q->queue_lock);

       el_ret = elv_merge(q, &req, bio);           //先是尝试把bio 合并到队列里面已有request上面 
       if (el_ret == ELEVATOR_BACK_MERGE) {
               if (bio_attempt_back_merge(q, req, bio)) {
                       elv_bio_merged(q, req, bio);
                       if (!attempt_back_merge(q, req))
                               elv_merged_request(q, req, el_ret);
                       goto out_unlock;
               }
       } else if (el_ret == ELEVATOR_FRONT_MERGE) {
               if (bio_attempt_front_merge(q, req, bio)) {
                       elv_bio_merged(q, req, bio);
                       if (!attempt_front_merge(q, req))
                               elv_merged_request(q, req, el_ret);
                       goto out_unlock;
               }
       }

get_rq:
       /*
        * This sync check and mask will be re-done in init_request_from_bio(),
        * but we need to set it earlier to expose the sync flag to the
        * rq allocator and io schedulers.
        */
       rw_flags = bio_data_dir(bio);
       if (sync)
               rw_flags |= REQ_SYNC;

       /*
        * Grab a free request. This is might sleep but can not fail.
        * Returns with the queue unlocked.
        */
       req = get_request(q, rw_flags, bio, GFP_NOIO);        // 如果前面合并不成功这里申请一个空闲request，然后把bio放到这个request里面吧
       if (unlikely(!req)) {
               bio_endio(bio, -ENODEV);        /* @q is dead */
               goto out_unlock;
       }

       /*
        * After dropping the lock and possibly sleeping here, our request
        * may now be mergeable after it had proven unmergeable (above).
        * We don't worry about that case for efficiency. It won't happen
        * often, and the elevators are able to handle it.
        */
       init_request_from_bio(req, bio);

       if (test_bit(QUEUE_FLAG_SAME_COMP, &q->queue_flags))
               req->cpu = raw_smp_processor_id();

       plug = current->plug;
       if (plug) {
               /*
                * If this is the first request added after a plug, fire
                * of a plug trace. If others have been added before, check
                * if we have multiple devices in this plug. If so, make a
                * note to sort the list before dispatch.
                */
               if (list_empty(&plug->list))
                       trace_block_plug(q);
               else {
                       if (!plug->should_sort) {
                               struct request *__rq;

                               __rq = list_entry_rq(plug->list.prev);
                               if (__rq->q != q)
                                       plug->should_sort = 1;
                       }
                       if (request_count >= BLK_MAX_REQUEST_COUNT) {
                               blk_flush_plug_list(plug, false);
                               trace_block_plug(q);
                       }
               }
               list_add_tail(&req->queuelist, &plug->list);
               drive_stat_acct(req, 1);
       } else {
               spin_lock_irq(q->queue_lock);
               add_acct_request(q, req, where);
               __blk_run_queue(q);
out_unlock:
               spin_unlock_irq(q->queue_lock);
       }
}
EXPORT_SYMBOL_GPL(blk_queue_bio);       /* for device mapper only */


blk_queue_bio 尝试把bio合并进对垒的 request，或者放到新的requet里面去。
这一步做完 数据就从bio 放到request里面去了。  但这个request并不是马上发送到下面磁盘驱动的。应该是等那些电梯算法去调用， scsi_request_fn的时候，scsi_request_fn 才从队列里面去取出 request进行出来。到那个时候，request可能就已经包含多个bio了。
注意的是前面一段  generic_perform_write 里面调用  address_space操作的write_begin 函数的分析，只是有可能触发读取磁盘的bio操作。 写数据之前还是需要先读取磁盘来填充相应page cache的 buffer_head的。


现在回到 generic_perform_write 函数

static ssize_t generic_perform_write(struct file *file,
               status = a_ops->write_begin(file, mapping, pos, bytes, flags,
                                               &page, &fsdata);  //前面分析知道，这里吧page准备好了
               if (unlikely(status))
                       break;

               if (mapping_writably_mapped(mapping))
                       flush_dcache_page(page);

               pagefault_disable();
               copied = iov_iter_copy_from_user_atomic(page, i, offset, bytes);    //把用户空间数据复制到page cache的page里面去
               pagefault_enable();
               flush_dcache_page(page);

               mark_page_accessed(page);
               status = a_ops->write_end(file, mapping, pos, bytes, copied,
                                               page, fsdata);

现在看看  aops->write_end 对应的函数做了些什么。


static int ext4_ordered_write_end  然后又调用了下面这个函数
static int ext4_generic_write_end(struct file *file,
                                  struct address_space *mapping,
                                  loff_t pos, unsigned len, unsigned copied,
                                  struct page *page, void *fsdata)
{
        int i_size_changed = 0;
        struct inode *inode = mapping->host;
        handle_t *handle = ext4_journal_current_handle();

        copied = block_write_end(file, mapping, pos, len, copied, page, fsdata);

        /*
         * No need to use i_size_read() here, the i_size
         * cannot change under us because we hold i_mutex.
         *
         * But it's important to update i_size while still holding page lock:
         * page writeout could otherwise come in and zero beyond i_size.
         */
        if (pos + copied > inode->i_size) {
                i_size_write(inode, pos + copied);
                i_size_changed = 1;
        }

        if (pos + copied >  EXT4_I(inode)->i_disksize) {
                /* We need to mark inode dirty even if
                * new_i_size is less that inode->i_size
                * bu greater than i_disksize.(hint delalloc)
                */
               ext4_update_i_disksize(inode, (pos + copied));
               i_size_changed = 1;
       }
       unlock_page(page);
       page_cache_release(page);

       /*
        * Don't mark the inode dirty under page lock. First, it unnecessarily
        * makes the holding time of page lock longer. Second, it forces lock
        * ordering of page lock and transaction start for journaling
        * filesystems.
        */
       if (i_size_changed)
               ext4_mark_inode_dirty(handle, inode);   //修改了inode的,触发inode的 脏页写操作？？

       return copied;
}

int block_write_end(struct file *file, struct address_space *mapping,
                       loff_t pos, unsigned len, unsigned copied,
                       struct page *page, void *fsdata)
{
       struct inode *inode = mapping->host;
       unsigned start;

       start = pos & (PAGE_CACHE_SIZE - 1);

       if (unlikely(copied < len)) {
               /*
                * The buffers that were written will now be uptodate, so we
                * don't have to worry about a readpage reading them and
                * overwriting a partial write. However if we have encountered
                * a short write and only partially written into a buffer, it
                * will not be marked uptodate, so a readpage might come in and
                * destroy our partial write.
                *
                * Do the simplest thing, and just treat any short write to a
                * non uptodate page as a zero-length write, and force the
                * caller to redo the whole thing.
                */
               if (!PageUptodate(page))
                       copied = 0;

               page_zero_new_buffers(page, start+copied, start+len);
       }
       flush_dcache_page(page);

       /* This could be a short (even 0-length) commit */
       __block_commit_write(inode, page, start, start+copied);

       return copied;
}
EXPORT_SYMBOL(block_write_end);



static int __block_commit_write(struct inode *inode, struct page *page,
               unsigned from, unsigned to)
{
       unsigned block_start, block_end;
       int partial = 0;
       unsigned blocksize;
       struct buffer_head *bh, *head;

       blocksize = 1 << inode->i_blkbits;

       for(bh = head = page_buffers(page), block_start = 0;
           bh != head || !block_start;
           block_start=block_end, bh = bh->b_this_page) {
               block_end = block_start + blocksize;
               if (block_end <= from || block_start >= to) {
                       if (!buffer_uptodate(bh))
                               partial = 1;
               } else {
                       set_buffer_uptodate(bh);
                       mark_buffer_dirty(bh);        /// 设置脏页，page cache的 后台进程就会在适当的时候处理这些脏页，适当的时候放到bio里面发送给request queue里面？
               }
               clear_buffer_new(bh);
       }

       /*
        * If this is a partial write which happened to make all buffers
        * uptodate then we can optimize away a bogus readpage() for
        * the next read(). Here we 'discover' whether the page went
        * uptodate as a result of this (potentially partial) write.
        */
       if (!partial)
               SetPageUptodate(page);
       return 0;
}


 
mark_buffer_dirty  ->

http://lxr.linux.no/linux+v3.6.2/fs/buffer.c#L613
/*
 * Mark the page dirty, and set it dirty in the radix tree, and mark the inode
 * dirty.
 *
 * If warn is true, then emit a warning if the page is not uptodate and has
 * not been truncated.
 */
static void __set_page_dirty(struct page *page,
                struct address_space *mapping, int warn)
{
        spin_lock_irq(&mapping->tree_lock);
        if (page->mapping) {    /* Race with truncate? */
                WARN_ON_ONCE(warn && !PageUptodate(page));
                account_page_dirtied(page, mapping);
                radix_tree_tag_set(&mapping->page_tree,
                                page_index(page), PAGECACHE_TAG_DIRTY);   //设置对应的 radix tree上的 page 为dirty，后台进程根据树来扫描的吧？？
        }
        spin_unlock_irq(&mapping->tree_lock);
        __mark_inode_dirty(mapping->host, I_DIRTY_PAGES);    //inode 文件属性比如修改时间 也要导致磁盘写操作的？这个函数的说明 Put the inode on the super block's dirty list. 有需要的时候把inode接到一个列表里面去对
}



前面的整个generic_perform_write并没有马上把数据提交给磁盘驱动。
下面看看后台进程写数据的代码。

/*
* Handle writeback of dirty data for the device backed by this bdi. Also
* wakes up periodically and does kupdated style flushing.
*/
int bdi_writeback_thread(void *data)






void add_disk(struct gendisk *disk)
           ->  bdi_register  -> kthread_run(bdi_forker_thread， ->  kthread_create(bdi_writeback_thread
 
磁盘驱动往系统添加 gendisk的时候，add_disk 函数就为每个gendisk启动了 bdi_writeback_thread 内核线程了。不过根据 图书的说明，这个线程个数应该有的时候会自动增加的。



/*
 * Handle writeback of dirty data for the device backed by this bdi. Also
 * wakes up periodically and does kupdated style flushing.
 */
int bdi_writeback_thread(void *data)
      ->循环调用  pages_written = wb_do_writeback(wb, 0);
    

http://lxr.linux.no/linux+v3.6.2/fs/fs-writeback.c#L977
/*
* Retrieve work items and do the writeback they describe
*/
long wb_do_writeback(struct bdi_writeback *wb, int force_wait)
       ->wb_check_old_data_flush() 
                 ->wb_writeback()  -> queue_io ()  __writeback_inodes_wb（）
                              ->writeback_sb_inodes  
                                     ->__writeback_single_inode
                                              ->do_writepages



int do_writepages(struct address_space *mapping, struct writeback_control *wbc)
{
        int ret;

        if (wbc->nr_to_write <= 0)
                return 0;
        if (mapping->a_ops->writepages)    /// 调用  ext4_writepages
                ret = mapping->a_ops->writepages(mapping, wbc);
        else
                ret = generic_writepages(mapping, wbc);
        return ret;
}


--------------------------------------
ext4_writepages
    write_cache_pages
        ext4_writepage

ext4_writepage
        __block_write_begin
        block_commit_write
        block_write_full_page
             block_write_full_page_endio
                    __block_write_full_page
                               submit_bh       ///再调到这里就提交buffer_head 然后 bio了。这么前面有列举代码了。
                
     mpage_da_submit_io
                     submit_bh   
       
    有条件的，可以用kprobe 在 submit_bh   调用的时候backtrace 打印一下调用栈。

-------------------------------------------------------
blk_run_queue() ->
scsi_request_fn() -> scsi_dispatch_cmd ->  调用 底层scsi驱动注册的scsi host的 queuecommand 啊函数。

然后底层scsi 驱动在 自己的 queuecommand函数里面 得到 scsi request 和 cmd 再进行处理。 
```
