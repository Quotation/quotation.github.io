---
layout: post
title:  "使用CLion做嵌入式开发"
date:   2017-01-13 13:00:00
categories: development
---

摘要：本文简要介绍了如何使用CLion做STM32上的嵌入式开发。整个开发流程不离开CLion环境，提高工作效率。



## 简介

JetBrain家的开发工具基本都是最棒的，对编程语言、框架支持得最好。CLion经过几年的发展，从无到有，成为了一个越来越完善的C++ IDE。在官方论坛上，用户呼声很高的一个需求就是remote debugging、嵌入式开发。

在最近几次更新里，CLion逐渐添加了remote debugging相关的支持。笔者经过摸索配置，终于能够在CLion中完成嵌入式开发的全套流程了。从写代码、编译，到烧写、调试，全都在CLion环境下完成，不用来回切换窗口了。虽然CLion在调试方面的功能还不够完善，但代码编辑绝对是一流的，调试的“手感”也比Eclipse舒服很多。

本文以STM32开发为例讲解开发环境的配置。理论上，其他ARM的MCU是一样的原理，可根据需要自行修改配置。



## 环境

本文所使用的软硬件工具有：

* IDE：CLion 2016.3（建议使用这个版本，稍早的版本不确定能不能用）
* toolchain / 编译器：GNU GCC for ARM (gcc-arm-none-eabi)
* Debugger：OpenOCD 0.9.0
* MCU：STM32F103，自己的板子
* 仿真器：STLink-v2



## 配置方法

### 前提条件

首先假设你已经有了一份项目代码，安装好了`gcc-arm-none-eabi` toolchain，能够用Makefile或Eclipse等方式正常编译。这部分内容不再详述。本文中，项目名为`testprj`。

### CMake配置

CLion使用CMake系统，项目文件就是`CMakeLists.txt`。

首先，创建一个公用的toolchain配置文件`toolchain-arm-eabi-gcc.cmake`，这个文件让CMake使用`gcc-arm-none-eabi` toolchain中的工具，并设置特定平台的编译参数等。文件内容如下。

```toolchain-arm-eabi-gcc.cmake```

```cmake
include(CMakeForceCompiler)
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR cortex-m3)
 
find_program(ARM_CC arm-none-eabi-gcc ${TOOLCHAIN_DIR}/bin)
find_program(ARM_CXX arm-none-eabi-g++ ${TOOLCHAIN_DIR}/bin)
find_program(ARM_OBJCOPY arm-none-eabi-objcopy ${TOOLCHAIN_DIR}/bin)
find_program(ARM_SIZE_TOOL arm-none-eabi-size ${TOOLCHAIN_DIR}/bin)

CMAKE_FORCE_C_COMPILER(${ARM_CC} GNU)
CMAKE_FORCE_CXX_COMPILER(${ARM_CXX} GNU)

set(CMAKE_ARM_FLAGS
  "-mcpu=cortex-m3 -mthumb -fno-common -ffunction-sections -fdata-sections"
)
 
if (CMAKE_SYSTEM_PROCESSOR STREQUAL "cortex-m3")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${CMAKE_ARM_FLAGS}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CMAKE_ARM_FLAGS}")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${CMAKE_ARM_FLAGS}")
else ()
  message(WARNING
    "Processor not recognised in toolchain file, "
    "compiler flags not configured."
  )
endif ()
 
# fix long strings (CMake appends semicolons)
string(REGEX REPLACE ";" " " CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")
 
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}" CACHE STRING "")
 
set(BUILD_SHARED_LIBS OFF)
```

然后，在代码根目录下创建`CMakeLists.txt`，下面只是个示例（写得也不够好）。具体写法根据项目调整。注意最后一段，需要生成出`.elf`和`.bin`文件。

```CMakeLists.txt```

