# 操作系统实验4 #

## 关于实验4 ##

实验4要实现两部分内容，一部分是IPC（进程间通信）机制，另一部分是系统调用。
这里假定你已经看过了老师给的指导书。

## 实验四参考攻略（可能有错，仅供参考） ##

那个。。。这次的实验简直就是玄学。。。最后调出来的时候也不知道为啥。
还有一些问题没有完全解决，所以这次尽量把我所有的分析过程写出来，尽可能地提供些思路。
另外，为了更方便大家调试，这次力争把调试过程中遇到的错误，调试技巧
以及相关分析和解决方案写出来。也欢迎大家补充（Pull-Request或者找有维护权限的维护者代发均可）。

首先先道个歉，由于笔者的个人能力和知识储备不足，导致上次的实验攻略中出现一处致命错误，误导了大家。
且这个错误会影响到这次实验的完成，在此表示很抱歉。
错误在于`lib/env.c`中的`env_run`。上次对于`env_pop_tf`的分析不够深入，
导致了一个错误。我们重新来分析一下`env_pop_tf`。我们先来看一下`lib/env.c`中的接口定义：

```c
extern void env_pop_tf(struct Trapframe *tf,int id);
```

这个`id`具体传什么值我们通过分析汇编代码来完成。

```
LEAF(env_pop_tf)
.set    mips1          
        //1:    j   1b
        nop    
        // a0寄存器中储存了参数tf，a1储存了id。
        move        k0,a0 
        mtc0    a1,CP0_ENTRYHI
        // 以下略去
END(env_pop_tf)
```

可以看到，对于参数id，`env_pop_tf`把它赋值给了cp0中的entryhi寄存器。那么这个寄存器是干什么用的呢？
我们先来考虑这样一个问题，不同的进程都有其独有的逻辑地址空间。那么当我们切换进程的时候，
我们需要将当前页表切换为对应进程的页表。而TLB相当于页表的高速缓存，每次切换页表以后，
TLB中原有的内容是旧进程的，如果我们不把它清理掉的话，就相当于在用旧进程的页表做地址转换，
这样显然是不行的。但每次一切换进程就清空TLB效率太低，也会极大地降低命中率。
因此，MIPS的TLB表项中还包含了一个域叫做ASID，用于标识当前表项所属的进程。
有了它，我们就不需要每次把TLB清空了。由于不同进程的ASID不同，因此只要告诉CPU当前进程的ASID是多少，
那么CPU在做转换的时候就会忽略TLB中ASID与当前进程不同的项。这样就无需清空TLB了，可以提高效率。
而如何告诉CPU当前进程的ASID是多少呢？没错，正是这个神奇的cp0_entryhi寄存器。
更详细的说明，看可以参考《MIPS体系结构透视》。

所以，进过上述分析，我们可以知道，参数id实际上应该是进程的ASID，而不是简单的env_id。
因为它被`env_pop_tf`直接填到cp0_entryhi寄存器里去了。如何获得ASID呢？
在`include/env.h`中有一个宏`GET_ENV_ASID`。所以，这里应该改为

```c
env_pop_tf(&curenv->env_tf, GET_ENV_ASID(curenv->env_id));
```

至此，实验3的遗留问题我们就处理完了。

OK，那么正式开始！这次的实验有些摸不着头脑，因为init.c完全没有变化。
说明我们这次填写的代码全部是服务于进程的代码，因而难以再通过静态的顺序读代码的方式来理清关系。
那么，我们从哪里开始读呢？根据网上对于阅读内核的一些文章，笔者选择从MakeFile开始。
MakeFile表现了代码如何生成可执行文件。因此，MakeFile就像一张地图，可以为我们指引方向。

首先是最顶层的MakeFile（仅截取有用部分）:

```Makefile
# 多了个user_dir
drivers_dir       := drivers
boot_dir          := boot
user_dir          := user
init_dir          := init
lib_dir           := lib

# 同样，也是多了user

modules           := boot drivers init lib mm user

# 多了个$(user_dir)/*.x
objects           := $(boot_dir)/start.o                          \
                                 $(init_dir)/main.o                       \
                                 $(init_dir)/init.o                       \
                                 $(init_dir)/code.o                       \
                                 $(drivers_dir)/gxconsole/console.o \
                                 $(lib_dir)/*.o                           \
                                 $(user_dir)/*.x \
                                 $(mm_dir)/*.o
```

于是，我们来探究一下，这个`user`文件夹里面有什么东西。结果。。。我们发现有密密麻麻一大串文件。
好吧，接着找我们的地图，user下面的Makefile：

```Makefile
# For embedding one program in another
# 将一个程序嵌入另一个程序中

USERLIB := printf.o \
                print.o \
                libos.o \
                fork.o \
                pgfault.o \
                syscall_lib.o \
                ipc.o \
                string.o \
                fd.o \
                pageref.o \
                file.o \
                fsipc.o \
                wait.o \
                spawn.o \
                pipe.o \
                console.o \
                fprintf.o

# -nostdlib 关闭C语言标准库， -static 静态链接程序
CFLAGS += -nostdlib -static

# 可以看到，这里把一堆的.b、.x和.o文件编译到了一起。
all: fktest.x fktest.b testfdsharing.x testfdsharing.b pingpong.x pingpong.b idle.x testspawn.x testarg.b testpipe.x testpiperace.x icode.x init.b sh.b cat.b ls.b $(USERLIB) entry.o syscall_wrap.o

# 这是.x文件的生成规则
# %<代表第一个依赖文件，也就是%b.c
# %@代表目标文件，也就是%.x
# 所以这里就是编译所有的%b.c，输出文件名为%.x（%相当于通配吧。。。）
%.x: %.b.c
        echo cc1 $<
        $(CC) $(CFLAGS) -c -o $@ $<

# 这里是$b.c的生成规则，大致就是要把%.b的内容转换为C数组
# （就是上次实验中binary_user_a之类的数组）。
# 并最终生成类似code_a.c那样的C文件
%.b.c: %.b
        echo create $@
        echo bintoc $* $< > $@~
        ./bintoc $* $< > $@~ && mv -f $@~ $@
#       grep \. $@

# 编译出%.b二进制文件
%.b: entry.o syscall_wrap.o %.o $(USERLIB)
        echo ld $@
        $(LD) -o $@ $(LDFLAGS) -G 0 -static -n -nostdlib -T ./user.lds $^

# 将C文件编译为.o文件
%.o: %.c
        echo user1 cc $<
        $(CC) $(CFLAGS) $(INCLUDES) -c -o $@ $<

# 将汇编文件编译为.o文件
%.o: %.S
        echo as $<
        $(CC) $(CFLAGS) $(INCLUDES) -c -o $@ $<

%.o: lib.h

.PHONY: clean

clean:
        rm -rf *~ *.o *.b.c *.x *.b

include ../include.mk
```

好吧，于是我们发现这个目录相当于把一堆用户程序嵌入到了内核中（以类似上次的code_a.c的形式）。
这个新的目录就探索到这里，直接能够发现的新家伙差不多就是这些了。接下来，我们借助指导书，
进入我们第一个要填写的函数。

> void sys_ipc_recv(int sysno,u_int dstva)：设置当前进程为可接收状态，然后使当
前进程不可运行，暂停当前进程调度，调度其它进程占用 cpu。其中参数 dstva
是收到的页面的虚拟地址。暂停当前进程调度后，需要调用 sched_yield 函数
重新进行进程调度。 

这里我们再来回顾一下Env结构体的定义（翻译摘自指导书）：

