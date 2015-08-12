# 操作系统实验1 #

## 关于实验1 ##

个人感觉实验一相当于是操作系统中的Hello World。我们主要需要完成的工作是这样几个：
首先是需要完成环境的配置，这里也是最坑的一点，比较恶心。
第二是需要熟悉各种工具软件的使用，对于不熟悉命令行操作的同学会是一种折磨。
第三是需要改写lds文件，以及make相关的文件，并填写start.s，以编译出可启动的系统。
第四是需要填写lp_printf函数，做个printf函数出来。
最后完成各种实验报告之类的，这个实验就算完成了。

## 实验一参考攻略（可能有错，仅供参考） ##
### 环境配置 ###

环境配置这里如果参照指导书做会很难受，因为那套工具链的版本太低，而且据尝试过的
同学那个ISO里的安装脚本对高版本Ubuntu各种不兼容。所以这里我们使用一个兼容性更好
的方案。我们通过源代码来自己直接编译一个工具链，这样能折腾的少一点。

这里假设大家使用Debian/Ubuntu系列的操作系统，也就是使用apt来管理软件包的系统，
其他系统的话请自行安装编译gcc所需依赖。（自行百度“ＸＸＸ　编译ｇｃｃ”关键字）。
Apt有个很好的功能就是可以自动解决编译依赖，我们需要编译gcc及binutils两个软件，
所以首先执行：

    sudo apt-get build-dep binutils
    sudo apt-get build-dep gcc

这样就可以自动安装编译所依赖的软件包。其他发行版请自行解决编译依赖。

编译GCC需要这些依赖：`gmp`, `mpfr`, `mpc`。三个库：

Debian/Ubuntu系列发行版可以执行以下命令安装：

    sudo apt-get install libgmp-dev
    sudo apt-get install libmpfr-dev
    sudo apt-get install libmpc-dev

RHEL/Fedora系列发行版可以执行以下命令安装：

    sudo yum install libgmp-devel
    sudo yum install libmpfr-devel
    sudo yum install libmpc-devel

之后，在一个合适的位置建立个文件夹用于编译（我是在主目录下建立了一个mips-tools，
所以以下 **假定使用/home/wlm/mips-tools** 这个目录，大家根据自己的情况 **自行修改** ）