```cmake
cmake_minimum_required(VERSION 3.3)
project(testprj)

enable_language(ASM)

set(BUILD_MODE DEBUG)

set(USER_C_FLAGS "-mcpu=cortex-m3 -mthumb -fmessage-length=0 -fsigned-char -ffunction-sections -fdata-sections -ffreestanding -fno-move-loop-invariants -Wall")
set(USER_CXX_FLAGS  "-std=c++11 -fabi-version=0 -fno-exceptions -fno-rtti -fno-use-cxa-atexit -fno-threadsafe-statics")
set(USER_C_DEFINES
    "-DARM_MATH_CM3 \
    -DUSE_FULL_ASSERT \
    -DTRACE \
    -DOS_USE_TRACE_SEMIHOSTING_STDOUT \
    -DSTM32F10X_HD \
    -DUSE_STDPERIPH_DRIVER \
    -DHSE_VALUE=8000000"
    )

if(BUILD_MODE STREQUAL "DEBUG")
    set(USER_C_FLAGS "${USER_C_FLAGS} -Og -g3")
    set(USER_C_DEFINES "${USER_C_DEFINES} -DDEBUG")
else()
    set(USER_C_FLAGS "${USER_C_FLAGS} -O3")
endif()

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${USER_C_FLAGS} ${USER_C_DEFINES}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${USER_C_FLAGS} ${USER_CXX_FLAGS} ${USER_C_DEFINES}")


include_directories(
    ${TOOLCHAIN_DIR}/arm-none-eabi/include
    include
    system/include
)

file(GLOB_RECURSE SOURCE_FILES
    system/*.c system/*.cpp
    user/*.c user/*.cpp
)

set(CMAKE_EXE_LINKER_FLAGS
    "${CMAKE_EXE_LINKER_FLAGS} -L\"${PROJECT_SOURCE_DIR}/ldscripts\" -T libs.ld -T mem.ld -T sections.ld -fmessage-length=0 -fsigned-char -ffreestanding -nostartfiles -specs=nano.specs -Xlinker --gc-sections -Wl,-Map=${PROJECT_NAME}.map")

add_executable(${PROJECT_NAME}.elf ${SOURCE_FILES})

set(HEX_FILE ${CMAKE_BINARY_DIR}/${PROJECT_NAME}.hex)
set(BIN_FILE ${CMAKE_BINARY_DIR}/${PROJECT_NAME}.bin)
add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -Oihex $<TARGET_FILE:${PROJECT_NAME}.elf> ${HEX_FILE}
    COMMAND ${CMAKE_OBJCOPY} -Obinary $<TARGET_FILE:${PROJECT_NAME}.elf> ${BIN_FILE}
)
```

### CLion编译设置

在CLion中打开`testprj`目录，成为一个CLion项目。

打开**Preferences > Build, Execution, Deployment > CMake**，右侧**CMake options**填入

```
-DTOOLCHAIN_DIR=<path to gcc-arm-none-eabi>
-DCMAKE_TOOLCHAIN_FILE=<path to toolchain-arm-eabi-gcc.cmake>
```

，其中两个尖括号内的内容换成实际的路径。

**Generation path**选项保留默认的`cmake-build-debug`，如果修改了，后续有的步骤也需要对应修改。

![CMake设置选项](/images/20170113/01.png)

点**OK**后，再选择菜单**Tools > CMake > Reset Cache and Reload Project**，CMake窗口会有信息输出。如果看到`Generating done`的提示就可以了。做这一步是因为修改了**CMake options**，需要重新生成CMake cache。新版本CMake会有两处warning，忽略。点击CMake窗口工具栏按钮**Open CMakeCache File**，查看`ARM_CC`等路径是否已经设置正确。

Build一下试试。如果有编译错误，修复之，直到正常编译出`.elf`和`.bin`文件。

### 调试设置

首先建立一个公共的配置文件`writeflash.expect.sh`，这是供`expect`命令使用的，功能是telnet连上OpenOCD，交互式输入烧写、halt等命令。内容如下。

