# 操作系统实验2 #

## 关于实验2 ##

实验2是的核心在于内存管理。这里假定你已经理解了各种概念。

## 实验二参考攻略（可能有错，仅供参考） ##
### 环境配置 ###

### linux ###
这里助教推荐的是Vim+ctags。我个人采用的是GNU Emacs + GNU Global(gtags)的组合，
功能上大致相当，大家可以自由选择。其中Vim/Emacs是编辑器，而ctags/gtags的功能
是在全局查找函数定义之类的，对于阅读代码来说很好用。

另外，这里还有一个小技巧。如果觉得服务器上的环境比较糟糕，可以考虑将代码拉到本地
然后再本地阅读。Linux下可以使用scp命令完成这一工作。

上传文件采用：

```shell
scp /path/file 1306XXXX@10.111.1.110（这个IP自行修改）:/home/1306XXXX/file（目标路径）
```

下载采用：

```shell
scp 1306XXXX@10.111.1.110（这个IP自行修改）:/home/1306XXXX/file（目标路径）/path/file 
```

如果需要上传下载文件夹，可以添加`-r`选项。例如：

```shell
scp -r 1306XXXX@10.111.1.110（这个IP自行修改）:/home/1306XXXX/file（目标路径）/path/file 
```

### windows ###
在windows下可以使用psftp完成相同的工作，psftp可以在putty的工具包中找到，可以通过下面的链接进行下载：

```web
http://www.mycodes.net/do/job.php?job=down_encode&fid=130&id=681&rid=673&i_id=25&mid=101&field=softurl&ti=2
```

连接远程服务器：

```shell
open 10.111.1.11x
```

登录与进入13061xxx-lab文件与平时步骤相同

更改接收文件目录：

```shell
lcd D:\ (just a example)
```

下载采用：

```shell
get -r 接收文件夹名 远程文件夹名
example:get -r mm mm
```

### 阅读顺序及解析 ###

这次推荐从`mips_init`函数开始顺序阅读，这样比较容易理解代码的作用。

第一个映入我们视野的函数是`mips_detect_memory`。显然是在探测内存大小之类的
（实际上根本没探测，直接被指定成了64M，估计是便于大家实验吧）。
这里计算并设置了几个全局变量。

```c
u_long maxpa;            /* 最大物理地址 */
u_long npage;            /* 总页数 */
u_long basemem;          /* 总的base memory的大小 */
u_long extmem;           /* extend memory的总大小，在这次实验中为0*/
```

之后就进入一个非常关键的函数`mips_vm_init`，这里一定要看你一下注释。这个函数
只解决内核空间的部分。在这个函数中，我们碰到了第一个需要我们填写的函数`alloc`。
看上去很吓人，但自习阅读实验指导书发现其实并不复杂。

+ static void * alloc(u_int n, u_int align, int clear)

    分配 n 字节的内存区域，并以 align 对齐，若 clear 为 1 则将分配的区域置全 0。 

根据指导书中的提示，这个函数用于分配无需释放的内存。且指导书中友善地给出了这样的代码：

```c
if (freemem == 0)
   freemem = (u_long)end; 
```

这正是我们需要填的代码之一。而end位处内核镜像（也就是内核代码段、数据段之类的）的位置之后（如图中所示），
所以不难理解，stack到end之间的部分自然就是我们要用于分配的内存空间。这里笔者最开始的判断存在错误，
最开始认为end是类似堆的一个东西。后来从mmu.h中的图里发现，end实际上并不在kernel中，而是在kernel之后。
这样应该更为正确，否则end如果和stack的位置发生重叠（比如alloc过多），那么内核就要崩溃了。而如果把
end放在kernel之后，就不会有这样的问题。根据后面的代码，也可以大致判断，end位于kernel之后更为正确。
这里需要特别注意，lds中的end设置的位置有可能是错误的，需要大家手工改写一下。
这里盗一张图：

