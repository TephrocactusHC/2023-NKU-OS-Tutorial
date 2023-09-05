### 调试工具介绍

#### gdb使用

gdb 是功能强大的调试程序，可完成如下的调试任务：

- 设置断点
- 监视程序变量的值
- 程序的单步(step in/step over)执行
- 显示/修改变量的值
- 显示/修改寄存器
- 查看程序的堆栈情况
- 远程调试
- 调试线程

在可以使用 gdb 调试程序之前，必须使用 -g 或 –ggdb编译选项编译源文件。运行 gdb 调试程序时通常使用如下的命令：

	gdb progname

在 gdb 提示符处键入help，将列出命令的分类，主要的分类有：

- aliases：命令别名
- breakpoints：断点定义；
- data：数据查看；
- files：指定并查看文件；
- internals：维护命令；
- running：程序执行；
- stack：调用栈查看；
- status：状态查看；
- tracepoints：跟踪程序执行。

键入 help 后跟命令的分类名，可获得该类命令的详细清单。gdb的常用命令如下表所示。

表 gdb 的常用命令

<table>
<tr><td>break FILENAME:NUM</td><td>在特定源文件特定行上设置断点</td></tr>
<tr><td>clear FILENAME:NUM</td><td>删除设置在特定源文件特定行上的断点</td></tr>
<tr><td>run</td><td>运行调试程序</td></tr>
<tr><td>step</td><td>单步执行调试程序，不会直接执行函数</td></tr>
<tr><td>next</td><td>单步执行调试程序，会直接执行函数</td></tr>
<tr><td>backtrace</td><td>显示所有的调用栈帧。该命令可用来显示函数的调用顺序</td></tr>
<tr><td>where continue</td><td>继续执行正在调试的程序</td></tr>
<tr><td>display EXPR</td><td>每次程序停止后显示表达式的值,表达式由程序定义的变量组成</td></tr>
<tr><td>file FILENAME</td><td>装载指定的可执行文件进行调试</td></tr>
<tr><td>help CMDNAME</td><td>显示指定调试命令的帮助信息</td></tr>
<tr><td>info break</td><td>显示当前断点列表，包括到达断点处的次数等</td></tr>
<tr><td>info files</td><td>显示被调试文件的详细信息</td></tr>
<tr><td>info func</td><td>显示被调试程序的所有函数名称</td></tr>
<tr><td>info prog</td><td>显示被调试程序的执行状态</td></tr>
<tr><td>info local</td><td>显示被调试程序当前函数中的局部变量信息</td></tr>
<tr><td>info var</td><td>显示被调试程序的所有全局和静态变量名称</td></tr>
<tr><td>kill</td><td>终止正在被调试的程序</td></tr>
<tr><td>list</td><td>显示被调试程序的源代码</td></tr>
<tr><td>quit</td><td>退出 gdb</td></tr>
</table>

用gdb查看源代码可以用list命令，但是这个不够灵活。可以使用"layout src"命令，或者按Ctrl-X再按A，就会出现一个窗口可以查看源代码。也可以用使用-tui参数，这样进入gdb里面后就能直接打开代码查看窗口。其他代码窗口相关命令：

<table>
<tr><td>info win</td><td>显示窗口的大小</td></tr>
<tr><td>layout next</td><td>切换到下一个布局模式</td></tr>
<tr><td>layout prev</td><td>切换到上一个布局模式</td></tr>
<tr><td>layout src</td><td>只显示源代码</td></tr>
<tr><td>layout asm</td><td>只显示汇编代码</td></tr>
<tr><td>layout split</td><td>显示源代码和汇编代码</td></tr>
<tr><td>layout regs</td><td>增加寄存器内容显示</td></tr>
<tr><td>focus cmd/src/asm/regs/next/prev</td><td>切换当前窗口</td></tr>
<tr><td>refresh</td><td>刷新所有窗口</td></tr>
<tr><td>tui reg next</td><td>显示下一组寄存器</td></tr>
<tr><td>tui reg system</td><td>显示系统寄存器</td></tr>
<tr><td>update</td><td>更新源代码窗口和当前执行点</td></tr>
<tr><td>winheight name +/- line</td><td>调整name窗口的高度</td></tr>
<tr><td>tabset nchar</td><td>设置tab为nchar个字符</td></tr>
</table>

#### 结合gdb和qemu源码级调试ucore

##### 编译可调试的目标文件

为了使得编译出来的代码是能够被gdb这样的调试器调试，我们需要在使用gcc编译源文件的时候添加参数："-g"。这样编译出来的目标文件中才会包含可以用于调试器进行调试的相关符号信息。

##### 使用远程调试

为了与qemu配合进行源代码级别的调试，需要先让qemu进入等待gdb调试器的接入并且还不能让qemu中的CPU执行，因此启动qemu的时候，我们需要使用参数-S –s这两个参数来做到这一点。-s 可以使 Qemu 监听本地 TCP 端口 1234 等待 GDB 客户端连接，而 -S 可以使 Qemu 在收到 GDB 的请求后再开始运行。因此，Qemu 暂时没有任何输出。

```makefile
	$qemu-system-riscv64 \
		-machine virt \
		-nographic \
		-bios default \
		-device loader,file=$(UCOREIMG),addr=0x80200000\
		-s -S
```

在使用了前面提到的参数启动qemu之后，qemu中的CPU并不会马上开始执行，这时我们启动gdb，然后在gdb命令行界面下，使用下面的命令连接到qemu：

```makefile
	riscv64-unknown-elf-gdb \
    -ex 'file bin/kernel' \
    -ex 'set arch riscv:rv64' \
    -ex 'target remote localhost:1234'
```

riscv64-unknown-elf-gdb: 这是 GDB 调试器的可执行文件名。它用于启动 GDB 调试器。

-ex 'file bin/kernel': 这个选项告诉 GDB 加载名为 bin/kernel 的目标文件。目标文件通常是编译后的可执行文件或库文件，这里指定的是一个名为 kernel 的文件。

-ex 'set arch riscv:rv64': 这个选项设置 GDB 的目标体系结构为 RISC-V 的 64 位版本（rv64）。这样，GDB 将使用 RISC-V 体系结构的调试器特性。

-ex 'target remote localhost:1234': 这个选项告诉 GDB 连接到本地主机的端口 1234，GDB 将尝试通过与本地主机的端口 1234 建立连接来远程调试该目标文件。