（Mac系统自带了expect命令。Windows系统应该可以装ActiveState提供的[Expect for Windows](http://www.activestate.com/activetcl/expect)，没试过。）

```writeflash.expect.sh```

```
#!/usr/bin/expect
set timeout 10
spawn telnet localhost 4444
expect ">" 
send "reset init\n"
expect ">"
send "flash write_image erase [lindex $argv 0] 0x8000000\n"
expect ">"
send "reset halt\n"
expect ">"
send "arm semihosting enable\n"
expect ">"
send "exit\n"
expect off
```

然后，到CLion添加自定义工具，**Preferences > Tools > External Tools**，点“＋”号。

第一个工具是OpenOCD，**Program**和**Parameters**中分别填入`openocd`的路径和命令行参数，与用其他工具调试时保持一致。勾上“Show console when a message is printed...”。

![OpenOCD工具配置](/images/20170113/02.png)

第二个工具命名为“Write Flash”，**Program**选择`expect`命令，**Parameters**写

```
<path to writeflash.expect.sh> $ProjectFileDir$/cmake-build-debug/$ProjectName$.bin
```

，其中尖括号内的内容换成文件实际路径。

![Write Flash工具配置](/images/20170113/03.png)

然后添加调试配置项。CLion菜单**Run > Edit Configurations**，点左上角“＋”号，添加**GDB Remote Debug**项。修改名称为`testprj remote debug`。

* **GDB**选择`gcc-arm-none-eabi`目录下的`bin/arm-none-eabi-gdb`。（请注意这个选项是CLion 2016.3才有的，如果是稍早些的版本，要在**Preference > Toolchain > Debugger**里改全局debugger设置。）
* **target remote args**填`localhost:3333`。
* **Symbol file**选择编译生成的`cmake-build-debug/testprj.elf`。
* **Before launch**框点“＋”号，添加**Run Another Configuration**，选择`Build All`，这样在启动调试前会先编译项目。（也可以选择`testprj.elf`，但要保证**Executable**下拉框选中`Not selected`）。
* 再次点“+”号，选择**Run External Tool**，选择刚才添加的`Write Flash`工具。

![remote debug配置项](/images/20170113/04.png)

### 调试

首先手工启动OpenOCD，这个步骤只需要一次，后续调试时不需要。选择菜单**Tools > External Tools > OpenOCD**，OpenOCD会运行在Run窗口中，并顺利连上设备。建议在Run窗口上点右键，选择**Split Mode**，这样调试的时候可以同时看到两个工具窗口的信息。

激动人心的调试时刻到了！在代码里下个断点，选择菜单**Run > Debug... > testprj remote debug**，CLion会首先编译项目，从OpenOCD的输出信息可以看到烧写完毕，debugger连上来了。Debug窗口已经打开，代码窗口停在了断点处！

![开始调试](/images/20170113/05.png)



## 不便之处

由于IDE尚不完善，前面这些配置方法有些属于hack，不是IDE原生支持的。所以在调试过程中有些不便之处，相信CLion会持续改进的。

* 目前CLion调试不能查看内存、寄存器，不能查看汇编。我是基本不需要的，有些人的工作也许要经常用到。
* 调试时，编译失败无法停止，还会继续debug。
* 调试一次之后，由于当前configuration修改了，无法“只Build”不调试。需要手工修改configuration为Build All或testprj.elf，才能使用Build功能。（记住**Run > Edit Configurations**的快捷键会方便很多。）



## 参考

* [CLion 2016.2 EAP: Remote GDB debug](https://blog.jetbrains.com/clion/2016/07/clion-2016-2-eap-remote-gdb-debug/)
* [CLion for embedded development](https://blog.jetbrains.com/clion/2016/06/clion-for-embedded-development/)
* [Debugging Success](http://www.mazin.net/electronics/2016/07/24/debugging-success.html)
* [Linux下STM32开发环境的搭建](http://www.cnblogs.com/amanlikethis/p/3803736.html)
* [OpenOCD doc: GDB and OpenOCD](http://openocd.org/doc/html/GDB-and-OpenOCD.html)