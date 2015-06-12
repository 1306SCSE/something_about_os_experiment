# 操作系统实验5 #

## 关于实验5 ##

实验5是的核心在于文件系统。这里假定你已经理解了各种概念。

## 实验三参考攻略（可能有错，仅供参考） ##

终于到第五个实验了，干掉全部6个实验已经指日可待了，让我们一起加油吧！
这个实验应该是代码量最少的一个实验了。

这次的指导书一上来就给出任务，也是醉了。我们直接照着指导书来。
首先阅读`user/serv.c`，这个文件是文件系统服务进程的入口。
我们可以看到，我们的小操作系统还是挺有趣的，因为文件系统的功能是以用户进程的形式实现的。
看起来，这个小操作系统是个微内核的架构，很多系统服务都是以用户进程的方式实现的。

用户进程阅读起来比较方便，因为它是按照我们平时的程序运行的顺序走的，
不像前几个实验，操作系统被中断驱动，跳转的过程很乱。`serv.c`首先执行了`serve_init()`

```c
void
serve_init(void)
{
    int i;
    u_int va;

    va = FILEVA;
    for (i=0; i<MAXOPEN; i++) {
        opentab[i].o_fileid = i;
        opentab[i].o_ff = (struct Filefd*)va;
        va += BY2PG;
    }
}
```

接下来调用`user/fs.c:fs_init()`来初始化文件系统。第一步是读取超级块。
读取的过程很有意思，首先调用`read_block()`来读一个磁盘块。

```c
// Make sure a particular disk block is loaded into memory.
// Return 0 on success, or a negative error code on error.
// 这个函数会确保一个磁盘块已经被载入到内存中。
// 成功返回0，否则返回一个负值。
// 
// If blk!=0, set *blk to the address of the block in memory.
// 如果blk!=0，那么会把*blk设置为磁盘块在内存中的地址。
//
// If isnew!=0, set *isnew to 0 if the block was already in memory,
//  or to 1 if the block was loaded off disk to satisfy this request.
// 如果isnew!=0，那么在磁盘块已经在内存中时，会将*isnew设置为0，
// 若磁盘块不在内存中，则会将对应磁盘块载入内存，同时*isnew设置为1.
//
// (Isnew lets callers like file_get_block clear any memory-only
// fields from the disk blocks when they come in off disk.)
//
// Hint: use diskaddr, block_is_mapped, syscall_mem_alloc, and ide_read.
int
read_block(u_int blockno, void **blk, u_int *isnew)
{
    int r;
    u_int va;

    if (super && blockno >= super->s_nblocks)
        user_panic("reading non-existent block %08x\n", blockno);

    if (bitmap && block_is_free(blockno))
        user_panic("reading free block %08x\n", blockno);

    // 计算磁盘块所对应的虚地址
    va = diskaddr(blockno);

    if(blk) *blk = va;
    if(block_is_mapped(blockno))    //如果磁盘块已经在内存中
    {
        if(isnew)
            *isnew = 0;
    }
    else                //如果磁盘块不在内存中，则将其内容载入到va处。
    {
        if(isnew)
            *isnew = 1;
        syscall_mem_alloc(0, va, PTE_V);
        writef("fs.c:read_block(): before ide_read() blockno:%x va:%x\n",blockno,va);
        ide_read(1, blockno*SECT2BLK, va, SECT2BLK);
    }
//  user_panic("read_block not implemented");
    writef("fs.c:read_block(): end !\n");
    return 0;
}
```

回到`read_super()`，我们可以看到，超级块已经被载入到了blk处。然后，将超级块的设置为blk。
这时，super已经在内存里了，我们可以直接通过`super->s_magic`一类的方式访问超级块的信息，
是不是很有趣呢？

```c
if ((r = read_block(1, &blk, 0)) < 0)
    user_panic("cannot read superblock: %e", r);
writef("fs.c:read_super() 2\n");
// 到这里超级块已经在内存里了！
super = blk;
// 直接将blk强制转型成super指针后访问
if (super->s_magic != FS_MAGIC)
    user_panic("bad file system magic number %x %x",super->s_magic,FS_MAGIC);

if (super->s_nblocks > DISKMAX/BY2BLK)
    user_panic("file system is too large");
```

之后的`check_write_block()`没什么太有意思的，就是验证磁盘的写功能是否正常。
最后，调用`read_bitmap()`读取所有的bitmap。

我们再来看看`serv`这个程序的最核心的部分`serv()`。