```c
/*
 o VPT,KSTACKTOP-----> +----------------------------+----|-------0x8040 0000-------end
 o                     |       Kernel Stack         |    | KSTKSIZE            /|\
 o                     +----------------------------+----|------                |
 o                     |       Kernel Text          |    |                    PDMAP
 o      KERNBASE -----> +----------------------------+----|-------0x8001 0000    | 
 */
```

这里还有一个可能的坑点，就是lds中还需要设置下`.sdata : { *(.sdata) }`使得全局变量被放到正确的位置。

理解到这里，可以判断出freemem每次分配的时使用加法运算，例如分配4字节就是freemem+=4，类似这样。
另外，freemem还需要按照align对齐，这个功能只需使用ROUND宏即可实现：

```c
/* Rounding; only works for n = power of two */
#define ROUND(a, n) (((((u_long)(a))+(n)-1)) & ~((n)-1))
```

然后根据注释的提示就可以顺利完成这一部分。这里再给个提示，有一个神奇的用于清零的宏叫做`bzero`。
定义在了init.c中：

```c
void bzero(void *b, size_t len)
```

再往下阅读就碰到了两个宏：

+ ULIM 代表内核空间的起始地址。

+ PDMAP代表一页的大小

接着按照代码执行顺序进入`boot_pgdir_walk`函数。这个函数很巧妙，同时实现了在页目录`pgdir`中，
找到虚拟地址`va`所在的页表的任务，和创建页表项的功能（如果`create`为1的话）。
通过参数`create`的控制，完美地将这两个功能提取到一个函数中，有效避免了重复代码的产生。

而这时，`mips_vm_init`函数调用它的目的，正是要通过它来创建二级页表。由于这里还是在处理内核
部分的内存，所以分配依旧需要使用`alloc`函数。当需要创建页表的时候，通过`alloc`分配一下物理
内存给targetPde就好（因为此时targetPde为空）。之后重新计算pageTable的值，用于返回。

这里有一个细节，就是页表项里存的是是页面的物理地址。所以在给`targetPde`赋值时，
需要用`PADDR`宏来将内核部分的虚地址转换为物理地址。直接这样说还不够明确，
页表项中储存的实际上是物理页框号，而一页是4K（12位），所以物理页框号只需要20位。
于是MIPS就把剩下的12位定义为标记位。页表项是页面对物理页框的映射，所以
储存的实际是物理页框号和标志位（是的，这32位里同时包含这两个货）。
因此呢，这里还有一个 **大坑** ：整个所有的二级页表占据的4M空间，也需要通过页表管理，
所以这里也需要对应一个物理页框号及标志位，根据老师所讲，负责这个的正是页目录。
由于页目录的自映射，所以页目录里记录的说是二级页表的地址，
实质上是二级页表所在页框的页框号（因为一个页表正好对应一个页面，所以通过页框号即可找到对应的页表）。
说到页框号，大家懂得，也就是说后面需要跟着12位的标志位。
因此，这里需要或一个PTE_V，否则后面会出错：

```c
*targetPde = PADDR((struct Page*)alloc(BY2PG,BY2PG,1))|PTE_V;
```

接下来是`boot_map_segment`，这个函数的作用根据指导书：

> 将物理地址区域[pa, pa+size)映射到以 pgdir 为页目录的[va,va+size)虚拟地址区域
中，同时设置 pde 和 pte 的权限位为 perm。需要注意的是，size 必须是物理页面大小
的倍数，即 4KB 的倍数。 

所以，首先我们要调用boot_pgdir_walk获取va对应的页表项。之后遇到个比较麻烦的问题在于如何设置权限位。
指导书中并未明确说明这一点，不过通过阅读mmu.h中的定义，我们可以推测出具体方法。
这里最关键的宏定义是`PTE_ADDR`：

```c
#define PTE_ADDR(pte)   ((u_long)(pte)&~0xFFF)
```

`(u_long)(pte)&~0xFFF`等价于`(u_long)(pte)&0xFFFFF000`，也就是说，提取了前20位用于地址。
显然，后面的12位是用于储存权限位等的信息的。根据上面写页表部分的经验，这里的pa显然是物理页框号。
所以后面的12位便是标志位了。
接下来，我们可以看到相关标志位的宏定义：

