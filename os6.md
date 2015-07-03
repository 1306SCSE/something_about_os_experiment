# 操作系统实验6 #

## 关于实验6 ##

实验6关于管道的设计和shell，其实是学期大总结。你可以第一次感受到操作系统的运行。本次实验助教替换了一些关键文件，包括但不限于`tlb_asm.S`、`syscall_all.c`。

## 管道部分 ##

指导书要求的代码有两个文件，`user/pipe.c`和`user/sh.c`。先看管道部分：本实验划分一个内存区域在进程间共享，分配文件控制块映射到这个内存区域，因此这个区域就是管道。我们把管道视作文件，为了标记两个进程不同的读写状态，这个内存区域用两个不同的文件控制块来管理。

管道是置于内存中的文件，内容如下
```c
struct Pipe {
	u_int p_rpos;			// read position
	u_int p_wpos;			// write position
	u_char p_buf[BY2PIPE];	// data buffer
};
```
这个管道必须要在两个进程间共享，因此引入共享内存权限标志`PTE_LIBRIRAY`。一个管道文件对应读文件控制块，即编号`p[0]`的文件控制块和写文件控制块，即编号`p[1]`的控制块。

为了设计管道，首先需要了解管道如何使用。测试管道是否创建正确的程序是`user/testpipe.c`和`user/testpiperace.c`。以`testpipe.c`为例
```c
void
umain(void)
{
    char buf[100];
    int i, pid, p[2];

    if ((i=pipe(p)) < 0)
        user_panic("pipe: %e", i);
    // 根据函数命名的一般法则，pipe是管道的构造函数。
    // 管道结构放入数组p中，数组p就是访问管道的方式。
    // p[0]和p[1]是两个文件块编号，可以用来定位两个文件控制块的地址。
    // 一个文件不可能被两个进程以不同读写方式同时打开。
    // 管道需要一个进程读，一个进程写，因此需要两个文件控制块。
    // 定义p[0]是管道的读文件控制块编号，p[1]是管道的写文件控制块编号。

    if ((pid=fork()) < 0)
        user_panic("fork: %e", i);
    // 按照之前fork的copy-on-write设计，一旦动了文件控制块，两个进程
    // 不能继续共享，而是新建一页，那么辛苦映射好的管道无济于事

    if (pid == 0) {
        writef("[%08x] pipereadeof close %d\n", env->env_id, p[1]);
        close(p[1]);
        // 子进程关闭写文件p[1]
        writef("[%08x] pipereadeof readn %d\n", env->env_id, p[0]);
        i = readn(p[0], buf, sizeof buf-1);
        // 子进程从p[0]读
        // 此处省略一些
    } else {
        //user_panic("par((((((((((((((((((((");
        writef("[%08x] pipereadeof close %d\n", env->env_id, p[0]);
        close(p[0]);
        // 父进程关闭读文件p[0]
        writef("[%08x] pipereadeof write %d\n", env->env_id, p[1]);
        // 父进程向p[1]写
        if ((i=write(p[1], msg, strlen(msg))) != strlen(msg))
            user_panic("write: %e", i);
        close(p[1]);
        // 父进程关闭写文件p[1]
    }

    //此处省略一些
}
```
从这个测试程序不难发现，`user/testpipe.c`做的测试是父进程向子进程通过管道传输。由于管道通过文件控制块传输，构造函数`pipe(int p[2])`创建的文件控制块必须能够在`fork()`函数完成内存映射的时候做到进程间共享。为了完成这一点，需要阅读管道构造函数`pipe(int p[2])`。
```c
int
pipe(int pfd[2])
{
	int r, va;
	struct Fd *fd0, *fd1;

	// 分配文件控制块，注意特殊内存权限PTE_LIBRARY
	if ((r = fd_alloc(&fd0)) < 0
	||  (r = syscall_mem_alloc(0, (u_int)fd0, PTE_V|PTE_R|PTE_LIBRARY)) < 0)
		goto err;

	if ((r = fd_alloc(&fd1)) < 0
	||  (r = syscall_mem_alloc(0, (u_int)fd1, PTE_V|PTE_R|PTE_LIBRARY)) < 0)
		goto err1;

	// 把两个文件控制块对应的第一页数据块作为struct Pipe，即管道结构体。
	va = fd2data(fd0);
	if ((r = syscall_mem_alloc(0, va, PTE_V|PTE_R|PTE_LIBRARY)) < 0)
		goto err2;
	if ((r = syscall_mem_map(0, va, 0, fd2data(fd1), PTE_V|PTE_R|PTE_LIBRARY)) < 0)
		goto err3;

	// 设定管道结构
	fd0->fd_dev_id = devpipe.dev_id;
	fd0->fd_omode = O_RDONLY;
    // 可以看出p[0]是只读文件

	fd1->fd_dev_id = devpipe.dev_id;
	fd1->fd_omode = O_WRONLY;
    // 可以看出p[1]是只写文件

	// 此处省略一些
    // 输出的是文件块编号
	pfd[0] = fd2num(fd0);
	pfd[1] = fd2num(fd1);
	return 0;
    
    // 此处省略一些
}
```
由上面的程序我们知道`fork()`在处理`PTE_LIBRARY`权限时不能设定为COW而是直接共享映射。因此`fork()`调用的`duppage`要注意对`PTE_LIBRARY`权限的处理。

