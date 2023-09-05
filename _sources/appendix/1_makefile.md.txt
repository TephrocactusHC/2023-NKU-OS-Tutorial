# makefile 简介

makefile是一种自动构建项目的工具。在我们的ucore实现中， 我们使用了makefile进行项目的自动构建。下面我们来讨论一些关于makefile文件的基本知识。

## makefile文件的基本结构

makefile文件的基本结构是目标（target）、依赖（prerequisites）和命令（command）。每一个makefile文件的内部都包含了许多这样的生成规则。

```makefile
target ... : prerequisites ...
    command
    ...
    ...
```

这三个要件描述了一种依赖关系：当依赖比目标文件新的时候，则命令被执行。当我们在命令行输入make的时候，make程序会自动寻找目录下的makefile文件，依次检查依赖关系（以及依赖关系的依赖关系），最终按照合适的顺序执行命令，保证所需要的目标文件被生成。

## 使用宏变量

为了方便把重复的内容进行统一的管理，可以使用makefile的变量。这样，对于重复内容的修改就变成了对变量值的修改。我们一般使用大写的字符串表示变量，比如`OBJ`，`OBJECTS`等。变量定义的方法非常简单，只需要通过等号赋值即可，如下

```makefile
OBJ = a.o b.o
```

在需要使用变量的地方，我们用`$(变量名)`来使用变量。比如一段makefile代码如下所示：

```makefile
main: a.o b.o
	gcc a.o b.o -o main
```

使用了宏变量后就变成了下面这样：

```makefile
OBJ = a.o b.o
main: $(OBJ)
	gcc $(OBJ) -o main
```

如果我们需要添加一个新的`.o`文件，我们只需要修改第一行代码就可以啦~

变量也可以使用`shell`命令来定义，比如：

```makefile
cur_dir   := $(shell pwd)
```

使用这条命令会把当前所在的目录赋给变量`cur_dir`，这是也一种非常好的将shell指令和makefile结合起来的方式。

## 控制流和函数

在makefile中，可以使用`if`等方法控制makefile执行的控制流。比如下面这个例子：

```makefile
foo: $(objects)
ifeq ($(CC),gcc)
    $(CC) -o foo $(objects) $(libs_for_gcc)
else
    $(CC) -o foo $(objects) $(normal_libs)
endif
```

这里使用了`ifeq`指令判断变量`CC`是否为`gcc`，如果是则执行链接`gcc`相关的库，否则采取另外一套操作。类似的关键字还有`ifneq`（如果不相等）、`ifdef`（如果某个变量已定义）、`ifndef`（如果某个变量未定义）。

makefile中也可以使用函数。函数调用遵循下面的范式：

```makefile
$(<function> <arguments>)
```

makefile提供了许多函数，如字符处理函数，目录操作函数等等，可以帮助我们完成一些makefile中的常用操作。关于如何自定义函数的内容，感兴趣的同学可以自己去了解。

## **实验Makefile解析**

下面是对Lab中的makefile部分代码进行解析：

```makefile
PROJ	:= lab1
EMPTY	:=
SPACE	:= $(EMPTY) $(EMPTY)
SLASH	:= /
```

定义了变量PROJ，其值为lab1。
定义了变量EMPTY，其为空字符串。
定义了变量SPACE，其值为两个空格。
定义了变量SLASH，其值为斜杠。

```makefile
V       := @
```

定义了变量V，值为@。这个变量在后续的命令执行中用于控制命令是否显示在输出中。

```makefile
#ifndef GCCPREFIX
GCCPREFIX := riscv64-unknown-elf-
#endif
```

如果变量GCCPREFIX未定义，则将其设置为riscv64-unknown-elf-。这个变量用于指定编译器的前缀。

```makefile
ifndef QEMU
QEMU := qemu-system-riscv64
endif
```

如果变量QEMU未定义，则将其设置为qemu-system-riscv64。这个变量用于指定QEMU仿真器的名称。

```makefile
.SUFFIXES: .c .S .h
```

定义了文件后缀名的规则，.c、.S和.h文件被视为源文件。

```makefile
.DELETE_ON_ERROR:
```

如果在执行过程中发生错误或中断，删除目标文件。

```makefile
HOSTCC		:= gcc
HOSTCFLAGS	:= -Wall -O2
```

定义了主机编译器的名称和编译选项。