```c
struct Env {
    // ...

    // Lab 4 IPC
    u_int env_ipc_value;            // 进程接收的别的进程发送来的值
    u_int env_ipc_from;             // 发送进程的 ID
    u_int env_ipc_recving;          // 为 0 时表示进程不接收发送来的信息
    u_int env_ipc_dstva;        // 用来映射收到的页面的虚拟地址
    u_int env_ipc_perm;     // 收到的页面映射

    // Lab 4 fault handling
    u_int env_pgfault_handler;      // 页中断
    u_int env_xstacktop;            // 异常栈栈顶指针

    // ...
};
```

好，那么到这里以后，直接参照上面的结构体定义和注释（源代码中的英文注释也许更为清晰），
设定`curenv`的状态并调用`sched_yield`重新调度就好。

**但这里请注意，笔者认为这里注释有问题** 我们不应该调用`sched_yield()`，而应该调用`sys_yield()`。
为什么呢？因为`sys_yield()`中做了一个很关键的步骤

```c
void sys_yield(void)
{
    // 这里是关键：sys_yield保存了当前运行环境到TIMESTACK-sizeof(struct Trapframe)，以供sched_yield()使用。
    bcopy((int)KERNEL_SP-sizeof(struct Trapframe),TIMESTACK-sizeof(struct Trapframe),sizeof(struct Trapframe));
    sched_yield();
}
```

显然，调用`sched_yield()`的话需要将当前进程的运行环境保存至`TIMESTACK-sizeof(struct Trapframe)`，
否则`env_run`中的进程切换就会出错。而在`sys_ipc_recv()` 中我们并没有这么做，
所以笔者认为应该直接调用`sys_yield()`来重新进行进程调度。

之后查看指导书要求实现的第二个函数——`sys_ipc_can_send()`。这里参看了指导书和代码中的注释后，
可以看到代码中的注释还是比较清晰的。我们参照代码中的注释实现这个函数。

```c
// Try to send 'value' to the target env 'envid'.
// 尝试向目标进程(env) 'envid'发送'value'
// 
// The send fails with a return value of -E_IPC_NOT_RECV if the
// target has not requested IPC with sys_ipc_recv.
// 如果目标进程没有使用sys_ipc_recv请求IPC，那么发送会失败，
// 并且返回-E_IPC_NOT_RECV。
//
// Otherwise, the send succeeds, and the target's ipc fields are
// updated as follows:
//    env_ipc_recving is set to 0 to block future sends
//    env_ipc_from is set to the sending envid
//    env_ipc_value is set to the 'value' parameter
// The target environment is marked runnable again.
// 否则，发送成功，目标进程的ipc相关的域被按照如下方式设置：
//    env_ipc_recving 被设置为 0 以阻塞之后发送的值
//    env_ipc_from 被设置为发送者的 envid
//    env_ipc_value 被设置为'value'
// The target environment is marked runnable again.
// 目标进程被重新设置为可运行的
//
// Return 0 on success, < 0 on error.
// 成功时返回0，错误时返回值小于0
//
// Hint: the only function you need to call is envid2env.
// 注意：你需要调用的唯一的函数是envid2env。
```

首先是需要查一些`envid2env`的用法，这个函数定义在`env.c`中。

```c
//
// Converts an envid to an env pointer.
//
// RETURNS
//   env pointer -- on success and sets *error = 0
//   NULL -- on failure, and sets *error = the error number
//
int envid2env(u_int envid, struct Env **penv, int checkperm)
{
  
    struct Env *e;
    u_int cur_envid;
    if (envid == 0) {
        *penv = curenv;
        return 0;
    }

    e = &envs[ENVX(envid)];
    if (e->env_status == ENV_FREE || e->env_id != envid) {
        *penv = 0;
        return -E_BAD_ENV;
    }

    if (checkperm) {
        // TODO: 这一段检查很是怪异，有些细节还尚未分析清楚，哪位大神来补充一下？
        cur_envid = envid;
        while (&envs[ENVX(cur_envid)] != curenv && ENVX(cur_envid) != 0)
        {
            envid = envs[ENVX(cur_envid)].env_parent_id;
            cur_envid = envid;
        }
        if (ENVX(cur_envid) == 0)
        {
            *penv = 0;
            return -E_BAD_ENV;
        }
    }
    *penv = e;
    return 0;
    
}
```

首先我们可以看到。。。函数的注释是错的。。。移植到mips的时候忘记改了吧。。。
大致扫一眼函数我们可以知道，这个函数的用法应该是传入一个`envid`，
然后函数会把`envid`所对应的`env`结构体的指针存在你传入的`penv`中。
如果出错会返回`-E_BAD_ENV`。分析到这里，这个函数的用法应该就相对清晰了。
传入一个`envid`，一个env结构体指针变量的地址，`checkperm`的用途不明，貌似是在检查权限。
传1的话会出现些问题，所以这里传了0，直接不检查（虽然这样的话安全性堪忧，但也无所谓）。

接下来按照注释填写`sys_ipc_can_send()`，但按照注释填写完我们会发现一个问题，
有一些变量我们完全没有使用到。例如srcva、perm还有代码中定义的一些局部变量。
为了探究它们的作用，我们需要去看一下，用户进程是怎样使用这个系统调用的。
经过一番搜索，我们找到了`user/ipc.c`这个文件，这个文件是用户程序使用ipc系统调用的接口。
你可以理解这个东西类似于C库的位置（当然，这个类比不大准确，只是为了便于理解这么说的），
里面定义了两个函数，其代码能够解释很多东西。

```c
// Send val to whom.  This function keeps trying until
// it succeeds.  It should panic() on any error other than
// -E_IPC_NOT_RECV.  
//
// Hint: use syscall_yield() to be CPU-friendly.
void
ipc_send(u_int whom, u_int val, u_int srcva, u_int perm)
{
    int r;

    while ((r=syscall_ipc_can_send(whom, val, srcva, perm)) == -E_IPC_NOT_RECV)
    {
        syscall_yield();
        //writef("QQ");
    }
    if(r == 0)
        return;
    user_panic("error in ipc_send: %d", r);
}

// Receive a value.  Return the value and store the caller's envid
// in *whom.  
//
// Hint: use env to discover the value and who sent it.
u_int
ipc_recv(u_int *whom, u_int dstva, u_int *perm)
{
//printf("ipc_recv:come 0\n");
    //syscall_ipc_recv(dstva);
    // 上面的syscall被注释掉我也是醉了。。。
    env->env_ipc_recving = 1;
    while( env->env_ipc_recving == 1 )writef("");
    
    if (whom)
        *whom = env->env_ipc_from;
    if (perm)
        *perm = env->env_ipc_perm;
    return env->env_ipc_value;
}
```

这两个函数中的第二个函数为我们解释了`perm`的作用。在实现`sys_ipc_can_send()`的时候，
我们还需要设置env_ipc_perm这个域。另外，这个函数其实会让我们发现一个很逗的事实，
sys_ipc_recv好像根本没有被调用。用户进程都是在调用`ipc_recv`来实现进程间通讯。
而`ipc_recv`这个函数中`syscall_ipc_recv`那一行调用竟然被注释掉了。
之后竟然自己把自己的env_ipc_recving设置为1，通过不断轮询来实现ipc，这是有趣。。。
移植的时候忘改了？还是调试忘恢复了？好吧。。就不继续吐槽这里了。

笔者把这里改成了

```c
syscall_ipc_recv(dstva);
// env->env_ipc_recving = 1;
// while( env->env_ipc_recving == 1 )writef("");
```

但从功能上看应该区别不大，请大家自行斟酌吧。。。不过从`ipc_recv`的实现我们可以看出，
`dstva`什么用都没有，所以同理`srcva`也不会起到任何作用。
直接忽视之。`sys_ipc_can_send`到这里我们就算是实现完了吧。

开始看下一个需要实现的函数`sys_mem_alloc()`，这个函数的作用在指导书中有一段长长的解释，
摘录如下：

