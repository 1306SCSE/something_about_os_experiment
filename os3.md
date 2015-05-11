# 操作系统实验3 #

## 关于实验3 ##

实验3是的核心在于进程管理。这里假定你已经理解了各种概念。

## 实验三参考攻略（可能有错，仅供参考） ##

哎，以后两周一交，出攻略的压力又大了，求人手啊。。。

好吧，吐槽完毕，我们开始。同往常一样，我们从`mips_init()`函数开始。先看看有什么变化，
好的，显而易见的是，多了个`env_init()`的调用。于是我们顺着进入这个函数，
然后惊喜地发现——这就是我们需要实现的第一个函数。

`env_init()`定义在`lib/env.c:81`，这次我尽量写出行号（由于种种原因，可能有误差）。
因为上次发现很多同学不方便找具体定义，为了方便大家，这里直接写出来具体位置。
这里的注释相对清晰，首先是初始化`env_free_list`。这个操作和之前一次使用初始化`page_free_list`是很像的。
所以，参考上一次的代码，我们使用`LIST_INIT`宏初始化`env_free_list`。
之后，按照注释对结构体数组进行初始化和插入（`LIST_INSERT_HEAD`）即可。
这里要注意注释中的一句话

> the env_free_list. attention :Insert in reverse order

也就是说，我们需要按照envs的倒序插入到`env_free_list`中。

之后按照提示，我们需要创建两个进程，进程的二进制代码位于`init/code_a.c`和`init/code_b.c`中。
我们打开这两文件，会发现很好玩的事——居然把两个进程的二进制代码直接嵌在.c里了。。。
好拼啊。。。那么长的二进制。。。好吧，我们继续。查看这两个文件我们可以看到，每个文件实际上都定义了两个全局变量。
我们以`code_a.c`为例，`unsigned char binary_user_A_start[]`代表的是二进制代码，`unsigned int binary_user_A_size`
代表的是二进制代码的大小。而根据注释

> you may want to create process by MACRO, please read env.h file, in which you will find it. this MACRO is very
  interesting, have fun please

我们需要用定义在`env.h`中的宏来创建新的进程。于是，我们去查看`env.h:66`，可以看到用于创建进程的宏。

```c
#define ENV_CREATE(x) \
{ \
    extern u_char binary_##x##_start[];\
    extern u_int binary_##x##_size; \
    env_create(binary_##x##_start, \
        (u_int)binary_##x##_size); \
}
```

这个宏里的语法大家可能以前没有见过，这里解释一下：##代表拼接，例如
（以下例子是转载的，出处为http://www.cnblogs.com/hnrainll/archive/2012/08/15/2640558.html，
想深入了解的同学可以参考这篇博客）。

```c
#define CONS(a,b) int(a##e##b) 
int main() 
{
    printf("%d\n", CONS(2,3));  // 2e3 输出:2000 
    return 0; 
} 
```

所以，结合`init/code_a.c`和`init/code_b.c`两个文件以及`include/env.h`，我们可以发现，创建进程的代码为

```c
ENV_CREATE(user_A);
ENV_CREATE(user_B);
```

我们顺着这个宏看，会发现，它实际上就是调用`env_create()`函数来创建进程。跟踪进入`env_create()`函数（`lib/env.c:188`）。
于是，发现了第二个要我们填写的函数。注释比较清晰，这里翻译一下重点： **分配一个新的Env结构体并将二进制代码载入到对应地址空间**
同时，注释里还给了一个注意事项 **新的env的父env结构体id是0** 。分配env结构体需要调用`env_alloc()`函数（`lib/env.c:141`），函数原型如下：

```c
// Allocates and initializes a new env.
// 分配并初始化一个新的env
//
// RETURNS
//   0 -- on success, sets *new to point at the new env 
//        成功时，设置*new指向新分配的env结构体
//   <0 -- on failure
//        失败时，返回一个小于0的值
//
int env_alloc(struct Env **new, u_int parent_id)
```

也就是说，首先要传入一个`struct Env*`类型的变量的地址，用于存放新分配的新分配的Env结构体，同时需要传一个父进程的id。
（关于为什么struct Env **new是用于接收新分配的env结构体的解析可以参看上一次的攻略，和其中page_alloc的用法相仿）。
所以，我们在`env_create`中定义一个`struct Env`指针变量，并调用`env_alloc`即可完成第一步。

```c
struct Env *new = NULL;
/*1. allocate a new Env*/
env_alloc(&new, 0);
```

然后根据注释，我们需要调用`load_icode`，原型如下：

