# #define 操作系统宏查找手册 #


###No.1 ROUND  
`宏名称:` `ROUND(freemem,align)`   
`宏位置:` `types.h`  
`宏定义:`
```C
#define ROUND(a, n)   (((((u_long)(a))+(n)-1)) & ~((n)-1))
```
`宏作用:`  
`以align对齐freemem 。`  
`Example:freemem=0x80103111，align=0x1000。
对齐后freemem=0x80104000(这里align必须为2的幂次)`


###No.2   bzero
`宏名称:` `bzero(void*, size_t)`   
`宏位置:` `mmu.h`  
`宏定义:`
```C
void bzero(void *b, size_t len){
        void *max;
        max = b + len; 
        while (b + 3 < max)
        {
                *(int *)b = 0;
                b+=4;
        }
        while (b < max)
        {
                *(char *)b++ = 0;
        }
}

```
`宏作用:`  
`bzero函数的主要功能就是用来清零。`   
`我们可以发现，实际上在空间大于4的时候使用强制转换为int类型的指针来进行清零；当空间小于4时，则使用char型指针清零。（因为int是4个字节，而char是1个字节）`

###No.3   Pde,Pte
`宏名称:` `Pde,Pte`   
`宏位置:` `mmu.h`  
`宏定义:`
```C
typedef  u_long  Pde;
typedef  u_long  Pte;
```
`宏作用:`  
`Pde，Pte的原形其实就是u_long,u_long即unsigned long,无符号整数,占4个字节。`   
`其实选择unsigned long,4个字节,而指针是要寻找到4G虚拟空间的,所以指针也是4个字节,所以在这一点上来说,指针和u_long的互相转换是完全没有问题的。`

###No.4   PDX(va),PTX(va)
`宏名称:` `PDX(va),PTX(va)`   
`宏位置:` `mmu.h`  
`宏定义:`
```C
#define PDX(va)    ((((u_long)(va))>>22) & 0x03FF)
#define PTX(va)    ((((u_long)(va))>>12) & 0x03FF)
```
`宏作用:`  
`PDX宏实际上就是提取va的页目录项索引,所以要右移22位。再和只有低12位为1的0x03FF&一下,即可得到页目录索引。PTX宏实际上是用来提取页表项的索引，向右移动12位，低10位则是页表项的索引。`

###No.5   PTE_ADDR
`宏名称:` `PTE_ADDR(pte)`   
`宏位置:` `mmu.h`  
`宏定义:`
```C
#define PTE_ADDR(pte)   ((u_long)(pte)&~0xFFF)
```
`宏作用:`  
`PTE_ADDR宏是用来将pte的值后12位清零。`  
`注意一点细节：每一页的基址都是后12位为零的，不论是页表，页目录，或者是普通的页。`

###No.6  KADDR，PADDR
`宏名称:` `KADDR(pa),PADDR(kva)`   
`宏位置:` `mmu.h`  
`宏定义:`
```C
#define PADDR(kva)                                              \
({                                                              \
        u_long a = (u_long) (kva);                              \
        if (a < ULIM)                                   \
                panic("PADDR called with invalid kva %08lx", a);\
        a - ULIM;                                               \
})
#define KADDR(pa)                                               \
({                                                              \
        u_long ppn = PPN(pa);                                   \
        if (ppn >= npage)                                       \
                panic("KADDR called with invalid pa %08lx", (u_long)pa);\
        (pa) + ULIM;                                  \
})

内部宏补充：
#define PPN(va)  (((u_long)(va))>>12)
```
`宏作用:`  
`KADDR是给定物理地址获得其虚拟地址，PADDR是给定虚拟地址获得其物理地址`  
`KADDR和PADDR的功能相反，KADDR是给定一个物理地址，获得其虚拟地址，如果给定的物理地址超过最大物理内存的话则报错；PADDR是给定一个内核虚拟地址，获得其物理地址，如果给定的虚拟地址超过ULIM的话则报错。实际上这里的ULIM的值就是0x80000000，即内核与用户区的分界线。PPN这个宏实际上就是就是提取pa(physical page)的物理页框号,npage是最大的物理页框号。`