> 一般的用户进程运行时，会有自己的用户栈（以 USTACKTOP 为栈顶），当发生页中断后，
我们让内核重新启动该进程并使其在另外一个栈（以 UXSTACKTOOP 为栈顶）用户
异常栈上运行用户定义的中断处理程序，该中断处理程序通过系统调用来处理
页中断，当然运行该中断处理程序之前，首先需要这个系统调用函数来给用户
异常栈分配空间。在分配空间时，涉及到 page_alloc 和 page_insert 函数，函
数具体用法可参考 mm/pmap.c。 

在结合一下注释：

```c
//
// Allocate a page of memory and map it at 'va' with permission
// 'perm' in the address space of 'envid'.
// 分配一个页面，并将它映射到'envid'的地址空间'va'，权限设置为'perm'
//
// If a page is already mapped at 'va', that page is unmapped as a
// side-effect.
// 如果一页已经被映射到了'va'，那么作为一个副作用，这页会被取消映射。
//
// perm -- PTE_U|PTE_P are required, 
//         PTE_AVAIL|PTE_W are optional,
//         but no other bits are allowed (return -E_INVAL)
// 这段注释的权限位完全是错的。。。根本对不上啊。。。
//
// Return 0 on success, < 0 on error
//  - va must be < UTOP
//  - env may modify its own address space or the address space of its children
// 当成功是返回0,失败是返回一个<0的数
//  - va必须小于UTOP
//  - env可能会修改它自己的地址空间或者它的子进程的地址空间。
```

调用`page_alloc`分配物理页框，然后调用`page_insert`建立映射相信进过前两次的实验大家已经很熟悉了，
之类就不再赘述了。这里有三处提醒一下，一个是需要建立的是进程的地址空间到物理页框的映射，
所以`page_insert`需要传入的是进程页表，也就是`e->env_pgdir`。e通过`envid2env()`函数获得（envid作为参数传入）。
另外一个是务必要判断`page_alloc`和`page_insert`的返回值，因为这两个函数可能会发生错误，
如果发生错误，则`sys_mem_alloc`也需要直接返回错误（返回值即为`page_alloc`或者`page_insert`的错误值）。
最后一个，需要判断下va是否小于UTOP，如果不成立则返回-E_INVAL，表示出错。

写到这里我们还会发现一个问题，按照函数注释我们还需要检查`perm`是否合法。但注释完全就不对啊。。。
所以我们结合指导书做一个猜测。首先，由于这个函数是在异常处理时调用的，
用于给用户异常栈分配空间，因此，perm必须是有效且可写的。所以原注释中`PTE_U|PTE_P are required`，
指的应该是`PTE_V|PTE_R are required`。所以，我们判断一下`perm`是否设置了这些位（`(perm&PTE_V) && (perm&PTE_R)`），
如果没设置就返回个-E_INVAL。英文注释中其他关于`perm`的要求是在是找不到对应的东西，
就暂且不管它了，应该是实验移植到MIPS上的时候遗留的，估计问题不大。

其余部分按照注释和指导书的提示完成即可，这个函数到这里告一段落。

我们进入下一个函数`sys_set_pgfault_handler`，直接看注释。

```c
// Set envid's pagefault handler entry point and exception stack.
// (xstacktop points one byte past exception stack).
// 设置envid的页中断处理程序进入点和异常栈。
//
// Returns 0 on success, < 0 on error.
// 成功返回0，失败返回一个小于0的值
```

根据指导书的提示，看env结构体的定义。

```c
struct Env {
    // ...
    
    // Lab 4 fault handling
    u_int env_pgfault_handler;      // page fault state
    u_int env_xstacktop;            // top of exception stack
    
    // ...
}
```

看样子直接赋值就好了，`sys_set_pgfault_handler`的参数有`envid`、`func`和`xstacktop`，
第一个用于获取env结构体，后面两个直接和结构体中的域对应。这个函数应该这样就OK了。
至此任务一结束。当然，在查各方面的资料和代码时还看到了一个奇怪的用法，对于传入的参数`xstacktop`，
还需要做这样一个操作：

```c
xstacktop = TRUP(xstacktop);
```

其中，TRUP的定义如下：

```c
#define TRUP(_p)                        \
({                              \
    register typeof((_p)) __m_p = (_p);         \
    (u_int) __m_p > ULIM ? (typeof(_p)) ULIM : __m_p;   \
})
```

这个宏实际上就是在判断地址是否大于`ULIM`，如果大于则返回`ULIM`。通过`include/mmu.h`中的图可以看到，
`ULIM`以上是用于Interrupts & Exception，也就是说，`ULIM`是用户空间最高的地址了，
如果传入了更高的地址，说明地址有错，为了安全，直接设置为`ULIM`。
所以，这一步操作应该是为了系统安全考虑的。避免用户设置一个过高的`xstacktop`位置，影响到系统的运行。

开始任务二，第一个函数是`sys_env_alloc`，为子进程分配空执行环境。这里也是直接读函数的注释即可。

```c
// Allocate a new environment.
//
// The new child is left as env_alloc created it, except that
// status is set to ENV_NOT_RUNNABLE and the register set is copied
// from the current environment.  In the child, the register set is
// tweaked so sys_env_alloc returns 0.
// 新的子进程除了status需要设置为ENV_NOT_RUNNABLE以外，其他的都
// 保持env_alloc分配出来的时候的状态就行。子进程的寄存器组
// 需要设置为当前运行环境的拷贝。在子进程中，
// 寄存器需要被调整，以使得sys_env_alloc返回0.
//
// Returns envid of new environment, or < 0 on error.
```

首先一处坑点是，这里当前进程的运行环境被保存在了`(int)KERNEL_SP-sizeof(struct Trapframe)`。
这一点可以根据`sys_yield`这个函数中的语句推测出来。我们需要将运行环境拷贝给子进程：

```c
bcopy((int)KERNEL_SP-sizeof(struct Trapframe), &child->env_tf,sizeof(struct Trapframe));
```

值得注意的是在将父进程的运行环境拷贝到子进程后，还需要调整一下，使得子进程的调用返回0.
也就是说，子进程相当于父进程的一份复制，寄存器状态等等都一样，只不过，
一个返回子进程的id，一个返回0.（也就是fork的功能）。
从汇编的角度看，函数调用的返回值是储存在v0和v1中的，因此，我们只要改
子进程的v0寄存器就可以了。这样当子进程被调度的时候，其v0寄存器中就是0，
子进程就会认为调用的返回值为0.其余部分照着注释填就可以了。（提示：v0是2号寄存器，从0开始编号的话）

这里为了方便大家，再给出`struct Trapframe`的定义

```c
struct Trapframe { //lr:need to be modified(reference to linux pt_regs) TODO
    /* Saved main processor registers. */
    unsigned long regs[32];

    /* Saved special registers. */
    unsigned long cp0_status;
    unsigned long hi;
    unsigned long lo;
    unsigned long cp0_badvaddr;
    unsigned long cp0_cause;
    unsigned long cp0_epc;
    unsigned long pc;
};
```

最后，还有一处坑点，子进程接下来被调度的时候，需要从epc的位置开始运行，这一点和`env_run`相同。
所以，我们还需要：

```c
child->env_tf.pc = child->env_tf.cp0_epc;
```

下一个，实现`sys_mem_map`函数。根据指导书，这个函数用于将父进程空间的内容映射给子进程。
其英文注释解释得相对清晰：

```c
// Map the page of memory at 'srcva' in srcid's address space
// at 'dstva' in dstid's address space with permission 'perm'.
// Perm has the same restrictions as in sys_mem_alloc.
// (Probably we should add a restriction that you can't go from
// non-writable to writable?)
// 将srcid的地址空间中的srcva所对的页映射到dstid的地址空间中的dstva
// 的位置上，权限位设置为perm。
// 权限位的限制条件和sys_mem_alloc相同（呵呵了。。。sys_mem_alloc的相关注释完全没法看啊。。。）
//
// Return 0 on success, < 0 on error.
// 成功时返回0， 失败时返回小于0的值。
//
// Cannot access pages above UTOP.
// 不能访问高于UTOP的页面。
```