首先下载binutils和gcc的源码包，为了快，我们选用[北京交通大学的镜像站](http://mirror.bjtu.edu.cn/gnu/) 。
我们需要下载[binutils的最新版本](http://mirror.bjtu.edu.cn/gnu/binutils/binutils-2.25.tar.gz)，
[gcc的最新版本](http://mirror.bjtu.edu.cn/gnu/gcc/gcc-4.9.2/gcc-4.9.2.tar.gz)，
以及[gxemul的0.4.6.6版](http://gxemul.sourceforge.net/src/gxemul-0.4.6.6.tar.gz)。
请特别注意 **gxemul一定要用0.4的，最新的0.6版本和咱们的实验不兼容**　，
(**笔者注：在完成后面的实验时，笔者发现gcc 4.9.2会造成实验产生奇怪的错误，可以的话还是选用4.0版本的gcc吧**)
将这三个包解压到mips-tools目录（关于这个目录，参见上文的假定）中，解压完后大致是这个样子：

    ~/mips-tools $ ls
    binutils-2.25         gcc-4.9.2         gxemul-0.4.6.6

接着我们建立两个目录用于编译binutils和gcc，他们必须在单独的目录中构建，
特别是gcc，直接 **在源码所在目录下构建可能会出错** ！

    # 首先进入到mips-tools目录，请根据自己的目录路径自行调整下面的命令
    $ cd /home/wlm/mips-tools
    # 建立一个单独的目录用于构建
    $ mkdir build
    $ cd build
    $ mkdir binutils
    $ mkdir gcc

接下来开始构建交叉编译工具链：

    # 这段操作请接着上段操作做
    $ cd binutils
    # 执行configure对构建进行配置，target代表目标机的信息，
    # prefix代表一会儿要安装的位置，默认是在/usr/local/下
    $ ../../binutils-2.25/configure --target=mips-elf --prefix=/home/wlm/mips-tools
    # 如果之前依赖什么的都安装正确的话，最后会显示config.status: creating Makefile
    # 之后开始构建
    $ make 
    # 然后安装。这里注意，因为我是安装在自己的主目录下，所以无需root权限。
    # 如果安装在某些位置可能需要sudo make install，使用root权限安装。
    $ make install
    # --- 开始编译gcc ---
    # 切到我们为gcc准备的构建目录下
    $ cd ..
    $ cd gcc
    # 下面这句中的prefix也请根据自己的需要改
    $ ../../gcc-4.9.2/configure --target=mips-elf --prefix=/home/wlm/mips-tools --disable-shared --without-headers --with-newlib --enable-languages=c --disable-decimal-float --disable-libgomp --disable-libmudflap --disable-libssp --disable-threads --disable-multilib
    $ make
    $ make install

至此，你应该能在你安装到的位置上（我这里是mips-tools）看到（忽略里面的gxemul）:

    ~/mips-tools/bin $ ls
    gxemul              mips-elf-elfedit     mips-elf-gcov     mips-elf-ranlib
    mips-elf-addr2line  mips-elf-gcc         mips-elf-ld       mips-elf-readelf
    mips-elf-ar         mips-elf-gcc-4.9.2   mips-elf-ld.bfd   mips-elf-size
    mips-elf-as         mips-elf-gcc-ar      mips-elf-nm       mips-elf-strings
    mips-elf-c++filt    mips-elf-gcc-nm      mips-elf-objcopy  mips-elf-strip
    mips-elf-cpp        mips-elf-gcc-ranlib  mips-elf-objdump

到这里工具链就OK了。接下来我们编译gxemul

    # 直接切到gxemul源码目录中
    $ cd /home/wlm/mips-tools/gxemul-0.4.6.6
    # 配置并编译
    $ ./configure
    $ make

在编译gxemul的过程中如果出现错误，很可能是因为依赖`libX11-devel`包的原因，Debian/Ubuntu系列发行版可以通过如下命令安装：

    sudo apt-get install libX11-dev
    
RHEL/Fedora系列发行版可以执行以下命令安装：

    sudo yum install libX11-devel

然后重新运行

    $ ./configure
    $ make

之后会在这里产生一个gxemul可执行文件，将它拷贝到合适的位置即可使用。
为了方便我把它拷到了mips-tools/bin下面，和其他的工具放在一起。
至此环境就配置OK了。

为了方便，可以将这个目录加到环境变量里面，具体方法自行百度就好。
如果懒得加的话就直接用路径使用即可，例如：

    $ ~/mips-tools/bin/gxemul @r3000_test

### 关于Makefile ###

Makefile类似于大家以前接触过的VC工程。只不过不像VC那样有图形界面，
而是直接用类似脚本的方式实现的。尽管看上去很复杂，但我们实际需要关心的
只有include.mk这个文件而已。

需要改首先是CROSS_COMPILE后面的路径，根据自己的情况改成工具链所在的位置
，例如/home/wlm/mips-tools/bin。CC和LD改为gcc和ld的位置。

CFLAGS最后的-fPIC也需要删掉，这里是因为咱们的GCC版本高，-mno-abicalls的含义
发生了变化。笔者也搞不懂这里该怎么办，最终几番尝试发现去掉-fPIC可以顺利
编译通过并运行。

### 关于Linker Script ###

咱们编写的是内核，所以需要让程序最终处于内核所该在的地址。这个根据源文件
中的注释，应该查include/mmu.h中的那张内存布局图。我们让连接器将内核的地址
放到Kernel Text的地址上就OK了，具体语法参照实验指导书。

### start.s ###

根据注释这里需要初始化栈指针。根据include/mmu.h中的那张图，可以查出Kernel Stack
的地址，将sp寄存器的值改为这个地址即可。这里注意栈的增长方向问题。上学期
学mips汇编的时候，需要用到栈的地方几乎都是这样的  

    # 假设我们要用4字节的栈空间
    addi sp,sp,-4
    # 各种代码
    addi sp,sp,4
    ret
    
栈是向地址减小的方向增长的，所以，大家应该会设这个地址了吧？
    
### printf的实现 ###

这一部分个人感觉最重要的是了解printf中的格式化字符串的标准格式。这里推荐
cplusplus上的[说明](http://www.cplusplus.com/reference/cstdio/printf/)。

一个format specifier（笔者英文渣，不知道中文该怎么翻译）的原型如下：

    %[flags][width][.precision][length]specifier 
    
有了这个就好读这次实验的源代码了。按照％，flags，width,prec,length,specifier的顺序
解析即可。specifier的解析已经写好了，大家只需要填写前面的部分就行。
这里注意参考PrintNum等三个辅助函数的代码，注意理解接口，并搞清lp_Print函数中
的那几个局部变量（longFlag、negFlag等）的含义，即可顺利写出printf函数。
咱们的printf函数是简化的，不需要实现c里面printf中的完整功能就可以把这个
实验跑起来。

这一步做完了，实验一也就基本完成了，重新编译并执行`gxemul @r3000`查看结果并调试就OK了。

## 关于一些名词的补充资料 ##

这一部分主要是补充一些实验指导书中没细说的名词。由于操作系统实验采用Linux作为平台，
很多同学以前没有接触过。所以指导书中的部分名词可能会比较陌生，这里作一些补充。
重要程度不高，略看一下即可。

### Grub和Lilo ###

Grub是GNU项目的一个多操作系统启动程序。简单的说，就是可以用于在有多个操作系统的机器上，
在刚开机的时候选择一个操作系统。Lilo的功能也是一样的。如果安装过Ubuntu一类的发行版的话，
一开机出现的那个选择系统用的菜单就是GRUB。

在指导书中，列举grub仅是为了举一个实际的例子说明bootloader的stage1和stage2在实际实现中的
一些变化。

### 如何在Linux下装软件 ###

Linux不同的发行版区别比较大，但基本上可以分为使用包管理器和编译安装两种做法。
Linux下的软件多是开源软件，所以一般的发行版为了用户方便，已经将大量的软件打包，
形成了一个巨大的软件仓库。软件之间的依赖关系等信息已经由发行版的维护者写在了软件包中，
只需通过相应的软件包管理器即可直接安装。

一般这种软件包分为deb包和rpm包（还有其他的一些，但主流的是这两个）。
使用deb包的发行版有Debian、Ubuntu、Deepin等，安装命令一般为`sudo apt-get install 软件包名`，
这样包管理器就会自动去软件源里面找相应的包，自动下载并安装。
RPM包一般是Fedora、OpenSuse等发行版使用，安装命令一般为`sudo yum install 软件包名`

本次实验提供的gxemul则是源代码包，需要直接编译安装。类似于在windows上安装绿色软件。
编译gxemul包需要c++编译器，请确认自己的系统上已经安装了c++编译器（一般为g++/gcc）。
之后，执行`./configure`。这将检查你系统中编译所需的各类环境是否齐全，如果一切正常，
则会生成一个Makefile。Makefile用于构建整个程序，类似于大家熟悉的VC中的工程的概念，
但用起来一般更为灵活。执行`make`开始构建，构建完成后将会产生一个名叫gxemul的可执行文件。
这里请注意，gxemul是没有扩展名，Linux下的可执行文件一般是没有扩展名的，
这点与Windows不同（windows下一般为exe）。最后将这个可执行文件拷贝到方便使用的位置即可。

### stdarg.h是个啥 ###

stdarg.h这个头文件在printf.c等文件中被引用。据查，这个头文件主要是用于处理变长参数的问题的。
例如printf()函数的参数个数就是变长的，不一定有多少个。为了处理这样的参数，就需要这个头文件
中提供的一些函数。`va_start`可以使一个va_list指向起始参数，`va_arg`用于获取参数，`va_end`用于
回收va_list的内存空间，防止内存泄露。

## 一些坑点 ##

### gxemul 0.6难以载入配置文件 ###

实验是针对gxemul0.4版本设计的，而0.6版本中，据说配置文件部分有更新，
笔者经过多番尝试仍未能成功导入配置文件，所以强烈建议直接使用gxemul 0.4版本。

### gcc　4.9.2　make会出错 ###

笔者自行编译了一套gcc 4.9.2的工具链，但发现编译实验代码的时候会出错。
经查，原因是`-mno-abicalls`这个选项的含义发生了细微变化，和`-fPIC`发生冲突。

### Makefile中的TAB字符 ###

Makefile中有些地方是必须使用tab的。比如对于每一个target具体需要执行的命令，前面的空白是TAB，
使用空格可能会出错。有些编辑器会自动把tab替换为几个空格，所以请一定注意。如果编辑器能够显示
具体是tab还是空格是最好的。推荐sublime/emacs/vi等成熟的文本编辑器。

### Linker Script ###

Linker Script在指导书中的那段样例是直接从
[GNU LD的文档](https://www.sourceware.org/binutils/docs/ld/Simple-Example.html#Simple-Example)中拷过来的。
和Makefile一样，这里有个坑要注意。

     SECTIONS
     {
       . = 0x10000;
       .text : { *(.text) }
       . = 0x8000000;
       .data : { *(.data) }
       .bss : { *(.bss) }
     }

上面这段代码中`. = 0x10000;`等号 **两侧的空格是必须要的** ！否则会出现`syntax error`这样一个错误。
不只是这里，后面`.text : { *(.text) }`中冒号两侧的空格也一样必须有，以下类推，请一定注意。

