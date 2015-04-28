# #define 操作系统宏查找手册 #


No.1   
`宏名称:` `ROUND(freemem,align)`   
`宏位置:` `types.h`  
`宏定义:`
```C
#define ROUND(a, n)   (((((u_long)(a))+(n)-1)) & ~((n)-1))
```
`宏作用:`
`以align对齐freememExample:freemem=0x80103111
align=0x1000
对齐后freemem=0x80104000
(这里align必须为2的幂次)`