这里直接参照注释做就可以了，首先获取srcid和dstid所对应的env结构体，
然后获得srcenv地址空间中srcva所对的页面。
最后调用`page_insert`将这个页面插入到dstenv地址空间中的dstva处，同时设置perm就OK了。

接下来的问题是如何获取`srcva`所对的物理页面。通过翻阅`pmap.c`我们可以找到一个函数`page_lookup`。
这个函数可以返回va所映射到的物理页面。但注释并没有解释函数用法，因此我们来详细解析一下。

```c
//return the Page which va map to.
struct Page*
page_lookup(Pde *pgdir, u_long va, Pte **ppte)
{
    struct Page *ppage;
    Pte *pte;

    // 获取va所对应的页表项。这里create参数为0，
    // 说明即使页表项不存在也不会建立相应页表项。
    pgdir_walk(pgdir, va, 0, &pte);
    // 如果页表项尚未建立，返回0。
    if(pte==0) return 0;
    // 检查页表项所对应页面的有效性
    if((*pte & PTE_V)==0) return 0;       //页面不在内存中返回0
    // 调用pa2page获取页面地址
    ppage = pa2page(*pte);
    // 顺便将va所对页表项通过参数返回
    if(ppte) *ppte = pte;
    return ppage;
}
```

那么分析到这里，调用这个函数获取page应该不是什么大问题了。这里也是做一点提醒，
记得判断函数返回值，如代码中可以看到的那样，如果不存在对应的page，这个函数会返回0.
因而，调用的时候务必做好判断。

好了，我们开始实现`syscall_all.c`中最后一个需要我们实现的函数`sys_set_env_status`。
还是直接读英文注释：

```c
// Set envid's env_status to status. 
// 设置envid的env_status为status
//
// Returns 0 on success, < 0 on error.
// 成功则返回0，失败返回小于0的值。
// 
// Return -E_INVAL if status is not a valid status for an environment.
// 如果status不是一个合法的状态，那么返回-E_INVAL
```

这个函数很简单，唯一需要注意就是要判断`envid2env`的返回值以及`status`是否合法。
`status`只有三个合法的值`ENV_FREE`、`ENV_RUNNABLE`和`ENV_NOT_RUNNABLE`。判断一下即可。

OK，终于进入到fork.c了。这个文件里的函数是给用户程序调用的。不在是内核的部分了。
关于系统调用官方的指导书上给了一张图，大致就是从用户空间通过`syscall`指令进入内核空间再返回。
所以呢，接下来请一定要注意，我们之后实现的函数都是用户空间的函数。
也就是说，我们只能使用用户空间的资源以及系统调用。不能使用之前我们所一直在使用的那些函数了。
那些都是内核空间的东西，用户是无权使用的。这一点，在之后的实现中请务必注意。
`fork`通过调用系统调用实现fork的功能。我们按照指导书给出的顺序逐条实现。

> 设置缺页中断函数为 pgfault。
  生成新的进程。
  若 envid 为 0，说明返回的进程为子进程；否则，需要为子进程拷贝地址空间。
  遍历所有页表目录，将父进程的页映射到子进程的对应位置上。
  为子进程的 exception stack 单独分配一个新的内存页。
  设置子进程的缺页中断函数入口和 exception stack 的地址。
  设置子进程的状态为可以运行，将其加入到进程调度队列。
  返回新进程 ID。

目前我们处于用户空间，fork.c是用户级的实现，所以我们需要使用系统调用等用户进程可以
使用的功能来完成我们的所有操作。所有用户级系统调用实现在`lib.h`中。
为了方便，这里直接粘出lib.h声明部分的代码

```c
u_int syscall_getenvid(void);
void syscall_yield(void);
void syscall_env_destroy(u_int envid);
int syscall_set_pgfault_handler(u_int envid, u_int func, u_int xstacktop);
int syscall_mem_alloc(u_int envid, u_int va, u_int perm);
int syscall_mem_map(u_int srcid, u_int srcva, u_int dstid, u_int dstva, u_int perm);
int syscall_mem_unmap(u_int envid, u_int va);
int syscall_env_alloc(void);
int syscall_set_env_status(u_int envid, u_int status);
int syscall_set_trapframe(u_int envid, struct Trapframe *tf);
void syscall_panic(char *msg);
```
第一步，我们需要设置缺页中断函数为`pgfault`。`pgfault`函数用于处理页面被设置为`copy on write`的情况。
`copy on write`在课上讲过，这里再简单说一下。就是把父进程和子进程的所有页面都设置为不可写。
这样如果发生了写操作，CPU会发生中断。发生中断后，操作系统检测是否为`copy on write`的页面，
如果是的话将原页面复制一份，然后重新映射到导致中断的虚地址上，之后重新执行指令。
就可以正常进行写入操作了。而这个`pgfault`正是用于处理`copy on write`的函数。
所以，我们首先需要把父进程的处理函数设置为`pgfault`。这样创建子进程并重新映射页面后，
就可以处理与父进程相关的`copy on write`了。

设置缺页中断处理函数是个大坑，因为事实上有个用户空间的函数可以实现这个功能。
这个函数定义在`user/pgfault.c`中：

```c
//
// Set the page fault handler function.
// If there isn't one yet, _pgfault_handler will be 0.
// The first time we register a handler, we need to 
// allocate an exception stack and tell the kernel to
// call _asm_pgfault_handler on it.
// 设定页中断处理函数。如果目前还没有，那么_pgfault_handler会被置0.
// 我们第一次注册一个处理函数时，需要分配一个用于异常处理的栈，
// 并告知内核调用_asm_pgfault_handler。
//
void
set_pgfault_handler(void (*fn)(u_int va, u_int err))
{
    int r;  

    if (__pgfault_handler == 0) {
        // Your code here:
        // map one page of exception stack with top at UXSTACKTOP
        // register assembly handler and stack with operating system
        // 映射一页到UXSTACKTOP，为异常处理分配栈空间。
        // 注册一个汇编语言的处理函数__asm_pgfault_handler，并告知操作系统其需要运行在哪个栈上。
        if(syscall_mem_alloc(0, UXSTACKTOP - BY2PG, PTE_V|PTE_R)<0 || syscall_set_pgfault_handler(0, __asm_pgfault_handler, UXSTACKTOP)<0)
        {
            writef("cannot set pgfault handler\n");
            return;
        }
//      panic("set_pgfault_handler not implemented");
    }

    // Save handler pointer for assembly to call.
    __pgfault_handler = fn;
}
```

所以，我们可以看到，实际上我们向系统注册的处理函数是`__asm_pgfault_handler`，由它再去调用我们的`pgfault`函数。
我们继续深入进去，找到`entry.S`，`__asm_pgfault_handler`就定义在这里。

```
.globl __asm_pgfault_handler
__asm_pgfault_handler:
    // save the caller-save registers
    //  (your code here)
//1: j 1b
nop
    // 将发生错误的地址赋值给a0。
    // 这里是为了后面调用__pgfault_handler做准备
    lw  a0, TF_BADVADDR(sp)
    //sw    t0, (sp)
    //subu  sp,16
    // 以a0里的地址作为参数va调用__pgfault_handler
    lw  t1, __pgfault_handler
    jalr    t1

nop
    // 后面略去
```

`__pgfault_handler`的定义也在这个文件中

```
.globl __pgfault_handler
__pgfault_handler:
.word 0
```

所以，`__pgfault_handler`是一个全局的指针。再结合`pgfault.c`中的定义

```c
extern void (*__pgfault_handler)(u_int, u_int);
extern void __asm_pgfault_handler(void);
```