```c
#define PTE_G       0x0100  // Global bit
#define PTE_V       0x0200  // Valid bit
#define PTE_R       0x0400  // Dirty bit ,'0' means only read ,otherwise make interrupt
#define PTE_D       0x0002  // fileSystem Cached is dirty
#define PTE_COW     0x0001  // Copy On Write
#define PTE_UC      0x0800  // unCached
#define PTE_LIBRARY     0x0004  // share memmory
```

可以清晰的看到，全部只用了后12位，也就是16进制下的后3位。这里可以证明我们上述的判断是正确的。
之后关于权限还可以注意到一个非常关键的注释，在`boot_map_segment`这个函数前面，

> Use permission bits perm|PTE_V for the entries.

由此可知，PTE_V这个宏中为1的那个位置应该是用于设置权限的。所以之后就简单了，
只要将页表项设置为`pa|PTE_V|perm`即可，其中`pa`是要映射到的物理地址。这个地址貌似直接就是页框的地址，
后面的12位全部是0。本来以为需要手工左移12位，结果移了以后整个程序都凌乱了。

这里填完以后开始初始化Page数组。这里的第一步是建立空闲内存链表。关于链表的操作，需要查看`queue.h`。
现将重要的部分摘录如下：

```c
#define LIST_ENTRY(type)                        \
struct {                                \
    struct type *le_next;   /* next element */          \
    struct type **le_prev;  /* address of previous next element */  \
}
```

从这一段定义可以看出，实际上pp_link这个部分是一个双向链表的指针域的部分。page_free_list实质上是一个双向链表。
关于这个`page_init`函数到底做什么，有长长的一段注释。

> The exaple code here marks all pages as free.
> However this is not truly the case.  What memory is free?
>  1) Mark page 0 as in use(for good luck)
>  2) Mark the rest of base memory as free.
>  3) Then comes the IO hole [IOPHYSMEM, EXTPHYSMEM) => mark it as in use
>     So that it can never be allocated.
>  4) Then extended memory(ie. >= EXTPHYSMEM):
>     ==> some of it's in use some is free. Where is the kernel?
>     Which pages are used for page tables and other data structures?
>
> Change the code to reflect this.

恩，所以首先需要把第0页标记为已使用。这里之所以要标记为使用，然后再把用过的内存标记为使用。
其他部分插入page_free_list中。个人感觉这一段注释好像说得很有问题。
本质上这里需要思考哪里的内存真正是未被使用的，哪里的是被使用了的。
之前分配的内存外加整个的内核镜像，是在虚拟地址的0x80000000到freemem的位置上的。
但这两个地址都是虚拟地址，对应的实际物理内存是0到PADDR(freemem)。
所以，这之间的页面应该被标记为使用，其他的应该都是未使用的。

之后下一个需要填写的函数就是`page_alloc`。这个比较好填，因为根据提示，只需使用`LIST_REMOVE`
从链表中首先移除页面，然后将这个页面使用`page_initpp`初始化即可。
这里可能的一个困惑是关于参数为何是`struct Page **pp`。这实际上相当于对一个`struct Page *`类型，
也即一个Page结构体指针的引用。原理和下面的代码相似：

```c
void set_an_integer(int *k)
{
    *k = 10;
}

int main()
{
    int w;
    set_an_integer(&w);
    // w = 10 here
}
```

这种用法的作用和return的作用相当，差不多可以说是另一种返回函数结果的方式。
之所以这样做是因为在我们要填写的`page_alloc`中，返回值被用作表明函数是否成功结束。
所以被分配的Page必须以另一种形式传递回去。为了实现这一目的，这里的参数
便需要是一个`struct Page **pp`。函数本应该返回`struct Page*`类型的参数，
但用这种方式必须再加一个`*`才能真正访问到外面的那个变量的地址。
就如上面的C代码，本来应该返回`int`类型的参数，但为了实现这一点，
需要传入原参数的地址。因此参数类型为`int*`。