这里我们已经涉及到了文件控制块的一些重要内容，它们可以在`user/fd.h`中找到。
```c
int fd2num(struct Fd*);		// 把文件控制块地址变成文件控制块编号
int num2fd(int fd);			// 把文件控制块编号变成文件控制块地址
u_int fd2data(struct Fd*);	// 把文件控制块地址变成文件对应数据块地址
```
有了这些知识，可以进入管道的设计。判断管道关闭的函数`static int _pipeisclosed(struct Fd *fd, struct Pipe *p)`通过判断读文件或写文件的文件控制块引用次数`pp_ref`和管道文件的引用次数`pp_ref`来判断是否管道的另一端已经关闭。以读文件为例，正常建立管道之后`pageref(num2fd(p[0]))<pageref(fd2data(num2fd(p[0])))`。现在传入一个`*fd`也就是`num2fd(p[0])`那么如果`pageref(fd)=pageref(fd2data(fd))`则管道的另一端已经关闭。注意`p[0]`和`p[1]`都只是文件控制块编号，编号先要变成地址，地址还得变成管道（对应数据块）地址。还要注意判断管道关闭是原子操作，需要用`env_run`控制。

接下来完成管道读写操作。用户调用`read(int fdnum, void *buf, u_int n)`从`fdnum`编号的文件里面读文件。要从一个只知道文件控制块编号的东西里面读文件，首先得知道这个文件控制块地址，所以调用`fd_lookup(fdnum, &fd)`找到文件控制块地址，找到文件控制块地址，还得知道文件在什么设备上，文件控制块的成员变量`fd->dev_id`记录设备编号`dev_id`，有了设备编号`dev_id`还得找到设备控制块的地址，调用`dev_lookup(fd->fd_dev_id, &dev)`找到设备控制块。不同设备需要有不同的读写函数，因此设备控制块保存了不同设备的读写函数的指针。比如管道的设备控制块
```c
struct Dev devpipe =
{
.dev_id=	'p',
.dev_name=	"pipe",
.dev_read=	piperead,
.dev_write=	pipewrite,
.dev_close=	pipeclose,
.dev_stat=	pipestat,
};
```
因此对于管道设备来说，用户调用`read(p[0])`最后会转到`piperead`处理。

此部分仅需完成设备`devpipe`的三个函数，管道读`piperead`、管道写`pipewrite`，管道统计`pipestat`。注释均有详细的指导。管道设计成循环或者不循环都可以。

管道读的注释如下
```c
static int
piperead(struct Fd *fd, void *vbuf, u_int n, u_int offset)
{
	// Your code here.  See the lab text for a description of
	// what piperead needs to do.  Write a loop that 
	// transfers one byte at a time.  If you decide you need
	// to yield (because the pipe is empty), only yield if
	// you have not yet copied any bytes.  (If you have copied
	// some bytes, return what you have instead of yielding.)
	// If the pipe is empty and closed and you didn't copy any data out, return 0.
	// Use _pipeisclosed to check whether the pipe is closed.
}
```
每次只传输一个字节，当管道空`p_rpos>=p_wpos`时检查管道是否已经关闭，关闭时返回；或者读取到足够多字节时返回。如果某一次没有读到任何数据，可以考虑`syscall_yield()`。`testpipe`调用`piperead`时设定字符上限是99，因此只可能是管道另一端（写入段）关闭时关闭，这可以保证读比写后结束。