我们就可以完全地确定下来，`__pgfault_handler`就是进程的页中断处理函数的指针。
好吧，现在我们回到正题，我们需要调用下`set_pgfault_handler`，并将`pgfault`作为参数传入，即可完成这一步。

调用系统调用`syscall_env_alloc()`创建子进程，然后根据返回的envid，分别做出处理。
对于子进程，envid = 0，直接返回即可。对于父进程，需要继续进行下面的步骤。

遍历所有页表目录，将父进程的页映射到子进程的对应位置上。这一步让笔者思考了很久。我们位处用户空间，
只能使用系统调用以及用户地址空间里的东西，如何才能遍历页表呢？最后发现了注释中的提示

```c
// Hint: use vpd, vpt, and duppage.
// 提示：使用vpd，vpt和duppage。
```

接着，在`mmu.h`中，我们发现了vpd和vpt的定义

```c
extern volatile Pte* vpt[];
extern volatile Pde* vpd[];
```

这里加了个很有趣的修饰`volatile`，说明这两个变量可能在汇编等C语言之外的地方被改变。
避免编译优化造成的错误，我们接下来寻找对应的汇编代码。打开`entry.S`，我们终于发现了这两个货的定义。

```
    .globl vpt
vpt:
    .word UVPT

    .globl vpd
vpd:
    .word (UVPT+(UVPT>>12)*4)
```

根据`mmu.h`，UVPT是用户进程地址空间中页表的位置。很显然，
vpd就是页目录的位置（页表的自映射及地址计算老师在上课已经讲过，这里不再赘述）。
所以，遍历页目录也就是遍历vpd。遍历vpd时，我们需要将所有分配给内存的物理页
（含二级页表所占的页以及其他以及分配给进程的页），映射给子进程。
遍历时需要判断二级页表是否存在，存在则映射。同时，二级页表中分配的页也需要映射。
二级页表的访问通过vpt来进行。这里还要注意，应该不需要遍历2G以上地址空间的页目录。
那里在`env_setup_vm`时已经映射好了，无需再动，只管用户空间就行。
映射使用`duppage`函数。pn为虚页号。

这里考虑再三，笔者决定直接给出实现遍历所有页并映射给子进程的功能的代码，
请务必理解后重新实现一下：

```c
for (i = 0; i < (UTOP/BY2PG); ++i) {
    if (((*vpd)[i/PTE2PT]) != 0 && ((*vpt)[i]) != 0) {
        duppage(envid,i);
        }
    }
}
```

这里有一点要特别注意vpt本身是一个地址，这个地址中储存的内容是`UVPT`，
所以使用的时候需要用`(*vpt)[offset]`的形式。

为子进程的 exception stack 单独分配一个新的内存页，可以使用我们之前实现的`sys_mem_alloc`来实现。
用户空间对应的系统调用为`syscall_mem_alloc`函数。我们为`UXSTACKTOP-BY2PG`这个地址开始映射一页，
作为异常处理所使用的内存。具体可以参考之前对于`sys_mem_alloc`的讨论。

设置子进程的页中断函数入口和 exception stack 的地址，可以使用系统调用`syscall_set_pgfault_handler`来完成，
其用法和我们之前实现的时候讨论的一样。不再赘述一遍了。很容易实现。

但这里也有一个 **大坑** ，我们需要将`__asm_pgfault_handler`设置为子进程的页中断函数入口。
通过上面的一长串解析，我们可以知道，`__asm_pgfault_handler`函数（汇编的那段），
将出错的地址提取了出来，并作为参数传递给`pgfault`。也就是说，如果我们将子进程的页中断函数设置为`pgfault`，
那么页中断时，就会直接进入`pgfault`函数。这样的结果就是：从cp0中提取出错的虚地址，
恢复寄存器等工作就没代码来做了。所以，我们需要将`__asm_pgfault_handler`设置为子进程的页中断处理函数。

设置子进程状态也是调用系统调用，最后返回子进程ID就OK了。这里比较简单，就不多说了。

我们开始实现`duppage`函数，这里的实现需要参看指导书的说明

> 若该内存页是可写(PTE_R)或者是 copy-on-write(PTE_COW)，则需将
页从父进程以 copy-on-write 的方式映射到子进程，并且将父进程的页也映射为
copy-on-write，否则的话则将父进程的页以原标记映射到子进程。

如何访问pn所对页表项呢？我们可以使用`vpt`，`(*vpt)[pn]`即是对应的页表项。
调用`syscall_mem_map`可以完成映射的过程，同时， 需要将父进程的页也映射为“copy-on-write”。
这是因为，如果不将父进程映射为`copy-on-write`，父进程可能修改一些地址上的内容，
而这些页子进程也在用，导致了子进程所获得的值是错误的。所以，为了保证`copy-on-write`的正确性，
我们需要将父进程的页也映射为“copy-on-write”。避免父进程对于页面的不可控的修改。

这里有一点要注意，`envid2env`这个函数，如果想获得当前进程的envid，只需要传入0即可。
所以，如果在调用`syscall_mem_map`一类的函数时，需要传自己的`envid`，则只需要简单地传入0即可。

下面我们进入最后一个函数，`pgfault()`。这里直接参见指导书的说明做就好（以下摘自指导书）。

> 首先判断该页是否为 copy-on-write，若不是则函数报错，
然后分配一个新的内存页到一个临时位置，接着将内容拷贝到刚刚分配的临时位置
上，最后将临时位置上的内容映射到指定地址 va。 

说句实话。。。这个函数笔者已经写得抓狂了。。。

第一个问题是`pgfault`到底是运行在用户空间还是内核空间，
这里翻看了一下指导书关于页中断处理的描述，可以发现`pgfault`是运行在用户空间的，
也就是说，我们依旧不能使用内核内的函数，只能利用系统调用来和系统通讯。
这一点在`traps.c`中的`page_fault_handler`的实现中可以得到印证。

```c
void
page_fault_handler(struct Trapframe *tf)
{
        u_int va;
        u_int *tos, d;
    struct Trapframe PgTrapFrame;
    extern struct Env * curenv;
//printf("^^^^cp0_BadVAddress:%x\n",tf->cp0_badvaddr);

    
    bcopy(tf, &PgTrapFrame,sizeof(struct Trapframe));
    if(tf->regs[29] >= (curenv->env_xstacktop - BY2PG) && tf->regs[29] <= (curenv->env_xstacktop - 1))
    {
        //panic("fork can't nest!!");
        tf->regs[29] = tf->regs[29] - sizeof(struct  Trapframe);
        bcopy(&PgTrapFrame, tf->regs[29], sizeof(struct Trapframe));
    }
    else
    {
        
        tf->regs[29] = curenv->env_xstacktop - sizeof(struct  Trapframe);
//      printf("page_fault_handler(): bcopy(): src:%x\tdes:%x\n",(int)&PgTrapFrame,(int)(curenv->env_xstacktop - sizeof(struct  Trapframe)));       
        bcopy(&PgTrapFrame, curenv->env_xstacktop - sizeof(struct  Trapframe), sizeof(struct Trapframe));
    }
//  printf("^^^^cp0_epc:%x\tcurenv->env_pgfault_handler:%x\n",tf->cp0_epc,curenv->env_pgfault_handler);

    // 系统的页中断处理最后，把中断处理的返回地址设置为了env_pgfault_handler的地址。
    // 因此，中断处理返回时会返回到用户空间的env_pgfault_handler中，
    // 由此推测，pgfault函数仍然是运行在用户空间。
    tf->cp0_epc = curenv->env_pgfault_handler;
    
    return;
}
```

第二个问题，这个页中断处理只处理了写时拷贝的情况，如果遇到其他原因引发的页中断怎么办？
这里要参考下`genex.S`中的代码。