另外还有一点，之前`alloc`的时候会将页面清空，所以`page_alloc`也需要将页面清空。
这里有个细节一定要把握好，`struct Page`仅仅是一个代表页面的结构体，它代表了4K的内存。
但是，请仔细注意它的定义：

```c
struct Page {
    Page_LIST_entry_t pp_link;  /* free list link */

    // Ref is the count of pointers (usually in page table entries)
    // to this page.  This only holds for pages allocated using 
    // page_alloc.  Pages allocated at boot time using pmap.c's "alloc"
    // do not have valid reference count fields.

    u_short pp_ref;
};
```

OK，仔细注意你看到了什么！对了，没错，仅仅有一个`pp_link`和一个`pp_ref`。说好的4K内存呢！？！
是的，问题就在这里，`struct Page`确实代表了4K内存，但这4K内存并不在这个结构体里。
我们需要找到它所代表的内存的确切地址。仔细一找，pmap.h中一个有趣的函数映入我们的眼帘：

```c
static inline u_long
page2kva(struct Page *pp)
{
    return KADDR(page2pa(pp));
}
```

没错，这个函数可以帮助我们把一个Page结构体转换成对应的地址。
所以，这里只需要获得这个虚拟地址（我们通过虚地址访问内存），然后就可以调用`bzero`来初始化它了。

下面的一个函数比较好填，`page_free`中有一条注释是:

```c
//when to free a page ,just insert it to the page_free_list.
//LIST_INSERT_HEAD(&page_free_list,pp,pp_link);
```

于是，好像把下面那一句前面的注释删了这个函数就实现了。

之后要参照`boot_pgdir_walk`来实现`pgdir_walk`，首先，把`boot_pgdir_walk`复制一份吧^_^
然后我们来对比一下这两个函数的差异。核心的差异在于`pgdir_walk`里面是使用`page_alloc`来分配的。
同时，由于可以分配失败，所以通过一个额外的参数返回页表项。这里函数中多了一个变量叫做
`pageTablePage`，显然，这个变量就是用来存放分配的页表的。不过分配了物理页面还不够，
因为物理页面不过是一个结构体，不是真实的内存。我们需要根据分配到的物理页面计算出真实物理地址的所在。
通过查阅关于page的操作，我们可以看到一个这样的函数：

```c
static inline u_long
page2pa(struct Page *pp)
{
    return page2ppn(pp)<<PGSHIFT;
}
```

显然，这就是我们所需要的。将换算过的真实物理地址给`targetPde`，这里还有两点要注意的，
一个是，这个地址同样也要`| PTE_V`，原因和之前的`boot_pgdir_walk`一样。
第二个是你需要执行`pageTablePage->pp_ref++`操作，因为页表所使用的内存一定是在物理内存里的，
因此我们在分配完页面以后需要给这个页面的引用计数增加1。

```c
*targetPde = page2pa(pageTablePage)|PTE_V;
```

最后通过对ppte赋值，将页表项的地址返回即完成整个工作。

```c
*ppte = (Pte *)(&pageTable[PTX(va)]);
```

至此我们就完成了分配内存的工作。最后两个函数`page_insert`和`page_remove`非常神奇地被填好了，
所以实验二做到这里应该就算完成了。

最终结果应该是这样的：