```c
void
serve(void)
{
    u_int req, whom, perm;
    for(;;) {
        perm = 0;
//writef("serve:come 0\n");
        // 不断循环获得请求，具体的请求信息储存在REQVA中。
        req = ipc_recv(&whom, REQVA, &perm);
//writef("serve:come 1\n");
        if (debug)
            writef("fs req %d from %08x [page %08x: %s]\n",
                req, whom, vpt[VPN(REQVA)], REQVA);

        // All requests must contain an argument page
        if (!(perm & PTE_V)) {
            writef("Invalid request from %08x: no argument page\n",
                whom);
            continue; // just leave it hanging...
        }

        // 判断请求的类型，根据请求的类型进行对应的服务。
        switch (req) {
        case FSREQ_OPEN:
            serve_open(whom, (struct Fsreq_open*)REQVA);
            break;
        case FSREQ_MAP:
            serve_map(whom, (struct Fsreq_map*)REQVA);
            break;
        case FSREQ_SET_SIZE:
            serve_set_size(whom, (struct Fsreq_set_size*)REQVA);
            break;
        case FSREQ_CLOSE:
            serve_close(whom, (struct Fsreq_close*)REQVA);
            break;
        case FSREQ_DIRTY:
            serve_dirty(whom, (struct Fsreq_dirty*)REQVA);
            break;
        case FSREQ_REMOVE:
            serve_remove(whom, (struct Fsreq_remove*)REQVA);
            break;
        case FSREQ_SYNC:
            serve_sync(whom);
            break;
        default:
            writef("Invalid request code %d from %08x\n", whom, req);
            break;
        }
        syscall_mem_unmap(0, REQVA);
    }
}
```

好吧，从这个文件看不出关于任务一的更多信息了。我们还是回到`user/fs.c`中吧。
这次我们简单直接一点，直接搜索File结构体。

首先找到了`file_block_walk()`，里面有两处一处是`ptr = &f->f_direct[filebno]`，一处是`f->f_indirect`。
感觉结构体里出现定长数组的可能性不大，此时笔者猜测`f->f_direct`是一个指针或者是一个数组。
再后面有一个`dir_lookup()`，里面有几处使用了File结构体。`nblock = dir->f_size / BY2BLK`和`f[j].f_dir = dir`。
之后，`walk_path()`中有`if (dir->f_type != FTYPE_DIR)`。后面，`file_create`中有`f->f_name`。
再往后，`file_close()`中有`f->f_dir`。

将这些域填入结构体中。f_name数组的大小为`MAXNAMELEN`。f_direct数组的大小为`NDIRECT`。
至于具体的类型，大家看简单看一下这些域的用法就能知道，就不再赘述了。

之后根据`serv.c`中的注释，整个结构体的大小为256，所以我们还需要一个域作填充。
前面的部分的大小不够256，需要额外填充一些字节（至于填充多少，大家先输出下sizeof(File)，然后差多少填充多少就好了）。
总之最终就是让sizeof(File)==256成立就OK了。

任务二是去填serv.c中使用的那些结构体。
我们直接从上往下挨个函数找，看到一个往对应结构体里填写一个就好，这里就不再赘述了。类型几乎都是u_int。
这个查看调用处的函数声明、局部变量类型等等就能推测处理。

任务三实现我们上次没管的通过dstva以及srcva来传递数据的那个功能。比较奇怪的是，貌似这个已经被实现了？
读了下这次助教发的代码，好像已经实现了这个功能。

通过`ENV_CREATE`创建一个进程user_serv。serv的偏移是0x80，入口地址是UTEXT。也就是说，从binary+0x80开始拷，
初始pc设置为UTEXT。最后，时钟信号被注释掉了，感觉是怕lab4中那个没人弄清楚的玄学问题骚扰lab5。
我们有两种解决方法，一个是的`lib/kclock_asm.S`中把`sb t0, 0xb5000100`前面的注释符删掉。
以打开时钟信号。另一个是在`init/init.c`中手工调用一次`sched_yield`，也就是手工执行一次调度。
这两种方法都行，应该问题不大。即使选择打开时钟信号，也没有碰上什么问题。

另外还有个问题是`user/serv.c`中那个`fs_test`没有实现。
（笔者选择直接懒懒地注释掉那句`fs_test()`，因为并不知道如何测试文件系统）
这里老师的建议是希望大家自己补充上。

最后，执行`make clean && make`重编译后，再执行`gxemul -E testmips -C R3000 -M 64 gxemul/vmlinux -d gxemul/fs.img`
运行就好了。输出结果应该如下。