```
    .extern tlbra
.set    noreorder
NESTED(do_refill,0 , sp)
            //li    k1, '?'
            //sb    k1, 0x90000000
            .extern mCONTEXT
//this "1" is important
1:          //j 1b
            nop
            lw      k1,mCONTEXT
            and     k1,0xfffff000
                mfc0        k0,CP0_BADVADDR
                srl     k0,20
                and     k0,0xfffffffc
            addu        k0,k1
            
            lw      k1,0(k0)
            nop
                    move        t0,k1
                    and     t0,0x0200
                    beqz        t0,NOPAGE
            nop
            and     k1,0xfffff000
                mfc0        k0,CP0_BADVADDR
                srl     k0,10
                and     k0,0xfffffffc
                and     k0,0x00000fff
            addu        k0,k1

            or      k0,0x80000000
            lw      k1,0(k0)
            nop
                    move        t0,k1
                    and     t0,0x0200
                    beqz        t0,NOPAGE
            nop
            move        k0,k1
            and     k0,0x1
            beqz        k0,NoCOW
            nop
            and     k1,0xfffffbff
// 这里做了个分发，根据不同的引发中断的原因分发给不同的处理函数。
NoCOW:
            mtc0        k1,CP0_ENTRYLO0
            nop
            tlbwr

            j       2f
            nop
// 这里笔者也没看太懂，感觉就是NO PAGE的情况会调用pageout函数处理。
NOPAGE:
//3: j 3b
nop
            mfc0        a0,CP0_BADVADDR
            lw      a1,mCONTEXT
            nop
                
            sw      ra,tlbra
            jal     pageout
            nop
//3: j 3b
nop
            lw      ra,tlbra
            nop

            j   1b
2:          nop

            jr      ra
            nop
END(do_refill)
```

虽然没完全读懂，但感觉应该是只需要考虑指导书上所说的只处理写时拷贝的情况就行。

之后开始实现这个函数，我们又面临了一个新问题，在用户空间如何获得va所对应的页表项。
一开始想用`pgdir_walk`之类的，但那些是内核空间的函数啊。。。所以，没办法，
我们只能通过vpd和vpt这两个家伙来自行找到页表项的位置。（时刻注意我们在用户空间）。

这里我们通过`(*vpt)[VPN(va)]`获得对应的页表项。
由于`pgfault`是专门处理页框有效但不可写的情况的（也就是`copy-on-write`），
所以，`va`所对的页表项一定存在。这也是为什么我们可以直接通过`(*vpt)[VPN(va)]`
获得对应的页表项，而不需要判断其一级页表是否存在的原因。
（注意，我们这里全部是通过用户空间的虚地址访问的）。获得对应页表项以后，
判断该页是否是 copy-on-write，若不是则函数报错。至于如何报错，笔者也不知，直接return或者user_panic吧？

下一步就比较简单了，分配一个也映射到一个临时的位置。`fork.c`中定义了一个宏

```c
//fmars define a free mem space for pgfault
#define PFTEMP (ROUNDDOWN(0x50000000, BY2PG))//( USTACKTOP - 2 * BY2PG )
```

显然这个`PFTEMP`正是所谓的临时位置。我们通过`syscall_mem_alloc`来分配一个新的内存页并映射到这个位置上。
然后调用`user_bcopy`将`va`处的内容拷贝到`PFTEMP`，最后将临时位置上的内容通过`syscall_mem_map`映射到指定地址。
按照指导书要求，我们还需要取消页框到临时位置的映射，于是，我们发现我们需要实现`sys_mem_unmap`这个函数。

```c
// Unmap the page of memory at 'va' in the address space of 'envid'
// (if no page is mapped, the function silently succeeds)
// 取消'envid'地址空间中'va'到物理页的映射。（如果并没有映射，这个函数会直接成功返回）
//
// Return 0 on success, < 0 on error.
// 成功返回0，错误返回小于0.
//
// Cannot unmap pages above UTOP.
// 不能将高于UTOP的页unmap掉。
int sys_mem_unmap(int sysno,u_int envid, u_int va)
```

查看`mm/pmap.c`，我们发现了一个神奇的`page_remove`函数，其原型如下：

```c
//
// Unmaps the physical page at virtual address 'va'
// 
void
page_remove(Pde *pgdir, u_long va)
```

一眼看上去就知道怎么用了，我就不再赘述了。总之实现了`sys_mem_unmap`后，回到`user/fork.c`，
调用`syscall_mem_unmap`取消当前进程`PFTEMP`的映射即可。

终于搞定了。。。再接再厉，我们完成最后的部分。我们来实现`user/syscall_wrap.s`，实现系统调用。
这里按照提示，首先将a0赋值给v0，然后将a0-3按照次序压入栈中。最后调用syscall指令。最后jr ra就OK了。
这样描述比较抽象，我们还是来具体地分析一下`user/syscall_wrap.s`如何实现吧。

我们先看调用它的函数的代码，这里需要了解底层的处理，所以我们make过后，通过objdump反汇编syscall_lib.o。

```
$ mips_4KC-objdump -d user/syscall_lib.o
```

然后，我们就得到用户空间的syscall一类的函数的汇编码。我们选个简单的来分析一下。

```c
u_int
syscall_getenvid(void)
{
        return msyscall(SYS_getenvid,0,0,0,0,0);
}
```

其对应的汇编代码为

```
00000034 <syscall_getenvid>:
  34:   27bdffe0        addiu   sp,sp,-32
  // 保存ra寄存器
  38:   afbf0018        sw      ra,24(sp)
  // 将第五和第六个参数压入栈（虽然它们都是0吧）
  3c:   afa00010        sw      zero,16(sp)
  40:   afa00014        sw      zero,20(sp)
  // 第一个参数（SYS_getenvid）存入a0
  44:   24042538        li      a0,9528
  // 第二、三个参数存入a1、a2
  48:   00002821        move    a1,zero
  4c:   00003021        move    a2,zero
  // 第四个参数存入a3（这条指令在延迟槽里^_^），然后跳转到'msyscall'。
  // 由于我反汇编的是.o文件，尚未链接，所以这里的地址是0.
  // 等链接器链接后，这里才会被替换成'msyscall'的地址。
  50:   0c000000        jal     0 <syscall_putchar>
  54:   00003821        move    a3,zero
  // 恢复ra寄存器
  58:   8fbf0018        lw      ra,24(sp)
  // 恢复sp寄存器，返回。
  5c:   03e00008        jr      ra
  60:   27bd0020        addiu   sp,sp,32
```

所以，我们可以看到，前4个参数存在a0~a3中，其中a0为系统调用号。
后两个参数存在`16(sp)`和`20(sp)`的位置。我们再来看看，进入到内核态后，
系统调用是如何被分发的。这一段在`lib/syscall.S`文件里。

```
NESTED(handle_sys,TF_SIZE, sp)

SAVE_ALL
CLI

//1: j 1b
nop
.set at
lw t1, TF_EPC(sp)
lw v0, TF_REG2(sp)
// 下面这段大概是在通过系统调用号来查找对应的系统调用的入口地址
// v0里是系统调用号。
subu v0, v0, __SYSCALL_BASE
sltiu t0, v0, __NR_SYSCALLS+1

addiu t1, 4
sw      t1, TF_EPC(sp)
beqz    t0,  illegal_syscall//undef
nop
sll     t0, v0,2
la      t1, sys_call_table
addu    t1, t0
lw      t2, (t1)
beqz    t2, illegal_syscall//undef
nop
// 这里提取了之前用户空间的栈指针的位置。
lw      t0,TF_REG29(sp)

// 这里是核心，这里是在获得参数。
lw      t1, (t0)
lw      t3, 4(t0)
lw      t4, 8(t0)
lw      t5, 12(t0)
lw      t6, 16(t0)
lw      t7, 20(t0)

subu    sp, 20

// 把参数存在当前的栈上。
sw      t1, 0(sp)
sw      t3, 4(sp)
sw      t4, 8(sp)
sw      t5, 12(sp)
sw      t6, 16(sp)
sw      t7, 20(sp)

// 把前四个参数移入a0~a3。
// 前四个参数在a0~a3，后面的参数在栈上，这应该是MIPS的ABI标准吧。
move    a0, t1
move    a1, t3
move    a2, t4
move    a3, t5

// 跳转到系统调用函数入口地址
jalr    t2
nop

// 恢复栈指针
addu    sp, 20

// 将返回值保存在进程的运行环境的v0寄存器中。
// 这样返回用户态时，用户就可以获得返回值了。
sw      v0, TF_REG2(sp)

j       ret_from_exception//extern?
nop

illegal_syscall: j illegal_syscall
                        nop
```