管道写的注释如下
```c
static int
pipewrite(struct Fd *fd, const void *vbuf, u_int n, u_int offset)
{
	// Your code here.  See the lab text for a description of what 
	// pipewrite needs to do.  Write a loop that transfers one byte
	// at a time.  Unlike in read, it is not okay to write only some
	// of the data.  If the pipe fills and you've only copied some of
	// the data, wait for the pipe to empty and then keep copying.
	// If the pipe is full and closed, return 0.
	// Use _pipeisclosed to check whether the pipe is closed.
}
```
每次只传输一个字节。管道写满`p_wpos-p_rpos>=BY2PIPE`时必须等待读进程增加`p_rpos`，如果没写完的情况下管道关闭了，就返回错误`0`。

管道统计函数`pipestat(struct Fd *fd, struct Stat *stat)`的内容就是把`struct Fd`对应到`struct Stat`。

接下来在`init/init.c`中先后填入`ENV_CREATE(user_testpipe)`和`ENV_CREATE(user_testpiperace)`完成测试，`icode`的起始地址仍然是`UTEXT+0x1000`，测试通过的标志可以在对应的文件中找到。两个测试分别看到`pipe tests passed`以及`race didn't happen`之后就可以进入下一步了。

## Shell部分 ##
Shell部分所需代码很少，都在`user/sh.c`中，需要完成的部分是重定向符号和管道符号的解析。

输出重定向部分自然可以参照输入重定向了。
```c
		case '<': // input indirection
			// 取输入文件名
			if(gettoken(0, &t) != 'w'){
				writef("syntax error: < not followed by word\n");
				exit();
			}
			// 打开文件名对应文件
			fdnum = open(t,O_RDONLY);
			// 把这个文件复制到标准输入文件（0号文件）里
			dup(fdnum,0);
			// 关闭这个文件
			close(fdnum);
			break;
```
不难写出输出重定向的解析。

至于管道符号，遇到管道符号，自然需要启动两个进程，可以考虑把shell给fork了，父子shell启动父子进程。这一段有一串详细的注释
```c
			// Your code here.
			// 	First, allocate a pipe.
			//	Then fork.
			//	the child runs the right side of the pipe:
			//		dup the read end of the pipe onto 0
			//		close the read end of the pipe
			//		close the write end of the pipe
			//		goto again, to parse the rest of the command line
			//	the parent runs the left side of the pipe:
			//		dup the write end of the pipe onto 1
			//		close the write end of the pipe
			//		close the read end of the pipe
			//		set "rightpipe" to the child envid
			//		goto runit, to execute this piece of the pipeline
			//			and then wait for the right side to finish
```
可以知道就是把shell给fork之后，父进程的控制的写文件复制到标准输出文件（1号文件）里，子进程控制的读文件复制到标准输入文件（0号文件），然后各自解析命令并运行程序。

启动此系统需要在`init/init.c`中创建`user_icode`和`fs_serv`进程，需要注意的是文件系统读写`user/fsipc.c`中所有的信号均发送给`envs[1]`，所以请确保`fs_serv`进程对应`envs[1]`进程控制块。不出意外，测试时就会出现
```
This is /motd, the message of the day.
Welcome to the 6.828 kernel, now with a file system!
```
以及
```
:::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::

::                                                         ::

::              Super Shell  V0.0.0_1                      ::

::                                                         ::

:::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::

$
```
这时可以输入`ls.b`会看到正常的`ls`命令结果，就说明shell能使用。

## Happy hacking! ##
如果`pageref(fd)`工作不正常，可以检查`mm/pmap.c`中`boot_map_segment()`函数是否缺少
```c
boot_map_segment(pgdir,UPAGES,npage*sizeof(struct Page),PADDR(pages),PTE_R);
```
如果缺少无法通过`pages`数组访问函数。（感谢罗天歌、李开意大神）

如果`fork()`出来的进程栈出了问题，请检查`duppage()`函数运用时是否将`UXSTACKTOP`这一页映射掉。这是异常返回栈，不能映射。映射到`UTOP-BY2PG`或者`UXSTACKTOP`都可以。（感谢何涛、罗天歌、李开意大神）

如果`testpipe`对了一半，也就是前一半读测试正确，但是后一半写测试`panic at pmap.c`的话，有可能要检查`pmap.c`的`page_alloc()`函数，如果调用了`static void page_initpp(struct Page *pp)`就请注意了，`sizeof(struct Page) != BY2PG`。所以最好使用`void bzero(void *, size_t);`来初始化页面。（感谢罗天歌、李开意大神）