```
main.c: main is start ...
init.c: mips_init() is called
Physical memory: 65536K available, base = 65536K, extended = 0K
to memory 80401000 for struct page directory.
to memory 80431000 for struct Pages.
mips_vm_init:boot_pgdir is 80400000
pmap.c:  mips vm init success
start page_insert
va2pa(boot_pgdir, 0x0) is 3ffe000
page2pa(pp1) is 3ffe000
pp2->pp_ref 0
end page_insert
page_check() succeeded!
panic at init.c:55: ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

这里主要看最后的`page_check() succeeded!`是否输出，能输出这句话说明助教在`page_check`中所做的判断
都已经通过了，看到这一条基本可以认为实验二已经顺利完成，加油！

## 一些补充知识 ##

### 数组与指针 ###

这次的实验大量使用了指针来传递数组，这个也许在一开始会让很多人困惑。
因为传指针给很多人带来的第一反应是这里传递了一个链表。实际上，在这次实验中，传递的是数组。
其实本质上，数组也是指针，例如：

```c
int a[10];
a[3] = 3;
```

这里的a[数字]的语法实际上一个针对指针的语法，而不是针对数组的语法。C语言中
其实没有特别为数组设立语法，而是直接通过这个指针的语法来实现对数组的访问。
这个语法实质上是一个语法糖。`p[num]`等价于`*(p+num)`，理解到这里，应该就基本上可以把握好了。

### 关于Pde和Pte ###

这里补充一个说明，关于Pde和Pte这两个类型，可能有人会对这两个类型感到很困惑。
因为追查到最后会发现这两个货的定义竟然是这样的：

```c
typedef u_long Pde;
typedef u_long Pte;
```

这是因为，页表本身就是虚地址到物理地址的映射。对于页目录来说，每一项储存的都是下一级页表的地址，
而地址正好是一个32位整数。所以这里采用一个`u_long`来表示一个页表项。这个`u_long`实质上表示的是一个地址。
对于一个页表的页表项，其储存的是一个20位的物理地址，以及12位的标志位，也是32位，采用一个`u_long`储存。

### 关于LIST的使用###

很多人吐槽list不会用，这里做一些简单的摘录。首先关于LIST的宏全部定义在`include/queue.h`中。

首先是如何定义LIST，这里有一个巧妙的宏：

```c
#define LIST_ENTRY(type)                        \
struct {                                \
    struct type *le_next;   /* next element */          \
    struct type **le_prev;  /* address of previous next element */  \
}
```

可以看到，这个宏定义出了指针域，而且是一个双向链表。只需要在type那里填入结构体的类型名，
即可定义出一个链表的指针域部分。在这次实验中，是这样使用的：

```c
typedef LIST_ENTRY(Page) Page_LIST_entry_t;
struct Page {
    Page_LIST_entry_t pp_link;  /* free list link */
    u_short pp_ref;
};
```

先把指针域定义为一个类型，然后再在Page里定义一个这样的域`pp_link`。

另外，链表头的定义也比较有趣（在queue.h中）：

```c
#define LIST_HEAD(name, type)                       \
struct name {                               \
    struct type *lh_first;  /* first element */         \
}
```

这个宏用于声明一个名称为`name`，类型为`type`的链表头指针。通过这个宏定义出来的结构体
正是链表的头指针。写成这样更有一种链表是一种 *对象* 的感觉^_^

之后还有两个重要的宏，分别是`LIST_INSERT_HEAD`和`LIST_REMOVE`。

```c
#define LIST_INSERT_HEAD(head, elm, field) do {             \
    if ((LIST_NEXT((elm), field) = LIST_FIRST((head))) != NULL) \
        LIST_FIRST((head))->field.le_prev = &LIST_NEXT((elm), field);\
    LIST_FIRST((head)) = (elm);                 \
    (elm)->field.le_prev = &LIST_FIRST((head));         \
} while (0)

#define LIST_REMOVE(elm, field) do {                    \
    if (LIST_NEXT((elm), field) != NULL)                \
        LIST_NEXT((elm), field)->field.le_prev =        \
            (elm)->field.le_prev;               \
    *(elm)->field.le_prev = LIST_NEXT((elm), field);        \
} while (0)
```

其中，`head`可以看出来，实际上是链表的头指针。`elm`代表你要插入的元素的指针。
在实验中，我们需要填写`pages+i`或者`&pages[i]`。
最后，`field`代表链表指针域的名字，比如对于咱们实验中的Page结构体，`field`部分就叫`pp_link`。


## 实验2参考图例 ##
![image](https://github.com/1306SCSE/something_about_os_experiment/raw/master/UnstandStand.PNG)
