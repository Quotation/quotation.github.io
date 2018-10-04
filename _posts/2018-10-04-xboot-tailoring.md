---
layout: post
title:  "XBOOT尺寸裁剪"
date:   2018-10-04 21:00:00
categories: development
---

## 介绍

* 硬件平台：全志F1C100s
* 系统：[XBOOT](https://github.com/xboot/xboot)（是个集成了驱动和库的bootloader，没有OS）
* 工具链：gcc-arm-none-eabi

XBOOT默认的配置带了很多功能，编译后的尺寸颇大，影响启动时间。因为XBOOT启动过程的大部分时间是花在从Flash拷贝代码到RAM，所以只要能把编译结果的尺寸降下来，就可以成比例地加快启动速度。

在对XBOOT做裁剪优化后，编译后的尺寸从默认的4MB降至<170kB（不带图形系统和Lua），启动速度完全满足要求。

<br/>
<hr/><br/>

## 裁剪

未优化前，默认编译结果`xboot.bin`尺寸为4MB。

以下的裁剪措施分若干个步骤，可根据自己的需要的功能，只采取部分步骤。

1. 去除不用的资源文件，尺寸减到3.3MB。XBOOT带了字体、图片、示例Lua脚本等资源文件，如果完全用不着图形界面，直接删掉。**（如果应用中确实需要用到大量资源，参见本文最后给出的优化方法。）**

    步骤：
    * 删除`src/romdisk`目录；
    * 修改`src/Makefile`，去除`$(CP) romdisk .obj`这行，但是保留`arch/`下的`romdisk`。
    * 

2. 去除Lua框架，尺寸减到3MB。快速开发点简单的东西用Lua比较方便，由于我们的产品是用C/C++开发，所以Lua完全用不着。

    步骤：
    * 修改`src/Makefile`，去除所有`lua`、`framework`相关的行；
    * 编译一下，根据编译错误，删除代码里相关的引用。
    * 
    
3. 去除图形库，尺寸减到403kB。图形图像相关的库比较多，可根据需要裁剪。很多应用场景不需要矢量图形、矢量字体，只需要libpng再加上位图字库就够用。
    步骤：
    * 修改`src/Makefile`，去除`zlib`、`libpng`、`pixman`、`cairo`、`freetype`、`chipmunk`相关的行；
    * 编译一下，根据编译错误删除代码里引用到图形库的地方（比如`do_showlogo`）。
    * 

4. 去除shell、command功能，尺寸减到371kB。开发、调试时可以用用shell，正式产品不需要保留。

    步骤：
    * 修改`src/Makefile`，去除`shell`、`command`相关的行；
    * 删除`src/arch/arm32/lib/cpu/cmd-*.c`，或移到另外的目录里；
    * 删除掉`main()`中的`do_showlogo`、`do_autoboot`、`run_shell`调用，改为直接调用实际的应用代码入口；
    * 编译一下，根据编译错误删除代码里几处引用到`system()`的地方。
    * 

5. 去除不用的驱动，尺寸减到308kB。根据自己的应用需求，删除不必要的驱动。
    
    步骤：
    * 修改`src/Makefile`，去除用不到的`driver/xxx`，我这里只保留了`block`、时钟、`console`、`dma`、`gpio`、`i2c`、`interrupt`、`pwm`、`reset`、`spi`、`uart`、`watchdog`等常用外设驱动；
    * 删除`mach-f1c100s/driver`里不用的驱动，或移到单独的目录里。
    * 

6. 去除不用的库，尺寸减到277kB。

    步骤：
    * 修改`src/Makefile`，去除`libc/charset`、`libc/crypto`、`fs/xfs`（视自己的需要删除）；
    删除代码里引用到的地方。
    * 

7. 链接时去除未用到的符号，尺寸减到170kB。XBOOT自带的`libc`和`libm`函数都通过`EXPORT_SYMBOL`宏加到了`.ksymtab.text`段，所以即使在代码中没有调用到，链接时也无法优化掉，白白占用空间。这个设计本意是为了编译动态链接库，不过在当前的XBOOT中并没有使用。

    步骤：
    * 修改`module.h`，把`EXPORT_SYMBOL`宏定义为空，即：`#define EXPORT_SYMBOL(symbol)`；
    * 修改`mach-f1c100s/xboot.mk`，给`LDFLAGS`增加`-Wl,--gc-sections`，给`MCFLAGS`增加`-ffunction-sections -fdata-sections`，让链接器删除未用到的符号。
    * 

8. 去除文件系统。鉴于这个改动涉及很多零散的点，且大多数应用都会需要文件系统，所以不建议这样改。如果实在有必要，可以把所有的`fs`和`block`驱动去掉，尺寸减到131kB。

以上为功能裁剪和链接优化方法。

<br/>
<hr/><br/>

## 资源打包
如果程序中需要用到大量资源，资源文件默认是放在romdisk中，会增加启动时间。一个可采取的优化措施是，把非必要的资源单独放在另一个romdisk中，在启动后异步加载。（还未实际验证过）

先分析XBOOT的代码，和romdisk相关的有如下几处：

* `src/Makefile`最后，`sinclude`这句，把源码的某个目录拷贝到`.obj`目录中，用`cpio`命令把目录打包成单个文件；
* `driver/block/romdisk/data.S`和`mach-f1c100s/xboot.ld`，把打包后的romdisk文件链接到单独的`.romdisk`段；
* 启动时，`sys_copyself`中把`__image_start`和`__image_end`两个符号之间的内容拷贝到RAM，如上所说`.romdisk`段也在此范围内；
* `subsys_init_romdisk`中，用`__romdisk_start`和`__romdisk_end`符号之间的内存创建romdisk，并在`subsys_init_rootfs`中挂载到`/`。

因此，优化方法是，将非必须（启动不需要）的资源文件打包到另一个romdisk，挂载到单独的目录下使用。大致做法如下：

* 启动过程中必须的资源仍然放在默认romdisk中；
* 启动非必须的资源单独放到一个源码目录，通过`Makefile` cpio打包成单个文件；
* `xboot.ld`中添加`.dataromdisk`段，参照XBOOT的做法，用一个`.S`文件把上一步的打包文件链接到`.dataromdisk`段；该段定义在`.data`段之后，因此启动的时候不会被`sys_copyself`拷贝；
* 系统启动之后，参照`sys_copyself`的方法，把`.dataromdisk`段拷到RAM；
* 按照`subsys_init_romdisk`和`subsys_init_rootfs`的做法，初始化另一个romdisk，挂载到`/data`。