```makefile
GDB		:= $(GCCPREFIX)gdb
```

定义了调试器的名称，使用了前缀变量GCCPREFIX。

```makefile
CC		:= $(GCCPREFIX)gcc
CFLAGS  := -mcmodel=medany -std=gnu99 -Wno-unused -Werror
CFLAGS	+= -fno-builtin -Wall -O2 -nostdinc $(DEFS)
CFLAGS	+= -fno-stack-protector -ffunction-sections -fdata-sections
CTYPE	:= c S
```

定义了编译器的名称和编译选项。
CFLAGS包含了一系列编译选项，如代码模型、标准、警告设置等。
CTYPE定义了源文件的类型。

```makefile
LD      := $(GCCPREFIX)ld
LDFLAGS	:= -m elf64lriscv
LDFLAGS	+= -nostdlib --gc-sections
```

定义了链接器的名称和链接选项。

```makefile
OBJCOPY := $(GCCPREFIX)objcopy
OBJDUMP := $(GCCPREFIX)objdump
```

定义了目标文件复制和转储的工具名称。

```makefile
COPY	:= cp
MKDIR   := mkdir -p
MV		:= mv
RM		:= rm -f
AWK		:= awk
SED		:= sed
SH		:= sh
TR		:= tr
TOUCH	:= touch -c
```

定义了文件复制、目录创建、文件移动、文件删除等命令的名称。

```makefile
OBJDIR	:= obj
BINDIR	:= bin
```

定义了目标文件和二进制文件的存放目录。

```makefile
ALLOBJS	:=
ALLDEPS	:=
TARGETS	:=
```

定义了用于存放所有目标文件、依赖文件和目标的变量。

```makefile
include tools/function.mk
```

包含了tools/function.mk文件，该文件包含了makefile中使用的函数定义。

```makefile
listf_cc = $(call listf,$(1),$(CTYPE))
```

定义了一个函数宏listf_cc，用于列出指定目录中指定类型的文件。

```makefile
add_files_cc = $(call add_files,$(1),$(CC),$(CFLAGS) $(3),$(2),$(4))
create_target_cc = $(call create_target,$(1),$(2),$(3),$(CC),$(CFLAGS))
```

定义了两个函数宏add_files_cc和create_target_cc，用于添加文件和创建目标。

```makefile
# for hostcc
add_files_host = $(call add_files,$(1),$(HOSTCC),$(HOSTCFLAGS),$(2),$(3))
create_target_host = $(call create_target,$(1),$(2),$(3),$(HOSTCC),$(HOSTCFLAGS))
```

这段代码继续定义了一些函数和变量，与主机编译器相关。

`add_files_host`函数：它是在之前定义的`add_files`函数的基础上进行了扩展。`add_files_host`函数用于添加主机编译器编译的文件。该函数将使用主机编译器和选项编译源文件，并将生成的目标文件添加到文件列表中。

create_target_host`函数：它是在之前定义的`create_target`函数的基础上进行了扩展。`create_target_host`函数用于创建主机编译的目标。该函数将使用主机编译器和选项创建目标文件。
这些函数和变量与主机编译器相关，用于在构建过程中使用主机编译器编译文件。它们提供了用于添加主机编译文件和创建主机编译目标的功能。

```makefile
cgtype = $(patsubst %.$(2),%.$(3),$(1))

objfile = $(call toobj,$(1))

asmfile = $(call cgtype,$(call toobj,$(1)),o,asm)

outfile = $(call cgtype,$(call toobj,$(1)),o,out)

symfile = $(call cgtype,$(call toobj,$(1)),o,sym)

\# for match pattern