###No.7  PTE_V,PTE_R
`宏名称:` `PTE_V,PTE_R`   
`宏位置:` `mmu.h`  
`宏定义:`
```C
#define PTE_V      0x0200
#define PTE_R      0x0400
```
`宏作用:`  
`PTE_V的功能如注释所说，是Valid bit,而PTE_R是dirty bit。`  
`这两者实际上对应的十六进制数均为在低12位中某一位为1的一个二进制数。比如PTE_V，0x0200H = 10 0000 0000 b 。产生的原因则是因为我们在页目录项和页表项中存入的物理地址只有前20位是有效的地址，因为我们操作系统从0开始按页分配，1页是4KB，所以一页的物理地址的基址后12位始终为0。而我们在页目录项和页表项中存入的最终却是32位的二进制数，一方面是考虑到了字节对齐，一方面在后12位中加入了一些flag位来表明其目前的状态。而PTE_这里的宏定义的值即规定了第几位是什么flag。`

###No.8  LIST_INIT
`宏名称:` `LIST_INIT (&page_free_list);`   
`宏位置:` `queue.h`  
`宏定义:`
```C
#define LIST_INIT(head) do {                                            \
        LIST_FIRST((head)) = NULL;                                      \
} while (0)
内部宏补充：
LIST_FIRST((head)): #define LIST_FIRST(head)  ((head)->lh_first)
head: #define LIST_HEAD(name, type)                                   \
struct name {                                                           \
        struct type *lh_first;  /* first element */                  \
}
LIST_HEAD(Page_list, Page);
```
`宏作用:`   
`初始化空闲物理页面链表头。`  
`空闲物理页面链表是双向链表。head为指向srtuct Page_list结构体的首指针，这个宏定义的意思是初始化首指针中的lh_first的值为0。`

###No.9  LIST_INSERT_HEAD
`宏名称:` `LIST_INSERT_HEAD(&page_free_list,&pages[i],pp_link)`   
`宏位置:` `queue.h`  
`宏定义:`
```C
#define	LIST_INSERT_HEAD(head, elm, field) do {				\
	if ((LIST_NEXT((elm), field) = LIST_FIRST((head))) != NULL)	\
		LIST_FIRST((head))->field.le_prev = &LIST_NEXT((elm), field);\
	LIST_FIRST((head)) = (elm);					\
	(elm)->field.le_prev = &LIST_FIRST((head));			\
} while (0)
```
`宏作用:`   
`把结点插入空闲物理页面链表中。`  
`Head是头指针，elm是当前元素指针，field是指在struct里面的成员，实际上这个就是一个建立头结点，并依次在头结点前面插入结点的一个宏。其中尤其值得注意的就是如果一开始head为空的话，那么就会自动帮你建立一个头指针，之后再进行结点的插入，每次插入的结点都会成为新的头结点。`

###No.10  LIST_REMOVE 
`宏名称:` `LIST_REMOVE (LIST_FIRST(&page_free_list),pp_link); `   
`宏位置:` `queue.h`  
`宏定义:`
```C
#define  LIST_REMOVE(elm, field) do {					\
	if (LIST_NEXT((elm), field) != NULL)				\
		LIST_NEXT((elm), field)->field.le_prev = 		\
		    (elm)->field.le_prev;				\
	*(elm)->field.le_prev = LIST_NEXT((elm), field);		\
} while (0)
```
`宏作用:`  
`稍后会放图片来解释一下这个过程`

###No.11  page2pa
`宏名称:` `page2pa(pageTablePage)  `   
`宏位置:` `pmap.h`  
`宏定义:`
```C
static inline u_long
page2pa(struct Page *pp)
{
	return page2ppn(pp)<<PGSHIFT;
}
内部宏补充：
page2ppn: 
static inline u_long
page2ppn(struct Page *pp)
{
	return pp - pages;
}
PGSHIFT:
#define PGSHIFT	12
```
`宏作用:`  
`使用结构体指针寻找其对应物理地址。`    
`事实上我们发现这个函数用结构体指针来找其物理地址是很玄妙的。首先用pp-pages得到的是物理页框号，这一步很巧妙，使用了映射的相对偏移比例一直。因为一页对应一个4B的物理地址，对应一个页框号。那么页框号左移12位其实得到的就是物理地址。`
