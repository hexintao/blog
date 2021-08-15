闲来无事，照着《深入解析 Mac OS X 操作系统》这本书的例子，探索了一下动态库的加载过程，身为一名 iOS 开发，启动优化这个绕不开的话题中总是会提到这么一项，动态库加载究竟做了些什么工作呢？

这是一段非常简单的 C 语言程序。

``` c
// a.c 文件
#include <stdio.h>
#include <unistd.h>

int main (int argc, char **argv) {
    printf("Hello, xt");
    printf("world");
    return 0;
}
```
程序里我们使用到了 `printf`，它的定义肯定没在我们的 `a.c` 中，所以这是一个外部符号，需要链接器在程序启动的时候进行动态解析，我们就以它来看看动态库的加载过程。

首先，我们需要把这段程序编译成可执行程序 a，这里别忘了 -g 参数，它的作用就是在可执行文件中生成调试信息，否则后边调试会无法下断点：

``` bash
cc a.c -g -o a
```

我们先看下 `printf` 的调用，经过汇编以后，会变成什么样子？

```
otool -p _main -tV a
```

执行上述命令以后，输出如下信息：

```
a:
(__TEXT,__text) section
_main:
0000000100003f40	pushq	%rbp
0000000100003f41	movq	%rsp, %rbp
0000000100003f44	subq	$0x20, %rsp
0000000100003f48	movl	$0x0, -0x4(%rbp)
0000000100003f4f	movl	%edi, -0x8(%rbp)
0000000100003f52	movq	%rsi, -0x10(%rbp)
0000000100003f56	leaq	0x45(%rip), %rdi        ## literal pool for: "Hello, xt"
0000000100003f5d	movb	$0x0, %al
0000000100003f5f	callq	0x100003f82             ## symbol stub for: _printf
0000000100003f64	leaq	0x41(%rip), %rdi        ## literal pool for: "world"
0000000100003f6b	movl	%eax, -0x14(%rbp)
0000000100003f6e	movb	$0x0, %al
0000000100003f70	callq	0x100003f82             ## symbol stub for: _printf
0000000100003f75	xorl	%ecx, %ecx
0000000100003f77	movl	%eax, -0x18(%rbp)
0000000100003f7a	movl	%ecx, %eax
0000000100003f7c	addq	$0x20, %rsp
0000000100003f80	popq	%rbp
0000000100003f81	retq
```
我们可以看到，两次 `printf` 的调用，都转化成了 `callq` 跳转指令，且地址都是 `0x100003f82`，注释中也提到了 `symbol stub for: _printf`，`printf` 符号的"桩"信息。

所以，外部符号的调用，编译器会打上"桩"信息，以此表明该符号需要动态解析。我们再看下程序的内存分配信息：

```
otool -l -V a
```

这次输出内容比较多，只贴出来部分：

```
...
Section
  sectname __stubs
   segname __TEXT
      addr 0x0000000100003f82
      size 0x0000000000000006
    offset 16258
     align 2^1 (2)
    reloff 0
    nreloc 0
      type S_SYMBOL_STUBS
attributes PURE_INSTRUCTIONS SOME_INSTRUCTIONS
 reserved1 0 (index into indirect symbol table)
 reserved2 6 (size of stubs)
Section
  sectname __stub_helper
   segname __TEXT
      addr 0x0000000100003f88
      size 0x000000000000001a
    offset 16264
     align 2^2 (4)
    reloff 0
    nreloc 0
      type S_REGULAR
attributes PURE_INSTRUCTIONS SOME_INSTRUCTIONS
 reserved1 0
 reserved2 0
...
```

我们会发现：

1. 有外部符号调用的程序，二进制文件中会存在 `__stubs` 的段。
2. '__stubs' 部分的地址 `addr` 和刚才汇编代码中的地址是一样的。

然后我们就把程序运行起来：

```
lldb a
```
这时命令行就会进入 `lldb` 的调试状态，当然此时程序 `a` 还没开始执行，我们再输入 `r`，接着就会在命令行中看到：

```
(lldb) r
Process 5039 launched: '/Users/hexintao/Desktop/Demo/C/a' (x86_64)
Hello, xtworldProcess 5039 exited with status = 0 (0x00000000)
```
程序已经执行完毕，我们的 `Hello, xtworld` 已经打印出来了。先运行下，再调试，系统会自动加一些方便我们看的调试信息。

我们看下桩的内存地址处的指令信息：

```
(lldb) x/2i 0x100003f82
0x100003f82: ff 25 78 40 00 00  jmpq   *0x4078(%rip)             ; (void *)0x0000000100003f98
0x100003f88: 00 00              addb   %al, (%rax)
```
可以看到执行了一个跳转 `jmpq` 到 `0x0000000100003f98`，我们继续看目标地址处的指令信息：

```
(lldb) x/2i 0x0000000100003f98
0x100003f98: 68 00 00 00 00  pushq  $0x0
0x100003f9d: e9 e6 ff ff ff  jmp    0x100003f88
warning: Not all bytes (10/30) were able to be read from 0x100003f98.
```

