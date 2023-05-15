---
title: VSCode写一切——MCU篇
date: 2023-04-27 02:07:53
tags:
    - MCU
categories:
    - MCU
    - 瞎折腾
---

## Why VSCode?
首先，Keil、IAR之流虽然不是不能用，但是UI看着有一种复古的美(逃)，并且.....笔者习惯于在Linux下开发，这俩货都不支持Windows之外的任何平台，所以这俩玩意被pass了。其次，笔者早年确实折腾过CubeIDE等一系列基于Eclipse的IDE。确实能跨平台，魔改完CDT的补全触发机制之后功能也算可用。但是，eclipse的内存泄漏问题不可谓不恶名昭彰。笔者那台8G RAM的开发机，如果长时间开着eclipse + Firefox开上十几个tab，RAM就不够了...所以，eclipse系也不行。最终我把目光转向了VSCode，一番折腾之下，搞出了一套笔者自己用着挺顺手的方案。

## 依赖安装

### Linux篇
以笔者用的Linux Mint为例，首先安装gcc,make这些编译环境
```sh
sudo apt-get update
sudo apt-get install gcc-arm-none-eabi libnewlib-arm-none-eabi binutils-arm-none-eabi gdb-multiarch make cmake
```
然后安装Language Server和生成CompileDB的工具
```sh
sudo apt-get install ccls bear
```
最后安装OpenOCD:
```
sudo apt-get install openocd
```
__注意： 某些MCU可能并不被主干OpenOCD支持，需要自行编译下游OpenOCD__

### Windows篇

先安装MSYS2，在[MSYS2官网](www.msys2.org)下载安装包安装即可。然后打开mingw64环境，更新一下系统包
```sh
pacman -Syyu
```
再安装make等编译工具

```sh
pacman -S make cmake mingw-w64-x86_64-arm-none-eabi-gcc mingw-w64-x86_64-arm-none-eabi-newlib mingw-w64-x86_64-arm-none-eabi-binutils
```

然后，安装clangd和Python3

```sh
pacman -S mingw-w64-clang-x86_64-clang-tools-extra python
```

安装pip(可能需要魔法上网)
```sh
wget https://bootstrap.pypa.io/get-pip.py
python get-pip.py
```
安装compiledb:
```sh
pip3 install compiledb -i https://pypi.tuna.tsinghua.edu.cn/simple
```
### macOS篇
其实非常类似Linux。。。

```sh
curl https://bootstrap.pypa.io/get-pip.py | python3
pip3 install compiledb -i https://pypi.tuna.tsinghua.edu.cn/simple
brew install make cmake ccls
brew install --cask gcc-arm-embedded
```

似乎brew里没有OpenOCD，所以....只能自食其力了，先安装brew里有的依赖和编译环境

```sh
brew install capstone hidapi libusb
brew install pkg-config libtool autoconf automake
```

手动编译安装libjaylink

```sh
git clone https://github.com/syntacore/libjaylink.git --depth=1
cd libjaylink
./autogen.sh
./configure
make -j
sudo make install
make clean
```

编译安装OpenOCD
```sh
git clone git://git.code.sf.net/p/openocd/code openocd-code
cd openocd-code
./bootstrap
./configure
make -j 
sudo make install
mak clean
```

__不要乱删源码，卸载的时候需要makefile__


## VSCode插件安装

### Linux 和 Mac的插件列表：

| 插件                            | 功能                                             | 是否必须 | 备注                                                      |
|---------------------------------|------------------------------------------------|--------|-----------------------------------------------------------|
| Arm Assembly                    | 为ARM汇编提供高亮                                | 可选     | -                                                         |
| C/C++                           | vcFormat工具人                                   | 可选     | vcFormat配置起来比较方便，而且功能比较强大                 |
| ccls                            | LSP客户端，为VSCode提供C/Cpp语言支持              | 必须     | 比MS官方的C/C++插件提供的功能强大，并且速度更快            |
| Coretex-Debug                   | 为MCU提供调试功能                                | 必须     | 其实用C/C++那个cppdbg也能配置出来，只是极其麻烦            |
| Doxygen                         | Doxygen支持                                      | 可选     | Doxygen可以说是文档神器了2333                             |
| Doxygen Documentation Generator | Doxygen格式注释模板自动生成                      | 可选     | 再也不用手动敲doxygen模板辣                               |
| GieLens                         | 更强大的Git集成                                  | 可选     | 不用Git的可以忽略                                         |
| ident-rainbow                   | 渲染缩进时上色，便于分辨缩进                      | 可选     | -                                                         |
| Makefile Tools                  | 提供Makefile语言支持和自动配置C/C++ IntelliSense | 必须     | 如果不配置IntelliSense C/C++会给你渲染很多并不存在的Error |
| MemoryView                      | 提供内存监视功能                                 | 可选     | -                                                         |
| RTOS Views                      | 为调试RTOS提供了一些诸如task监视的功能           | 可选     | -                                                         |
| Todo Tree                       | 高亮注释中的TODO和FIXME，并且生成树状图           | 可选     | -                                                         |