```c
// Sets up the the initial stack and program binary for a user process.
// 为一个用户进程初始化栈并载入二进制程序
//
// This function loads the complete binary image, including a.out header,
// 这个函数会载入完整的二进制镜像，包括a.out的头部，
// into the environment's user memory starting at virtual address UTEXT,
// and maps one page for the program's initial stack
// at virtual address USTACKTOP - BY2PG.
// 镜像会被载入到环境的用户内存中（虚拟地址起始于UTEXT），
// 同时为程序的初始栈将USTACKTOP-BY2PG开始的一页内存映射到物理内存。
// Since the a.out header from the binary is mapped at virtual address UTEXT,
// the actual program text starts at virtual address UTEXT+0x20.
// 由于a.out的头部被映射到了虚拟地址UTEXT，所以程序实际的text段起始于UTEXT+0x20。
//
static void load_icode(struct Env *e, u_char *binary, u_int size)
```

这个函数的调用方式一看就可以看懂，第一个参数为env结构体，第二个参数为程序的二进制，第三个参数为程序的长度。
最后，在载入完程序后，把进程设置为ENV_RUNNABLE，让它可以被调度即可。
到这里，`env_create()`就完成了。在填写`env_create()`的时候，我们发现了`env_alloc()`和`load_icode()`都需要填写。
所以，接下来我们来解决`env_alloc()`的问题。

`env_alloc()`函数比较简单，按照注释填写即可，这里就不再赘述了。获取`env_free_list`的第一个元素采用`LIST_FIRST`宏。
之后根据指导书调用`env_setup_vm`来为这个env初始化内核内存布局。下一句注释比较恶心，它说需要把env的各个段设置为合适的值。
（什么叫做合适的值啊！？！）根据注释，env_ipc*,env_pgfault_handler和env_xstacktop都是实验4使用的，
后面的env_runs是实验6使用的，所以应该不需要设置。
于是，我们需要设置parent_id、id、status和env_tf就可以了。全局唯一的id需要调用`u_int mkenvid(struct Env *e)`（`lib/env.c:22`）来获得。
parent_id正常设置就好，status应该设为ENV_NOT_RUNNABLE。比较关键的是后面的那句注释：

> focus on initializing env_tf structure, located at this new Env. especially the sp register,
  CPU(个人感觉这里应该是笔误，应当是CP0) status and PC register(the value of PC can refer the comment of load_icode function)

sp寄存器的位置（29号寄存器）根据`include/mmu.h`可以知道是`USTACKTOP`这个地址。pc根据`load_icode()`中的注释，
应该设在`UTEXT+0x20`的位置，也即程序的text段的位置。 
**但这里请务必注意，如果你按照注释设置为UTEXT+0x20。那么你的实验一定搞不定。因为这个地址是错误的！！**
正确的地址为`UTEXT+0xb0`，具体如何找出这个地址的，笔者会在附录中说明。
不得不吐槽一下 **注释神坑！！！这么关键的地址居然都是错的！！！注释的语气还那么肯定！！！**
好吧，我们继续，吐槽就到这里。 **请一定记住要将PC初始化为UTEXT+0xb0，血的教训换来的结论**
同时，根据指导书，我们为了支持时钟中断，需要将e->env_tf.cp0_status的值初始化为0x10001004。

最后调用`LIST_REMOVE`宏将新的env从`env_free_list`中移除。`env_alloc()`函数至此结束。

下面一个函数让笔者纠结了很久。根据提示，需要将代码拷贝到env的虚地址上。想拷贝的话显然要将虚地址先映射到物理内存上，
这样才能进行拷贝。虚地址不对应实际的物理内存的话，是根本没法进行读写操作的。
就好像一个逻辑上的东西必须对应到具体的物理事物上才能进行操作一样。虚地址也必须对应到物理内存上，
才能真正对这个地址进行读写操作。因为此时尚未开启中断支持（到这里我们还没调用`trap_init()`），所以应该是不能处理缺页中断的。
也就是说，所有的地址我们都需要自行把它映射到物理内存中。映射到物理内存是两个步骤，首先调用`page_alloc`获取一个物理页，
然后用`page_insert`实现映射。这里还是多说一句，一共需要映射够`size`那么多的内存，
所以这里需要一个循环，循环到映射够足够的内存为止。