match = $(shell echo $(2) | $(AWK) '{for(i=1;i<=NF;i++){if(match("$(1)","^"$$(i)"$$")){exit 1;}}}'; echo $$?)
```

这段代码定义了一些函数和变量，用于处理文件名和模式匹配。

`cgtype`函数：它使用`patsubst`函数将文件名中的后缀名替换为指定的新后缀名。

`objfile`函数：它调用`toobj`函数将文件名转换为目标文件名（以`.o`结尾）。

`asmfile`、`outfile`和`symfile`函数：它们使用`cgtype`函数来生成特定类型的文件名。它们接受一个参数：`$(1)`代表文件名。`asmfile`用于生成汇编文件名（以`.asm`结尾），`outfile`用于生成输出文件名（以`.out`结尾），`symfile`用于生成符号文件名（以`.sym`结尾）。

`match`函数：它使用`awk`命令来进行模式匹配。它接受两个参数：`$(1)`代表模式，`$(2)`代表要匹配的字符串。它返回一个值，如果匹配成功则为0，否则为1。该函数用于判断目标是否与指定的模式匹配。

```makefile
# create kernel target
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
```

这段代码用于构建内核（kernel）目标文件。首先，它使用`$(call totarget,kernel)`宏来将目标名称转换为目标文件名。然后，它定义了一个规则，指定了目标文件（$(kernel)）的依赖关系和构建命令。

依赖关系：

1. `tools/kernel.ld`：作为目标文件的一个依赖项，表示该文件需要先于目标文件构建完成。
1. `$(KOBJS)`：这是一个变量，表示内核目标文件的一组依赖项。具体的依赖项可能在Makefile的其他地方定义。

构建命令：

1. `@echo + ld $@`：打印构建命令，其中$@表示目标文件名。
1. `$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)`：这是实际的构建命令。它使用LD变量指定的链接器（可能是GNU ld）来链接目标文件。LDFLAGS变量包含了链接器的一些选项。-T选项指定链接脚本文件为`tools/kernel.ld`，-o选项指定输出文件为目标文件名，$(KOBJS)表示依赖项的列表。
1. `@$(OBJDUMP) -S $@ > $(call asmfile,kernel)`：使用OBJDUMP变量指定的反汇编工具将目标文件反汇编，并将结果重定向到一个文件中，该文件由`$(call asmfile,kernel)`宏指定。
1. `@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)`：使用OBJDUMP变量指定的工具获取目标文件的符号表，并使用SED变量指定的工具对符号表进行一些处理，最后将结果重定向到一个文件中，该文件由`$(call symfile,kernel)`宏指定。

```makefile
# create ucore.img
UCOREIMG	:= $(call totarget,ucore.img)


$(UCOREIMG): $(kernel)
	$(OBJCOPY) $(kernel) --strip-all -O binary $@

$(call create_target,ucore.img)
```

这段代码用于创建`ucore.img`文件。首先，它使用`$(call totarget,ucore.img)`宏来将目标名称转换为目标文件名。这个宏可能在Makefile的其他地方定义。然后，它定义了一个规则，指定了`ucore.img`目标文件的依赖关系和构建命令。

依赖关系：

1. `$(kernel)`：作为目标文件的一个依赖项，表示该文件需要先于目标文件构建完成。

构建命令：

1. `$(OBJCOPY) $(kernel) --strip-all -O binary $@`：这是实际的构建命令。它使用OBJCOPY变量指定的工具，将`$(kernel)`目标文件进行处理。`--strip-all`选项表示移除所有符号表和调试信息，`-O binary`选项表示将目标文件输出为二进制文件。最后，`$@`表示目标文件名，即`ucore.img`。

```makefile
.PHONY: qemu 
qemu: $(UCOREIMG) $(SWAPIMG) $(SFSIMG)
#	$(V)$(QEMU) -kernel $(UCOREIMG) -nographic
	$(V)$(QEMU) \
		-machine virt \
		-nographic \
		-bios default \
		-device loader,file=$(UCOREIMG),addr=0x80200000
```

这段代码就是我们启动qemu的命令，这段代码首先通过宏定义$(UCOREIMG) $(SWAPIMG) $(SFSIMG)的函数进行目标文件的构建，然后使用qemu语句进行qemu启动加载内核。其中：

- -machine virt 表示将模拟的 64 位 RISC-V 计算机设置为名为 virt 的通用虚拟平台。
- -nographic 表示模拟器不需要提供图形界面，而只需要对外输出字符流。

- 通过 -bios 可以设置 Qemu 模拟器开机时用来初始化的引导加载程序（bootloader），这里我们使用默认的Opensbi

- 通过 -device loader 可以在 Qemu 模拟器开机之前将一个宿主机上的文件载入到 Qemu 的物理内存的指定位置中， file 和 addr 分别可以设置待载入文件的路径以及将文件载入到的 Qemu 物理内存上的物理地址。

## 参考资料

[跟我一起写makefile](https://seisman.github.io/how-to-write-makefile/index.html)