### Windows插件列表

| 插件                            | 功能                                             | 是否必须 | 备注                                                      |
|---------------------------------|------------------------------------------------|--------|-----------------------------------------------------------|
| Arm Assembly                    | 为ARM汇编提供高亮                                | 可选     | -                                                         |
| C/C++                           | vcFormat工具人                                   | 可选     | vcFormat配置起来比较方便，而且功能比较强大                 |
| clangd                          | LSP客户端，为VSCode提供C/Cpp语言支持              | 必须     | 稍稍逊色于ccls，但比MS官方那坨东西强了很多                 |
| Coretex-Debug                   | 为MCU提供调试功能                                | 必须     | 其实用C/C++那个cppdbg也能配置出来，只是极其麻烦            |
| Doxygen                         | Doxygen支持                                      | 可选     | Doxygen可以说是文档神器了2333                             |
| Doxygen Documentation Generator | Doxygen格式注释模板自动生成                      | 可选     | 再也不用手动敲doxygen模板辣                               |
| GieLens                         | 更强大的Git集成                                  | 可选     | 不用Git的可以忽略                                         |
| ident-rainbow                   | 渲染缩进时上色，便于分辨缩进                      | 可选     | -                                                         |
| Makefile Tools                  | 提供Makefile语言支持和自动配置C/C++ IntelliSense | 必须     | 如果不配置IntelliSense C/C++会给你渲染很多并不存在的Error |
| MemoryView                      | 提供内存监视功能                                 | 可选     | -                                                         |
| RTOS Views                      | 为调试RTOS提供了一些诸如task监视的功能           | 可选     | -                                                         |
| Todo Tree                       | 高亮注释中的TODO和FIXME，并且生成树状图           | 可选     | -                                                         |

## Happy Coding(以STM32为例)

先生成makefile工程，just like this:

![Makefile_Project_Generate.png](https://s2.loli.net/2023/04/27/z5Tf2tRWcvFJqdp.png)

然后cd到工程目录，Linux用户执行：

```sh
bear -- make -j
```

macOS/Windows用户执行:

```sh
compiledb make -n
```

这一步是为了生成compild_commands.json，无论是clangd还是ccls都需要用它来配置includePath等工程信息。(对于使用cmake的工程直接加上-DCMAKE_EXPORT_COMPILE_COMMANDS=on即可生成，不需要外部工具)。

然后用VSCode打开工程目录，就可以Happy Coding辣。

![VSCode.png](https://s2.loli.net/2023/04/27/U1Qcgfdb8TY7sEN.png)

__注意：Windows用户需在mingw64中启动code，否则环境变量不全会导致奇奇怪怪的问题__

### Debug配置
在工作目录下建立.vscode文件夹，并在.vscode中建立tasks.json和launch.json两个文件。我的tasks.json如下

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build",
            "type": "process",
            "command": "make",
            "args": ["-j"]
        }
    ]
}
```

tasks.json可以直接抄过去，但接下来的launch.json需要根据实际情况配置，我的一个工程的launch.json如下:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "request": "launch",
            "type": "cortex-debug",
            "servertype": "openocd",
            "preLaunchTask": "build",
            "runToEntryPoint": "main",
            "gdbPath": "gdb-multiarch",
            "cwd": "${workspaceFolder}",
            "name": "Debug with OpenOCD",
            "showDevDebugOutput": "both",
            "objdumpPath": "arm-none-eabi-objdump",
            "executable": "${workspaceFolder}/build/${workspaceFolderBasename}.elf",
            "svdFile": "/media/jinyi/Jinyi/Software/CubeProgrammer/SVD/STM32F103.svd",
            "configFiles": [
                "interface/stlink.cfg",
                "target/stm32f1x.cfg"
            ],
        }
    ]
}
```

对于macOS用户，需要把gdbPath改为arm-none-eabi-gdb。如果没有对应芯片的SVD请直接删除svdFile一项。如果是自己使用CubeMX直接生成的文件夹可以不动executable，如果是从网上下载/Clone的代码，请将executable替换为你的elf文件路径。最后,configFiles需要根据自己的调试器和MCU配置，可以在/usr/local/share/openocd/scripts或者/usr/share/openocd/scripts/下找到target和interface文件夹，在里面选择自己使用的配置即可。需要注意的是configFiles中两个配置项的顺序 __不能调换__ ，否则会导致无法调试。