还有一个问题就是，这里我们所使用的页表依旧是内核的页表，而不是env的页表，
所以把env的页表中的虚地址映射到物理内存上
并不能使我们访问这段内存（现在不能，进程被调度的时候可以）。
CPU此时还在使用内核页表进行地址转换，只有后面进程被调度的时候，才开始使用进程页表做转换。
因此虽然此时已经做了映射，但由于是映射到了进程页表上，所以我们依旧无法通过这个虚地址直接访问相应内存。
这也就是为什么我们需要通过内核虚地址访问这个内存页面。
也就是使用`page2kva`来获取物理内存的虚拟地址以访问到它。最后再调用`bcopy`函数进行拷贝。
其中，`bcopy()`定义在`init/init.c:33`，原型如下

```c
void bcopy(const void *src, void *dst, size_t len)
```

在这之后，根据函数声明前所给的注释（本攻略翻译的那段里），我们还需要为程序映射一页作为初始的栈。
于是，我们又需要调用一遍`page_alloc`和`page_insert`来实现这一映射。
这一处的映射和以往我们所做的有略微差异，调用的时候`perm`除了设置PTE_V以外还需要设置`PTE_R`，表示这里可写。
因为这是进程的栈，所以必须让它可写，否则程序就没法用了。也就是说`perm`参数需要传入`PTE_V|PTE_R`。

下面的一个注释是说，我们需要确认pc的位置是正确的（这里的注释误导了我好久。。。它的地址是错的）。
这里的确认应该指的是assert，所以我们需要添加一句`assert(e->env_tf.pc == UTEXT + 0xb0);`。

至此，我们就又完成了一大段内容。之后我们会发现一个很恶心的问题。我们不知道后面应该看何处的代码了。
因为之后调用的两个函数为`trap_init()`和`kclock_init()`。这两个函数都已经被实现了。
那么在此之后执行的是哪一段代码！？！原因是这样的，到了这里我们的操作系统已经由一个静态的过程，
变为一个动态的过程了。不再是一条一条地执行代码，而是不断被中断等等驱动，来执行后面的工作。
就好像一个服务员，先要穿戴整齐，准备好工作服之类的（这就是我们的操作系统之前所做的工作），
然后就要为用户服务，根据用户的需求东奔西跑、忙这忙那（这是我们的操作系统之后所要做的工作）。
所以，我们会发现，后面的代码不再是按照一个静态的顺序执行的了。

这时，我们重新看直到书给的图，可以发现这之后操作系统会开始响应中断，
并调用`sched_yield`函数来调度一个进程执行。所以，接下来，我们来看一看这个调度函数。
这个函数位于`lib/sched.c:16`

这里根据提示我们需要实现一个轮转调度算法。所以，我们需要定义一个局部静态变量，
用于记录我们当前轮转到了哪里。同时，寻找下一个可以被调度的进程，调用`env_run`来执行这个进程。
这里为了方便还是给出代码吧

```c
static int i = 0;
while(1)
{
    i = (i+1)%NENV;
    if (envs[i].env_status == ENV_RUNNABLE) {
        /*call the function we have implemented at env.c file, in which file I have give you some hints*/
        env_run(&envs[i]);
    }
}
```

我们先来看一看`env_run`，这个函数用于保存当前进程并切换到新的进程。
首先第一步是需要保存trap frame。这一步笔者折腾了好久，最后在`syscall_all.c:31`找到了灵感（感谢何涛大神）。

```c
// deschedule current environment
void sys_yield(void)
{
        bcopy((int)KERNEL_SP-sizeof(struct Trapframe),TIMESTACK-sizeof(struct Trapframe),sizeof(struct Trapframe));
        sched_yield();
}
```

根据这个函数的注释，这个函数会重新触发调度。我们可以看到，
它将当前的运行信息保存在了`TIMESTACK-sizeof(struct Trapframe)`
这个位置上了。（ *这里做个额外说明，上面引用的代码只是为了说明我们如何推断出当前运行信息所在的位置。* ）
因此，在`env_run`中，我们将保存在`TIMESTACK-sizeof(struct Trapframe)`的当前进程信息进一步转存到env结构体中。
也就是说，我们需要调用bcopy，把`TIMESTACK-sizeof(struct Trapframe)`的内容拷贝到`&curenv->env_tf`这个地址。
那么，这里有一个 **大坑** （还是感谢神一般的何涛大神）：这里还需要把env_tf.pc的地址设置为cp0_epc的位置。
个人推测是这样：进程切换是由于时钟中断等各种中断触发的，所以，这个进程在再次被恢复执行的时候，
应该执行导致（或遭受）异常的那条指令，相当于一个大的异常处理程序处理结束，返回到原来的程序中，
所需要执行的那条指令。而异常返回地址记录在EPC中。也就是说，应该执行EPC（异常返回地址）寄存器中储存的那条指令。
如果不加这样的处理，那么gxemul模拟时会产生`panic:^^^^^^TOO LOW^^^^^^^^^ `一类的输出。
这里做一个提醒，当前进程不一定存在，因为`curenv`可能为`NULL`（在一开始的时候）。
所以需要首先判断一下。（这里太坑了，所以直接给出代码，请大家自行理解后重写一下，不要照抄，否则没收获的）。