又一个跳转，去了 `0x100003f88`，继续打：

```
(lldb) x/4i 0x100003f88
0x100003f88: 4c 8d 1d 79 40 00 00  leaq   0x4079(%rip), %r11        ; _dyld_private
0x100003f8f: 41 53                 pushq  %r11
0x100003f91: ff 25 69 00 00 00     jmpq   *0x69(%rip)               ; (void *)0x0000000000000000
0x100003f97: 90                    nop
```

终于，我们发现了一点特殊的内容，出现了 `_dyld_private` 字样，后边又有一个跳转，但是跳转的内容写的是 `0x0000000000000000`，怎么会跳转到空呢？是不是出了什么 `bug`？

当然不是 `bug` 了，是因为我们的程序已经执行结束了，我们在 `main` 方法下个断点：

```
(lldb) b main
Breakpoint 1: where = a`main + 22 at a.c:5:5, address = 0x0000000100003f56
```

然后输入 `r` 重新运行程序，接着程序就会进入断点：

```
(lldb) r
Process 5087 launched: '/Users/hexintao/Desktop/Demo/C/a' (x86_64)
Process 5087 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000100003f56 a`main(argc=1, argv=0x00007ffeefbff6c8) at a.c:5:5
   2   	#include <unistd.h>
   3
   4   	int main (int argc, char **argv) {
-> 5   	    printf("Hello, xt");
   6   	    printf("world");
   7   	    return 0;
   8   	}
Target 0: (a) stopped.
```

这时候，我们再看下上述内存地址的内容，这次可以不用一直跳转了，直接查看最后一次地址的内容即可：

```
(lldb) x/4i 0x100003f88
    0x100003f88: 4c 8d 1d 79 40 00 00  leaq   0x4079(%rip), %r11        ; _dyld_private
    0x100003f8f: 41 53                 pushq  %r11
    0x100003f91: ff 25 69 00 00 00     jmpq   *0x69(%rip)               ; (void *)0x00007fff70369578: dyld_stub_binder
    0x100003f97: 90                    nop
```

之前 `0x00` 的地址变成了 `0x00007fff70369578`，并且系统标明了是 `dyld_stub_binder`，这个函数的作用就是进行符号绑定，执行完成之后，`printf` 符号就会有一个确定的地址，剩下的程序直接跳转执行就行了。

这里边还有一点，符号绑定的工作其实比较复杂，所以系统会对符号绑定的结果进行缓存，我们可以看下这个缓存的结果，思路很简单，我们先在第一个 `printf` 处打个断点，看下 `call` 目标地址的指令，然后在第二个 `printf` 处再打个断点，如果有缓存的话，第二次 `printf` 的时候，直接跳转到 `printf` 的地址即可。这里的地址，还是汇编代码里边程序执行的地址。

```
(lldb) br s -a 0000000100003f5f
Breakpoint 2: where = a`main + 31 at a.c:5:5, address = 0x0000000100003f5f
(lldb) br s -a 0000000100003f70
Breakpoint 3: where = a`main + 48 at a.c:6:5, address = 0x0000000100003f70
```

当程序停到第一个断点的时候，程序在执行动态绑定的流程：

```
(lldb) x/2i 0000000100003f5f
    0x100003f5f: e8 1e 00 00 00        callq  0x100003f82               ; symbol stub for: printf
    0x100003f64: 48 8d 3d 41 00 00 00  leaq   0x41(%rip), %rdi          ; "world"
(lldb) x/4i 0x100003f82
    0x100003f82: ff 25 78 40 00 00     jmpq   *0x4078(%rip)             ; (void *)0x0000000100003f98
    0x100003f88: 4c 8d 1d 79 40 00 00  leaq   0x4079(%rip), %r11        ; _dyld_private
    0x100003f8f: 41 53                 pushq  %r11
    0x100003f91: ff 25 69 00 00 00     jmpq   *0x69(%rip)               ; (void *)0x00007fff70369578: dyld_stub_binder
```

接着 `c` 继续执行，来到第二个断点处：

```
(lldb) x/2i 0000000100003f70
->  0x100003f70: e8 0d 00 00 00  callq  0x100003f82               ; symbol stub for: printf
    0x100003f75: 31 c9           xorl   %ecx, %ecx
(lldb) x/4i 0x100003f82
    0x100003f82: ff 25 78 40 00 00     jmpq   *0x4078(%rip)             ; (void *)0x00007fff703f9370: printf
    0x100003f88: 4c 8d 1d 79 40 00 00  leaq   0x4079(%rip), %r11        ; _dyld_private
    0x100003f8f: 41 53                 pushq  %r11
    0x100003f91: ff 25 69 00 00 00     jmpq   *0x69(%rip)               ; (void *)0x00007fff70369578: dyld_stub_binder
```

我们可以看到，`0x100003f82` 处虽然都是 `jmpq   *0x4078(%rip)`，但是第二次断点时，地址内容已经变成了 `0x00007fff703f9370: printf`。