```
main.c: main is start ...

init.c: mips_init() is called

Physical memory: 65536K available, base = 65536K, extended = 0K

to memory 80401000 for struct page directory.

to memory 80431000 for struct Pages.

mips_vm_init:boot_pgdir is 80400000

pmap.c:  mips vm init success

panic at init.c:29: ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

????DEBUG: sizeof(struct File)=256

?FS is running

FS can do I/O

????fs.c:read_super() 1

?sys_mem_alloc(): envid:0  va10001000  perm:200

fs.c:read_block(): before ide_read() blockno:1 va:10001000

ide.c: read_sector() offset=1000

?pageout:       @@@___0x10001000___@@@  ins a page 

ide.c: read_sector() offset=1200

ide.c: read_sector() offset=1400

ide.c: read_sector() offset=1600

ide.c: read_sector() offset=1800

ide.c: read_sector() offset=1a00

ide.c: read_sector() offset=1c00

ide.c: read_sector() offset=1e00

fs.c:read_block(): end !

fs.c:read_super() 2

superblock is good

?sys_mem_alloc(): envid:0  va10000000  perm:200

fs.c:read_block(): before ide_read() blockno:0 va:10000000

ide.c: read_sector() offset=0

?pageout:       @@@___0x10000000___@@@  ins a page 

ide.c: read_sector() offset=200

ide.c: read_sector() offset=400

ide.c: read_sector() offset=600

ide.c: read_sector() offset=800

ide.c: read_sector() offset=a00

ide.c: read_sector() offset=c00

ide.c: read_sector() offset=e00

fs.c:read_block(): end !

writeBLK():  blockno:1  va:10001000

ide_write(): offset_begin:1000 offset:0 src:10001000

ide_write(): offset_begin:1000 offset:200 src:10001000

ide_write(): offset_begin:1000 offset:400 src:10001000

ide_write(): offset_begin:1000 offset:600 src:10001000

ide_write(): offset_begin:1000 offset:800 src:10001000

ide_write(): offset_begin:1000 offset:a00 src:10001000

ide_write(): offset_begin:1000 offset:c00 src:10001000

ide_write(): offset_begin:1000 offset:e00 src:10001000

sys_mem_alloc(): envid:0  va10001000  perm:200

fs.c:read_block(): before ide_read() blockno:1 va:10001000

ide.c: read_sector() offset=1000

?pageout:       @@@___0x10001000___@@@  ins a page 

ide.c: read_sector() offset=1200

ide.c: read_sector() offset=1400

ide.c: read_sector() offset=1600

ide.c: read_sector() offset=1800

ide.c: read_sector() offset=1a00

ide.c: read_sector() offset=1c00

ide.c: read_sector() offset=1e00

fs.c:read_block(): end !

writeBLK():  blockno:1  va:10001000

ide_write(): offset_begin:1000 offset:0 src:10001000

ide_write(): offset_begin:1000 offset:200 src:10001000

ide_write(): offset_begin:1000 offset:400 src:10001000

ide_write(): offset_begin:1000 offset:600 src:10001000

ide_write(): offset_begin:1000 offset:800 src:10001000

ide_write(): offset_begin:1000 offset:a00 src:10001000

ide_write(): offset_begin:1000 offset:c00 src:10001000

ide_write(): offset_begin:1000 offset:e00 src:10001000

write_block is good

?sys_mem_alloc(): envid:0  va10002000  perm:200

fs.c:read_block(): before ide_read() blockno:2 va:10002000

ide.c: read_sector() offset=2000

?pageout:       @@@___0x10002000___@@@  ins a page 

ide.c: read_sector() offset=2200

ide.c: read_sector() offset=2400

ide.c: read_sector() offset=2600

ide.c: read_sector() offset=2800

ide.c: read_sector() offset=2a00

ide.c: read_sector() offset=2c00

ide.c: read_sector() offset=2e0

fs.c:read_block(): end !

read_bitmap is good

[ warning: LOW reference: vaddr=0x000000bc, exception TLBL, pc=0x00401834 <(no symbol)> ]
?panic at pmap.c:615: ^^^^^^TOO LOW^^^^^^^^^
```

最后出现那个panic应该是正常的。因为助教给的`sys_ipc_recv`好像存在问题（为了配合没时钟中断的缘故吧）。
应该不管它也无所谓。

如果想解决这个问题，在打开时钟的情况下（也就是修改了kclock_asm.S），
将`lib/syscall_all.c`修改如下就不会出现panic了。

```c
void sys_ipc_recv(int sysno,u_int dstva)
{
    ///////////////////////////////////////////////////////
    //your code here
    //
        if ((unsigned int)dstva >= UTOP || dstva != ROUNDDOWN(dstva, BY2PG))
                return -E_INVAL;

        curenv->env_ipc_dstva = dstva;
        curenv->env_ipc_recving = 1;
        curenv->env_status = ENV_NOT_RUNNABLE;
        // set the return value to be zero,
        // it is necessary, because the 'return' statement
        // after 'sched_yield' will never be executed,
        // actually it is skipped.
        //curenv->env_tf.tf_regs.reg_eax = 0;
        curenv->env_tf.regs[2] = 0;
        // give up the CPU
        sys_yield();
        //return 0;

    ///////////////////////////////////////////////////////
}
```

会停在`read_bitmap is good`这里，然后一直死循环。因为没有任何用户进程去请求文件系统操作，
因此它会一直死循环等待请求，就是这样。

祝大家实验愉快！

## 可能遇到的错误 ##

### panic at serv.c:314 ###

错误输出：

```
main.c: main is start ...
init.c: mips_init() is called
Physical memory: 65536K available, base = 65536K, extended = 0K
to memory 80401000 for struct page directory.
to memory 80431000 for struct Pages.
mips_vm_init:boot_pgdir is 80400000
pmap.c:  mips vm init success
panic at init.c:29: ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
????panic at serv.c:314: ?assertion failed: sizeof(struct File)==256
```

说明File结构体的大小有问题，一般是忘了填写填充块。