所以，我们通过上面的分析可以看出来，我们在`syscall_wrap.S`中把a0到a3的值保存在0(sp)到12(sp)的地方。
然后调用`syscall`指令触发系统调用，进入内核态。最后，返回就可以了。思索良久，还是决定给出这一部分。

```
    move    v0,a0
    sw      a0,0(sp)
    sw      a1,4(sp)
    sw      a2,8(sp)
    sw      a3,12(sp)

    syscall
    
    jr      ra
    nop
    // 这里注意，一定要有个nop，因为jr指令有延迟槽。
```

接下来，我们要弄个进程玩玩了。删去上次的ENV_CREATE。我们建立几个新的进程。
一共有三个可以跑的进程，`idle`,`fktest`和`pingpong`。
首先，我们先跑最简单的`idle`进程。它的代码在`user/idle.c`中，功能是无限输出`2!`。
所以我们通过`ENV_CREATE(user_idle);`来建立这个进程。

这时，我们又想到一个问题，初始的pc地址应该怎么设定？指导书的说法是设定为UTEXT+1000。
为了以防万一，笔者还是自己利用readelf工具查看了一下（下面展示的是pingpong的结果），
结果发现了个惊人的事实：

```
$ mips_4KC-readelf -h user/pingpong.b
ELF Header:
  Magic:   7f 45 4c 46 01 02 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF32
  Data:                              2's complement, big endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           MIPS R3000
  Version:                           0x1
  ------------ 注意这里 -------------------
  Entry point address:               0x400000
  -----------------------------------------
  Start of program headers:          52 (bytes into file)
  Start of section headers:          37444 (bytes into file)
  Flags:                             0x50001001, noreorder, o32, mips32
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         2
  Size of section headers:           40 (bytes)
  Number of section headers:         11
  Section header string table index: 8
```

入口地址又是错的！？！笔者又使用16进制编辑器查看了一下这个文件。发现代码还真是从0x1000的位置开始的。
前面的部分被elf文件头给占了。但是，可执行文件中的地址都是基于入口地址为`0x400000`计算的，
特别是跳转指令，跳向的绝对地址，全都是基于入口地址为`0x400000`算出来的。所以如果直接把二进制拷到`UTEXT`，
再从`UTEXT+0x1000`开始运行，则根本不可能跳转到正确的位置。

解决方法是跳过elf文件头，也就是从binary+0x1000的位置开始拷贝，这样binary+0x1000的内容就会到UTEXT了。
也就是，第一条指令就被放到了UTEXT。这之后，我们只需将pc设置为UTEXT即可。

调试的过程中还发现一个问题，错误现象是输出：

pageout:        @@@___0x7fdff000___@@@  ins a page

这个说明页目录的自映射做的有问题，需要改`env_setup_vm`这个函数：

```c
e->env_cr3 = page2pa(p) | PTE_V; // 这里需要或一个PTE_V
```

最后还有一处要改，在`lib/env.c`中：

```c
int envid2env(u_int envid, struct Env **penv, int checkperm)
{
        struct Env *e;
        u_int cur_envid;
        if (envid == 0) {
                *penv = curenv;
                return 0;
        }

        e = &envs[ENVX(envid)];
        if (e->env_status == ENV_FREE || e->env_id != envid) {
                *penv = 0;
                return -E_BAD_ENV;
        }

        if (checkperm) {
                cur_envid = envid;
                while (&envs[ENVX(cur_envid)] != curenv && ENVX(cur_envid) != 0)
                {
                        envid = envs[ENVX(cur_envid)].env_parent_id;
                        cur_envid = envid;
                }
        // 这里需要做一下修改，感觉应该是原来的代码有错。
        // 判断的时候判断条件有点小问题。
                if (&envs[ENVX(cur_envid)] != curenv && ENVX(cur_envid) == 0)
                {
                        *penv = 0;
                        return -E_BAD_ENV;
                }
        }
        *penv = e;
        return 0;

}
```

跑过之后，我们再修改`init/init.c`文件，开始跑`fktext`。这个用于测试`fork`的实现是否正确。
结果大致如下

```
main.c: main is start ...
init.c: mips_init() is called
Physical memory: 65536K available, base = 65536K, extended = 0K
to memory 80401000 for struct page directory.
to memory 80431000 for struct Pages.
mips_vm_init:boot_pgdir is 80400000
pmap.c:  mips vm init success
panic at init.c:28: ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
pageout:        @@@___0x7f3fe000___@@@  ins a page 
this is father: a:1
this is father: a:1
this is father: a:1
this is father: a:1
this is father: a:1
```

等片刻后出现

```
this is father: a:1
        this is child :a:2
        this is child :a:2
        this is child :a:2
                this is child2 :a:3
                this is child2 :a:3
```

由于多进程轮转的缘故，输出可能不是这么整齐，有可能输出一半另一个进程开始输出了，都很正常。
只要是这三句话就行。大概需要等5~10秒才会出现子进程输出的那两句，所以一开始没有不要着急中断程序。

最后我们要弄的进程的名字是`user_pingpong`。所以再把原来的`ENV_CREATE`换为`user_pingpong`，
这个是测试fork和ipc的实现是否正确的。正确输出大致如下

```
main.c: main is start ...
init.c: mips_init() is called
Physical memory: 65536K available, base = 65536K, extended = 0K
to memory 80401000 for struct page directory.
to memory 80431000 for struct Pages.
mips_vm_init:boot_pgdir is 80400000
pmap.c:  mips vm init success
panic at init.c:28: ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
pageout:        @@@___0x7f3fe000___@@@  ins a page 
pingpong is running
Fmars : parent env
Parent begins to send
1001 got 7777 from 800
@@@@@send 1 from 1001 to 800
[00001001] destroying 00001001
[00001001] free env 00001001
i am killed ... 
800 got 1 from 1001
[00000800] destroying 00000800
[00000800] free env 00000800
i am killed ... 
```

这里感谢罗天歌大神，如果你像笔者原来一样有如下错误输出

```
[1001    ] destroying 1001    
[1001    ] free env 1001    
i am killed ... 
800 got 1 from 1001
[800     ] destroying 800     
[800     ] free env 800  
```

那么说明lab1中对于%08x的支持有问题。需要回去做一些修正。

好了，就是这些了，祝实验愉快！

## 一些补充知识 ##

### 调试技巧 ###

在实现用户空间的函数的时候一定注意要善用`user_panic`。
当调用系统调用以后，一定要检查返回值是否正常，如果不正常，通过`user_panic`输出一条调试信息。
避免后面难以定位错误的位置。例如

```c
if (syscall_mem_map(0, PFTEMP, 0, va, PTE_V|PTE_R) < 0) {
    user_panic("syscall_mem_map error!");
}
```

有时候可以通过观察产生了什么中断来进行调试，添加-v选项可以让gxemul输出所有产生的中断。

