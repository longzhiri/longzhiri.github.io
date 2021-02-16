---
layout: post
title: 深入Golang函数变量的内部
tags: Golang
---

今天在对一个golang工程做性能分析时遇到一个需求：有一个函数列表，存放了各种函数，程序一个一个处理列表中的函数，现在需要统计每个函数执行时间，并且打印执行慢的函数的名字及所在文件名和行号。runtime.FuncForPC可以获取一个函数的这些信息，但是参数需要传入函数地址，如何正确获取函数地址呢？取决于一个函数变量和函数地址的关系，让我们看一个例子：
```Golang
package main

func foo() {
}

func main() {
    var f [2]func()
    f[0] = foo 
    f[1] = foo 
    f[0]()
    f[1]()
}                 
```
编译成二进制文件，objdump -D main 反汇编结果main.main部分：
```X86ASM
...
0000000000452340 <main.main>:
  452340:   64 48 8b 0c 25 f8 ff    mov    %fs:0xfffffffffffffff8,%rcx
  452347:   ff ff  
  452349:   48 3b 61 10             cmp    0x10(%rcx),%rsp
  45234d:   76 42                   jbe    452391 <main.main+0x51>
  45234f:   48 83 ec 18             sub    $0x18,%rsp
  452353:   48 89 6c 24 10          mov    %rbp,0x10(%rsp)
  452358:   48 8d 6c 24 10          lea    0x10(%rsp),%rbp
  45235d:   0f 57 c0                xorps  %xmm0,%xmm0
  452360:   0f 11 04 24             movups %xmm0,(%rsp)
  452364:   48 8d 05 dd 59 02 00    lea    0x259dd(%rip),%rax        # 477d48 <go.func.*+0x46>
  45236b:   48 89 04 24             mov    %rax,(%rsp)
  45236f:   48 89 44 24 08          mov    %rax,0x8(%rsp)
  452374:   48 8b 14 24             mov    (%rsp),%rdx
  452378:   48 8b 02                mov    (%rdx),%rax
  45237b:   ff d0                   callq  *%rax  
  45237d:   48 8b 54 24 08          mov    0x8(%rsp),%rdx
  452382:   48 8b 02                mov    (%rdx),%rax
  452385:   ff d0                   callq  *%rax  
  452387:   48 8b 6c 24 10          mov    0x10(%rsp),%rbp
  45238c:   48 83 c4 18             add    $0x18,%rsp
  452390:   c3                      retq   
  452391:   e8 0a 7b ff ff          callq  449ea0 <runtime.morestack_noctxt>
  452396:   eb a8                   jmp    452340 <main.main>
...
```
地址452364~45236b的指令可以认为匹配 f[0] = foo ，可以看出来函数变量实际上被赋予了地址0x259dd(%rip)，亦即注释中的地址值477d48，查找这个地址：
```X86ASM
    ...
  477d46:   00 00                   add    %al,(%rax)
  477d48:   30 23                   xor    %ah,(%rbx)
  477d4a:   45 00 00                add    %r8b,(%r8)
  477d4d:   00 00                   add    %al,(%rax)
  477d4f:   00 10                   add    %dl,(%rax)
  ...
```
后面的汇编指令无意义，只看477d48地址内存中8个字节的值是 452330 （little endian byte order），看起来也是一个地址值，查找这个地址：
```X86ASM
...
0000000000452330 <main.foo>:
  452330:   c3                      retq
  452331:   cc                      int3
  452332:   cc                      int3
  452333:   cc                      int3
  452334:   cc                      int3
  452335:   cc                      int3
  452336:   cc                      int3
  452337:   cc                      int3
  452338:   cc                      int3
  452339:   cc                      int3
  45233a:   cc                      int3
  45233b:   cc                      int3
  45233c:   cc                      int3
  45233d:   cc                      int3
  45233e:   cc                      int3
  45233f:   cc                      int3

0000000000452340 <main.main>:
...
```
正好是foo函数地址。由此我们可以得出结论：函数变量存的其实是一个指针指向函数地址，因此我们可以这样获取函数地址：
```Golang
package main

import (
    "log"
    "runtime"
    "unsafe"
)

func foo() {
}

func main() {
    var f func()
    f = foo 
    pc := **(**uintptr)(unsafe.Pointer(&f))
    fi := runtime.FuncForPC(pc)
    log.Printf("func name=%v", fi.Name())
}                            
```
go源码中没有函数变量内存布局的结构定义，但是slice，string类型都有：
```Golang
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}


type StringHeader struct {
	Data uintptr
	Len  int
}
```

特别的，对于interface{}类型变量，go源码中定义的内存布局是这样的：
```Golang
type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```

_type 表示类型，data 指向具体值，可以理解interface{}为什么保存了类型信息，这是为了保证type switch和type assert正常工作，同时虽然它很像c语言中的void*，但是他能相对安全的使用。

interface{}变量有一个很容易误用的case，变量i本身是nil和通过一个具体类型的nil值赋值的i是具有不同意义的。interface{}可以认为是一种特殊的interface type，所有类型都实现了这个特殊接口，因此这个容易误用的case存在在任何interface上，举一个例子：
```Golang
package main

import "log"

type IFoo interface {
    foo()
}

type Foo struct {
    d int
}

func (f *Foo) foo() {
    log.Printf("d=%v", f.d)
}

func main() {
    var ifs []IFoo
    var f Foo
    var pf *Foo
    ifs = append(ifs, &f, pf, nil)
    for _, i := range ifs {
        if i != nil {
            i.foo()
        }
    }
}
```
这段代码最终会panic在第二个i.foo()调用上，我们容易记得判断接口是否为nil，但是常常忘记判定接口中的值是否为nil，一种解决方法是使用反射：
```Golang
 for _, i := range ifs {
        if i != nil && !reflect.ValueOf(i).IsNil() {
            i.foo()
        }
   }
```