```c
struct Trapframe *old = NULL;
if (curenv) {
    old = TIMESTACK - sizeof(struct Trapframe);
    bcopy(old,&curenv->env_tf,sizeof(struct Trapframe));
    curenv->env_tf.pc = old->cp0_epc;
}
```

接下来，按照注释将curenv赋值为e。
第3和第4步有些小恶心，需要阅读`env_asm.s`中的汇编代码并调用这两个用汇编编写的函数。

直接看这两个函数的末尾部分，可以看到并没有向v0等寄存器赋值，说明没有返回值。
接着我们来分析传参。对于`env_pop_tf`，可以在函数开头看到

```asm
move        k0,a0 
mtc0        a1,CP0_ENTRYHI
```

使用了a0和a1寄存器的值，说明有两个参数。第一个参数根据下面的一串lw的用法可以看出，它是TrapFrame结构体的首地址。
第二个应该是该进程的id，根据变量名直接猜测的，后经验证应该是正确的。

而`lcontext`就比较好读懂了，直接将a0里的值赋值给mCONTEXT而已。根据`pmap.c:165`中的`mCONTEXT = (int)(pgdir);`
可以知道，这里也需要把当前进程的pgdir传入。

env.c中的两句extern也证明了我们对于传参的判断是正确的。

```c
extern void env_pop_tf(struct Trapframe *tf,int id);
extern void lcontext(u_int contxt);
```

那么接下来，我们只要根据上面的推断调用这两个函数就OK了。

之后，实验3的流程图（这个神坑的图只能在windows上用adobe的pdf阅读器打开。。。）里面还有两段说明。
首先，中断处理的代码段为

```asm
.section .text.exc_vec3
NESTED(except_vec3, 0, sp)
     .set noat
     .set noreorder
  1:
     mfc0 k1,CP0_CAUSE
     la k0,exception_handlers
     andi k1,0x7c
     addu k0,k1
     lw k0,(k0)
     NOP
     jr k0
     nop
END(except_vec3)
```

需要注意的一点是，这段代码中的`.set noat`定义了后面的程序中不使用at，而如果你把这段代码放在_start之前，那么后面的代码就需要用到at，此时会报一个error。
这时需要在`END`之后加入`.set at`来解决这个问题。
这段代码需要用到exception_handlers，所以应该是和traps.o链接到一起的一个汇编文件里。
所以我选择放在genex.S的最后。然后修改lds文件，增加：

```lds
. = 0x80000080;
.except_vec3 : {
    *(.text.exc_vec3)
}
```

然后根据指导书，我们的任务就已经完成了。接下来，自行编译调试除错就好了^_^祝大家实验3愉快！

如果一些正常，那么最后结果为

```
init.c: mips_init() is called

Physical memory: 65536K available, base = 65536K, extended = 0K

to memory 80401000 for struct page directory.

to memory 80431000 for struct Pages.

mips_vm_init:boot_pgdir is 80400000

pmap.c:  mips vm init success

panic at init.c:27: ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 2 2  
2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 2 2  
2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 2 2 2  
2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 2 2 2  
2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 12 2 2 2  
2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2  1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 2 2 2 2 2  
2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 1
```

后面会不断的输出1和2.

## 一些补充知识 ##

### 死循环可能原因 ###

笔者在本次实验中遇到的最大困难就是解决死循环的问题，死循环出现的原因大致现在有两种：

1、即使完成了实验第一部分，仍旧会死循环。

这时候的死循环现象会非常有特点，在第一个  
`panic at init.c:27(这个数无所谓): ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^`
显示之后，代码的输出会有 明显的停顿，注意这里有**一个明显的停顿 ！！**，之后会**无停顿连贯性地输出**  
从`main.c: main is start`到`panic at init.c:27(这个数无所谓):^^^^^^^^^^^^^^^^^^^^^^^^^^^`  
如果你是这样死循环的，那么恭喜你，你仅需要改动一小部分代码就有可能直接通过第一部分！需要添加的代码在我们的那个大图里，列在下面：

