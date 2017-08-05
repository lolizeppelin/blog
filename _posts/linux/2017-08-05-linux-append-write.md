---
layout: post
title:  "Linux 追加写操作的原子性"
date:   2017-08-05 15:05:00 +0800
categories: "系统基础"
tag: ["linux"]
---

* content
{:toc}



因为openstack的log代码里没有任何多进程写日志的处理代码,所以花了一点时间确认了下

先是查到[这个](http://www.jianshu.com/p/b5a731940ff9)

    于是当两个线程或进程在同一时间写同一文件时，我们可以肯定他们的数据不会交错插入。
    这使调用write(2)不需要互斥锁，而且还能保证原子性，因为内核已经帮我们做到那个了。

    但是还有一个问题，一般写文件时，你需要找到你想插入的具体的位置，然后再进行真正的写入操作。
    不幸的是，这会涉及到两个系统调用，lseek(2)和write(2).他们各自是原子性的，但是两个操作作为一个整体就不是原子性的


具体代码确认在[这里](http://www.pagefault.info/?p=139), 转载自pagefault

    先来描述一下，write系统调用的大体流程，首先内核会取得对应的文件偏移，
    然后调用vfs的write操作，而在vfs层的write操作的时候会调用对应文件系统的write方法，
    而在对应文件系统的write方法中aio_write方法，最终会调用底层驱动。
    这里有一个需要注意的就是内核在写文件的时候会加一把锁(有些设备不会加锁，比如块设备以及裸设备).
    这样也就是说一个文件只可能同时只有一个进程在写。而且快设备是不支持append写的。

    而这里append的原子操作的实现很简单，由于每次写文件只可能是一个进程操作(计算文件偏移并不包含在锁里面)，
    而append操作是每次写到末尾(其他类型的写是先取得pos才进入临界区，
    而这个时间内有可能pos已经被其他进程改变，而append的pos的计算是包含在锁里面的),因此自然append就是原子的



---

具体代码分析

    因此可以看到最关键的操作都是放在aio_write中，也就是generic_file_aio_write，
    这个函数我们可以看到在调用具体的实现__generic_file_aio_write之前会加一把锁(i_mutex),
    这样就保证了一个文件同时只会有一个进程来写。

```c
ssize_t generic_file_aio_write(struct kiocb *iocb, const struct iovec *iov,
        unsigned long nr_segs, loff_t pos)
{
    struct file *file = iocb->ki_filp;
    struct inode *inode = file->f_mapping->host;
    ssize_t ret;

    BUG_ON(iocb->ki_pos != pos);
//加锁
    mutex_lock(&inode->i_mutex);
//调用具体的实现
    ret = __generic_file_aio_write(iocb, iov, nr_segs, &iocb->ki_pos);
//释放锁
    mutex_unlock(&inode->i_mutex);

    if (ret > 0 || ret == -EIOCBQUEUED) {
        ssize_t err;

        err = generic_write_sync(file, pos, ret);
        if (err < 0 && ret > 0)
            ret = err;
    }
    return ret;
}

```

    上面可以看到先加锁然后调用__generic_file_aio_write，
    而对应的blkdev_aio_write则是直接调用__generic_file_aio_write，也就是不用加锁，
    下面就是内核里面的注释：
    * It expects i_mutex to be grabbed unless we work on a block device or similar
    * object which does not need locking at all.


## 总结：

    linux下使用O_APPEND打开文件后fork出的多个子进程对文件的追加写都是原子性的

    非O_APPEND打开文件的情况下
    获取到pos以后需要获取到锁才能写入并修改pos值
    只要没有做过seek,那么写入操作也是原子性的