```
$ gxemul -E testmips -C R3000 -M 64 gxemul/vmlinux -v
[ exception SYS v0=9527 a0=0x2537 a1=48 a2=0 a3=0 pc=0x004000e4 ]
[ exception SYS v0=9527 a0=0x2537 a1=49 a2=0 a3=0 pc=0x004000e4 ]
[ exception SYS v0=9527 a0=0x2537 a1=10 a2=0 a3=0 pc=0x004000e4 ]
```

### 在fork.c中如何访问当前进程的env结构体 ###

当前用户进程的env结构体，可以通过env这个变量访问。这个货定义在`user/lib.h`中。

```c
extern void umain();
extern void libmain();
extern void exit();

extern struct Env *env;
```

然后，我们再看`user/libos.c`文件，上面的几个extern，多数都被定义在了`libos.c`中。

```c
#include "lib.h"
#include <mmu.h>
#include <env.h>

void
exit(void)
{
    //close_all();
    syscall_env_destroy(0);
}

struct Env *env;

void
libmain(int argc, char **argv)
{
    // set env to point at our env structure in envs[].
    env = 0;    // Your code here.
    //writef("xxxxxxxxx %x  %x  xxxxxxxxx\n",argc,(int)argv);
    int envid;
    envid = syscall_getenvid();
    envid = ENVX(envid);
    // 可以看到env正是当前进程的env结构体。
    env = &envs[envid];
    // call user main routine
    umain(argc,argv);
    // exit gracefully
    exit();
    //syscall_env_destroy(0);
}
```

所以，在`fork.c`中，想访问当前进程的env结构体可以使用env这个变量。
其他引用了`lib.h`的地方应该也都可以这么做。

## 问题及解决方案 ##

### 内核不断从头开始跑 ###

这个有可能是由于lab4代码分发时把之前一次实验我们添加的那段用于分发异常的汇编代码给弄没了。
请参照上次实验把异常分发的汇编代码重新添加上就可以了。

### make过程中bintoc报错 ###

错误现象：make的时候报出类似如下错误

```
./bintoc fktest fktest.b > fktest.b.c~ && mv -f fktest.b.c~ fktest.b.c
/bin/sh: 1: ./bintoc: Permission denied
make[1]: *** [fktest.b.c] 错误 126
rm fktest.o
make[1]:正在离开目录 `/home/13061100/13061100-lab/user'
make: *** [user] 错误 2
```

错误原因：bintoc无执行权限。

解决方案：进入user目录下编辑Makefile，在执行bintoc之前添加可执行权限。

```
%.b.c: %.b
echo create $@
echo bintoc $* $< > $@~
chmod +x ./bintoc
./bintoc $* $< > $@~ && mv -f $@~ $@
```

### 运行直接卡死 ###

错误输出如下：

```
panic at init.c:26: ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

pageout:        @@@___0x7f3fe000___@@@  ins a page 
```

且如果使用`gxemul -E testmips -M 64 -C R3000 gxemul/vmlinux -v`来执行会出现下面的输出：

```
panic at init.c:27: ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

[ exception INT cause_im=0x10 pc=0x80011cec <_panic+0x5c> ]
[ exception TLBL <tlb> vaddr=0x00401000 pc=0x00401000 ]
[ exception TLBL <tlb> vaddr=0x7f3fe000 pc=0x00401000 ]
pageout:        @@@___0x7f3fe000___@@@  ins a page 

[ exception TLBS <tlb> vaddr=0x7f3fdffc pc=0x00401be0 ]
[ exception TLBS <tlb> vaddr=0x004085e0 pc=0x00401bfc ]
[ exception MOD vaddr=0x004085e0 pc=0x00401bfc ]
[ exception TLBL <tlb> vaddr=0x00000000 pc=0x00000000 ]
[ warning: LOW reference: vaddr=0x00000000, exception TLBL, pc=0x00000000 <(no symbol)> ]
[ exception TLBS <tlb> vaddr=0xffffff3c pc=0x800110f8 <handle_tlb+0x58> ]
[ exception TLBS <tlb> vaddr=0xfffffea0 pc=0x800110f8 <handle_tlb+0x58> ]
[ exception TLBS <tlb> vaddr=0xfffffe04 pc=0x800110f8 <handle_tlb+0x58> ]
[ exception TLBS <tlb> vaddr=0xfffffd68 pc=0x800110f8 <handle_tlb+0x58> ]
[ exception TLBS <tlb> vaddr=0xfffffccc pc=0x800110f8 <handle_tlb+0x58> ]
[ exception TLBS <tlb> vaddr=0xfffffc30 pc=0x800110f8 <handle_tlb+0x58> ]
[ exception TLBS <tlb> vaddr=0xfffffb94 pc=0x800110f8 <handle_tlb+0x58> ]
// 下面各种一直是TLBS的错
```

错误原因：在load_icode中，映射UTEXT等位置的时候需要使用标志位PTE_V|PTE_R，否则是无法改动全局变量之类的东西的。

### 死循环输出pageout ###

错误输出大致如下

```
pageout:        @@@___0x45e000___@@@  ins a page 
pageout:        @@@___0x45f000___@@@  ins a page 
pageout:        @@@___0x460000___@@@  ins a page 
pageout:        @@@___0x461000___@@@  ins a page 
pageout:        @@@___0x462000___@@@  ins a page 
```

这个问题是由于遍历`fork()`中遍历页表目录及duppage等部分出现错误导致的。具体错误不同需要根据具体情况分析。
一般是由于映射出错，所以在执行子进程的时候CPU读代码引发了缺页中断，然后触发了内核里面的pageout函数，
自动插入了一个空白页。空白页相当于一页的nop指令。于是CPU就欢乐地执行了一大片nop，然后开始访问下一页代码，
然后。。。哇，又是一堆nop。。。CPU愉快地nop啊nop，最后就崩了。。。

### TOO LOW ###

这里感谢罗天歌大神。当page_insert()的传入参数 page如果太小，会 TOO LOW。

准确说是`＊page`的值，就是咱们`sys_ipc_can_send`，看自己怎么写的，如果会这么调用

```c
page = page_lookup( curenv->env_pgdir, srcva, 0 );
page_insert( target->env_pgdir, page, target->env_ipc_dstva, 0 );
//这个时候page_insert里的page == 0，导致 TOO LOW
```

上面两句的主要需要注意的问题在于`page_lookup`在发生错误时会返回一个0。
如果不加判断直接`page_insert`的话就会导致TOO LOW这个错误。
解决方案可以通过判断`page_lookup`返回值，再根据返回值是否为0做分别的处理。

## 关于攻略的补充说明 ##

也许很多人不知道，除了第一次以外，每一次的攻略都是“第一视角攻略”（暂且起这么个名字吧）。
攻略并非完成实验后整理出来的实验流程，而是边做实验边写的，写出来的就是笔者在做实验的时候
真实的分析过程。之后调试完再修正其中错误的地方，再加以补充，最终形成的攻略。

之所以这样做是希望笔者分析问题的思路可以对读者起到帮助。操作系统实验具体做了什么可能大家很快就会忘记，
将来恐怕也不会再用到。但分析并解决问题的思路可能在将来也能起到很大的作用。
因此，攻略试图通过第一视角来展示笔者的分析。希望能够起到哪怕一点点的帮助也好。

攻略形成并发布后，会有诸位大神帮忙来进行修正，同时也会根据私下的一些交流对于攻略中不太好的地方做出修正。
深深地希望能有更多的人参与到这个过程中，分享自己遇到的问题，整理自己的分析和解决方法，
造福一起做实验的朋友们。这也是为什么笔者选择把攻略放在Github上，而非博客或者空间一类的地方的原因。
笔者始终坚信，一个人可以走得很快，但一群人可以走得更远。

最后，欢迎诸位的Pull-Request、Issue、维护以及其他各种贡献，很高兴能和大家一起度过折腾OS实验的欢乐时光^_^
