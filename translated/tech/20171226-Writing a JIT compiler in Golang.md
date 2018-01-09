
# Writing a JIT compiler in Golang
摘要：本文在 golang 中实现一个简单的 JIT 编译器，文章结尾有完整的示例代码。

A JIT compiler (Just-in-Time) is any program that runs machine code generated during runtime. The difference between JIT code and other code (eg. fmt.Println) is that the JIT code is generated at runtime.
![avatar](https://cdn-images-1.medium.com/max/1600/1*HoGqcpUrMoOujgv39YMYSQ.png)
Go 语言是静态类型的语言，需要预先编译才能运行。
Programs written in Golang are statically typed, and compiled ahead of time. It might seem impossible to generate arbitrary code, let alone execute said code. However, it is possible to emit instructions into a running go process. This is done using Type Magic — the ability to convert any type to any other type.
As a side note, If you’re interested in hearing more about Type Magic, please leave a comment below and I’ll write about it next.

## 本例代码适用于 x64 处理器
机器码是指对编译器有特殊意义的一系列字节码。笔者使用的是 x64
 处理器，因此在本例中会使用 x64 的指令集。这意味着这些代码也只能在 x64 的处理器上才能运行。

## 生成x64的输出“Hello World!”
为了打印“Hello World”，我们需要使用 syscall 函数中的 [`write（int fd，const void * buf，size_t count）`](http://man7.org/linux/man-pages/man2/write.2.html) 来命令处理器输出数据。
`write` 函数的首个参数 `int fd` 用一个文件描述符fd来代表要写入的位置。将输出打印到控制台是通过标准文件描述符 `stdout` 实现的。 而 `stdout` 对应的fd编码是 `1`.
第二个参数代表数据的位置，在下一节中，我们会详细阐述它。
第三个参数 `count` 代表要写入的字节数。以这里的“Hello World!”为例，要写入的字节数是 12。为了调用syscall函数，这三个参数都需要被保存在特定的寄存器中，如下表所示：
```
+----------+--------+--------+--------+--------+--------+--------+
| Syscall #| Param 1| Param 2| Param 3| Param 4| Param 5| Param 6|    
+----------+--------+--------+--------+--------+--------+--------+
| rax      |  rdi   |  rsi   |   rdx  |   r10  |   r8   |   r9   |    
+----------+--------+--------+--------+--------+--------+--------+
```

下面给出用来初始化寄存器的字节码序列：

```
0:  48 c7 c0 01 00 00 00    mov    rax,0x1 
7:  48 c7 c7 01 00 00 00    mov    rdi,0x1
e:  48 c7 c2 0c 00 00 00    mov    rdx,0xc
```

第一条指令将 `rax` 设置为 `1` - 表示调用 `write` 这一syscall函数。
第二条指令将 `rdi` 设置为 `1` - 表示stdout的文件描述符。
第三条指令将 `rdx` 设置为 `12` - 来表示要打印的字节数。
这里没有对数据位置的参数进行设置，实际的写入调用也是如此。
为了能够指定包含“Hello World！”的数据的位置，数据需要首先被存储在内存中的某个位置。

In order to specify the location of data containing “Hello World!”, the data needs to have a location first — i.e. it needs to be stored somewhere in memory.
代表“Hello World!”数据的字节码序列是 `48 65 6c 6c 6f 20 57 6f 72 6c 64 21`。它应该存储在一个处理器不会处理到它的位置，否则程序会抛出一个 segmentation fault error。
在本例中，数据可以存储在可执行指令的末尾，即 `return` 指令之后。 在 `return` 指令之后存储数据是安全的，因为处理器在遇到 `return` 时会转跳到不同的地址，并且不会再顺序执行下去。
由于直到返回指令被布置之前，地址过去的返回是未知的，所以可以使用它的临时占位符，并且一旦数据的地址已知就用正确的地址替换。 这是连接器所遵循的确切程序。 链接过程中只需填写这些地址，以指向正确的数据或功能。
In this case, the data can be stored at the end of the executable instructions — i.e. after a return instruction. It is safe to store data after the return instruction because the processor "jumps" to a different address on encountering return and will not execute sequentially anymore.
Since the address past return is not known until the return instruction is laid out, a temporary place holder for it can be used and then replaced with the correct address once the address of the data is known. This is the exact procedure followed by linkers. The process of linking simply fills out these addresses to point to the correct data or function.
```
15: 48 8d 35 00 00 00 00    lea    rsi,[rip+0x0]      # 0x15
1c: 0f 05                   syscall
1e: c3                      ret
```

上面的代码中，加载“Hello World!”数据地址的 `lea` 指令指向它自身 (rip+0x0)。这是因为数据还没被存储，数据的地址也就是未知的。
字节码 `0F 05` 代表 syscall 的调用。
而后可以看到 `return` 指令被安排在了下一行，那么现在就可以载入数据了。
```
1f: 48 65 6c 6c 6f 20 57 6f 72 6c 64 21   // Hello World!
```

随着整个程序的运行，我们现在可以更新 `lea` 指令，让其指向数据地址，更新后的代码如下：

```
0:  48 c7 c0 01 00 00 00    mov    rax,0x1
7:  48 c7 c7 01 00 00 00    mov    rdi,0x1
e:  48 c7 c2 0c 00 00 00    mov    rdx,0xc
15: 48 8d 35 03 00 00 00    lea    rsi,[rip+0x3]        # 0x1f
1c: 0f 05                   syscall
1e: c3                      ret
1f: 48 65 6c 6c 6f 20 57 6f 72 6c 64 21   // Hello World! 
```

在 Go 语言中用 uint16 类型的 array/slice 结构来呈现上述的代码是一个很好的选择，因为它可以保存小端有序的单词对，同时也保持了可读性，如下所示：
```go
printFunction := []uint16{
         0x48c7, 0xc001, 0x0,                // mov %rax,$0x1
         0x48, 0xc7c7, 0x100, 0x0,           // mov %rdi,$0x1
         0x48c7, 0xc20c, 0x0,                // mov 0x13, %rdx
         0x48, 0x8d35, 0x400, 0x0,           // lea 0x4(%rip), %rsi
         0xf05,                              // syscall
         0xc3cc,                             // ret
         0x4865, 0x6c6c, 0x6f20,             // Hello_(whitespace)
         0x576f, 0x726c, 0x6421, 0xa,        // World!
} 
```

There is a slight deviation in the above bytes when compared to the bytes laid out above. This is because it is cleaner(easier to read and debug) to represent the data “Hello World!” when it is aligned to the start of a slice entry.
Therefore, I used the filler instruction cc instruction (no-op) to push the start of the data section to the next entry in the slice. I have also updated the lea to point to a location 4 bytes away to reflect this change.
注：你可以通过[这个链接](https://filippo.io/linux-syscall-table/)来查找各种 syscall 函数。
## 将 slice 转换为 function
保存在 []uint16 结构中的指令需要转换成一个可以被调用的 function ，这样这些指令才能够执行。下方的代码演示了如何进行转换：
```
type printFunc func()
unsafePrintFunc := (uintptr)(unsafe.Pointer(&printFunction)) 
printer := *(*printFunc)(unsafe.Pointer(&unsafePrintFunc)) 
printer()
```

A Golang function value is just a pointer to a C function pointer (notice two levels of pointers). The conversion from slice to function begins by first extracting a pointer to the data structure which holds the executable code. This is stored in `unsafePrintFunc`. The pointer to `unsafePrintFunc` can be typecast into the desired function type.
这种方法仅适用于无参数函数或是无返回值的函数。指令动态
This approach only works for functions without arguments or return values. A stack frame needs to be created for calling functions with arguments or return values. The function definition should always start with instructions to dynamically allocate the stack frame to support variadic functions. More information about different function types are available here.

## 使 function 可执行化
上述的函数还并不能真正地运行，这是因为 Golang 将所有的数据结构都存储在二进制文件的数据部分。而这部分数据会带有 [No-Execute](https://en.wikipedia.org/wiki/NX_bit) 标识来阻止它们被执行。
`printFunction` slice 中的数据需要被存在一片可执行的内存中。方法有二：可以通过移除 `printFunction` slice 上的 No-Execute 标识或是将它拷贝到可执行的内存位置来实现。
其中，后一方法更为可取，因为前者在移除设置在整页上的 No-Execute 标识后，很可能导致其他部分也变得可执行了。
在下方的代码中，（通过使用 mmap函数）将数据拷贝到了一片新分配的且可执行的内存空间中。
```go
executablePrintFunc, err := syscall.Mmap(
     -1,
      0,
      128,  
      syscall.PROT_READ | syscall.PROT_WRITE | syscall.PROT_EXEC, 
      syscall.MAP_PRIVATE|syscall.MAP_ANONYMOUS
      )
 if err != nil {
  fmt.Printf("mmap err: %v", err)
 }
j := 0
 for i := range printFunction {
  executablePrintFunc[j] = byte(printFunction[i] >> 8)
  executablePrintFunc[j+1] = byte(printFunction[i])
  j = j + 2
 }
```

`syscall.PROT_EXEC` 标识可以确保新分配的内存地址都是可执行的。将这个 `uint16` 类型的数据结构转换成 function 就可以让它顺利执行。下面给出完整代码，你可以在 x64 的机器上运行它们：
```go
package main
import (
 "fmt"
 "syscall"
 "unsafe"
)
type printFunc func()
func main() {
 printFunction := []uint16{
  0x48c7, 0xc001, 0x0,          // mov %rax,$0x1
  0x48, 0xc7c7, 0x100, 0x0,     // mov %rdi,$0x1
  0x48c7, 0xc20c, 0x0,          // mov 0x13, %rdx
  0x48, 0x8d35, 0x400, 0x0,     // lea 0x4(%rip), %rsi
  0xf05,                        // syscall
  0xc3cc,                       // ret
  0x4865, 0x6c6c, 0x6f20,       // Hello_(whitespace)
  0x576f, 0x726c, 0x6421, 0xa,  // World!
 }
 executablePrintFunc, err := syscall.Mmap(
  -1,
  0,
  128,
  syscall.PROT_READ|syscall.PROT_WRITE|syscall.PROT_EXEC,
  syscall.MAP_PRIVATE|syscall.MAP_ANONYMOUS)
 if err != nil {
  fmt.Printf("mmap err: %v", err)
 }
j := 0
 for i := range printFunction {
  executablePrintFunc[j] = byte(printFunction[i] >> 8)
  executablePrintFunc[j+1] = byte(printFunction[i])
  j = j + 2
 }
type printFunc func()
 unsafePrintFunc := (uintptr)(unsafe.Pointer(&executablePrintFunc))
 printer := *(*printFunc)(unsafe.Pointer(&unsafePrintFunc))
 printer()
}
```

##结语
试着运行上方的代码，敬请期待笔者对 Go 语言的更多深入探索吧！（译者注：敬请期待 GCTT 的更多好文。:)）