```c
.section .text.exc_vec3
NESTED(except_vec3, 0, sp)
     .set noat
     .set noreorder
  1:
     mfc0 k1,CP0_CAUSE
     la k0,exception_handlers
     andi k1,0x7c
     addu k0,k1
     lw k0,(k0)
     NOP
     jr k0
     nop
END(except_vec3)
```
把这个添加到任意一个.S文件中都可以，可以添加到`./boot/start.S`中。第一部分成功通过的标志就是停在  
`panic at init.c:27(这个数无所谓): ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^`  
不会再输出别的信息，如果想检测第一部分是否完整的话，可以在`init.c`里面直接  
```c
ENV_CREATE(user_A);
env_run(&envs[0]);
```
结果是会死循环输出1

2、纯属无限输出循环。
最可能的地方是你的进程二进制代码段等的入口地址不正确！请仔细研读上方关于UTEXT+0xb0的部分，是个坑。

### ELF文件分析 ###

笔者来说说怎么发现的`UTEXT+0xb0`这个神地址的。
这次我们跑的进程是两个ELF格式的可执行文件。你可以将原来的程序拷贝一份，并添加如下语句，编译运行，以将其输出出来。

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

unsigned char binary_user_A_start[] = {} //此处略，太多了，
unsigned int binary_user_A_size = 5298;

int main() {
        int fd = open("test_user_a.o",O_WRONLY);
        write(fd,binary_user_A_start,binary_user_A_size);
        close(fd);
        return 0;
}
```

之后在命令行里执行`touch test_user_a.o`建立一个名字为`test_user_a.o`的文件。
接着编译（`gcc -o test 文件名.c`）并运行（`./test`）这个程序。

然后，我们会得到一个ELF格式的可执行文件。如何知道它是ELF格式呢？ELF格式的头4个字节叫做魔数（Magic），
这个数相当于是ELF文件的标识，只要头4个字节为`7f 45 4c 46`就可以判断这个文件是个ELF文件。
这四个字符翻译过来是`0x7f E L F`。ELF文件头部的具体信息如下（通过man elf可以查到）。

```c
#define EI_NIDENT 16

typedef struct {
    unsigned char e_ident[EI_NIDENT];
    uint16_t      e_type;
    uint16_t      e_machine;
    uint32_t      e_version;
    ElfN_Addr     e_entry;           // 这个就是程序的入口地址的位置
    ElfN_Off      e_phoff;
    ElfN_Off      e_shoff;
    uint32_t      e_flags;
    uint16_t      e_ehsize;
    uint16_t      e_phentsize;
    uint16_t      e_phnum;
    uint16_t      e_shentsize;
    uint16_t      e_shnum;
    uint16_t      e_shstrndx;
} ElfN_Ehdr;
```

其中，关于各个部分的大小，man中也有详细说明。

```
The following types are used for N-bit architectures (N=32,64, ElfN stands for Elf32 or Elf64, uintN_t stands for uint32_t or uint64_t):

ElfN_Addr       Unsigned program address, uintN_t
ElfN_Off        Unsigned file offset, uintN_t
ElfN_Section    Unsigned section index, uint16_t
ElfN_Versym     Unsigned version symbol information, uint16_t
Elf_Byte        unsigned char
ElfN_Half       uint16_t
ElfN_Sword      int32_t
ElfN_Word       uint32_t
ElfN_Sxword     int64_t
ElfN_Xword      uint64_t
```

其中u代表unsigned，int代表整数，16/32/64代表占多少位。

可以看到程序入口地址的位置位于binary_user_A_start[24]到binary_user_A_start[27]这4个字节。
我们可以通过实验环境中提供的`mips_4KC-readelf`来直接观察到这个地址。

```shell
mips_4KC-readelf -a ./test_user_a.o
```

然后会输出

```
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
  Entry point address:               0x4000b0
  Start of program headers:          52 (bytes into file)
  Start of section headers:          4016 (bytes into file)
  Flags:                             0x50001001, noreorder, o32, mips32
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         3
  Size of section headers:           40 (bytes)
  Number of section headers:         13
  Section header string table index: 10
  // 后面略
```

Entry point address就是入口地址，也就是0x4000b0。而UTEXT为0x400000，因此可以判断出需要加b0。
所以，如果希望你的实验更通用一点，也可以选择在`load_icode`中再设置pc的值。

```c
e->env_tf.pc=*((int*)binary+6); 
// 由于把binary转为了int型指针，所以+6相当于向后偏移24，也就是Entry point address这一信息的起始地址。
// 之后取这里的内容（由于已经转换为了int型指针，所以一次会取4字节）就是入口地址了。
```

