---
layout: post
title: MIT 6.858 Course 2 - Control Hijacking Attacks
categories: [MIT 6.858, Course]
---

## 缓冲区溢出的根本原因？

1. 系统软件大多是 C 写的，C 速度快、接近硬件...，而 C 暴露 raw memory 且没有边界检查
2. 对 x86 架构的了解：攻击者很清楚栈结构（所以溢出时知道怎么构造假的栈帧）、函数/系统调用约定（Calling Convention）

## 为什么 OS 不主动防止 buffer overflow？

因为系统（软件部分）不可能事无巨细、面面俱到，overflow 行为发生在完全属于程序自己的内存空间；而硬件才是会参与、干预每一项细微操作，因此这类问题可以从硬件角度解决。

## 程序如何消除 buffer overflow 漏洞？

消除漏洞的意思是，从语言和编程入手，让程序本身无漏洞。

### 措施 1. c 语言要避免写出 bug，甚至避免使用 c

对于 c 这种弱类型语言来说，很难区分 bug 非 bug。随着机器、语言的性能越来越好，很多时候仍然坚持使用 c 是没意义的。

### 措施 2. 使用工具来找 bug，包括

1. 静态分析
2. fuzzing

fuzz 本质上是找出隐藏 bug 的分支，普通的功能测试通常无法覆盖所有的分支，静态分析可以和 fuzz 结合（帮助生产 fuzz 用例）

```c
void foo(int *p){
    int offset;
    int *z = p + offset; //到这里，静态分析工具会提示有问题，offset 未初始化。
    if(offset > 7){ //从静态分析，fuzz 用例可以构造 <7、 =7 、>7 三种情况
        bar(offset);
    }
}
```

以上这些方法都无法完全消除安全 bug

### 措施 3. 使用内存安全语言 Python,Java,c#

这个方法的局限在于：

- 有很多历史遗留代码是 c
- 要在底层操作硬件（驱动程序等等）这个场景需要 c 的好处
- 性能变差（这些语言面世时无一例外都由解释器解释执行，上而不是直接使用 x86指令。即编译成中间码的高层级指令，然后由一个程序 loop 逐一读入指令并执行，例如 JVM 执行指令就是基于对 stack 的 push pop 操作来模拟这些操作）当然解决办法后面也有出来，比如 JIT 编译就是在运行时编译成 x86 指令

需要性能的场景：程序是 CPU bound，CPU速度很重要    
不需要性能的场景：程序是 IO bound，大部分时间都是在等待用户输入、硬盘、网络包等等，这些场景实际并不需要高速的原生计算，

## 如果 buffer overflow 不可避免，有什么缓解措施？

- buffer overflow 的本质
    1. 获取指令指针（IP）的控制权
    2. 劫持IP到恶意代码

缓解措施着眼于溢出发生时，阻止攻击者进行以下步骤：

- **改写代码指针**劫持控制流，例如返回地址，函数指针，C++ vtable， 异常处理句柄
- 在内存中后注入或已存在**恶意代码**
- 将恶意代码安置在**可预测位置**，令代码指针指向该位置

## 缓解措施 1. Stack Canary 

### Canary 如何防范被攻击者绕过？

1. 使用 0，CR，LF，-1 等特殊字符（攻击者要用这些字符来 overflow 构造 Canary 时，会让被攻击的函数停止，比如 gets 就是 0），当然对于那些不涉及特殊字符终止的场景，就无效了
2. 使用随机化的 canary

### Canary 无法防范的场景？

1. 指针地址覆盖。因为被践踏的指针地址被代码使用时，并没有 canary 检查，当然如果在每个指针使用前执行检查，那代价昂贵...

一个指针覆盖例子：

```c
int *ptr = ...;
char buf[128];
gets(buf); //Buffer is overflowed, and overwrites ptr.
*ptr = 5; //Writes to an attacker-controlled address!
//Canaries can't stop this kind of thing.
```

Canary 起作用的条件 overflow 发生后立即执行检查的场景，典型的就是 ret 前检查 buf。在 return 前执行栈检查指令，需要编译器的帮助，以拓展原来的调用约定 —— 原来是 leave 然后 ret，现在就是在 ret 前插入 canary 相关指令。

2. 随机数被猜测

3. malloc and free 攻击

```c
int main(int argc, char **argv) {
    char *p, *q;
    p = malloc(1024);
    q = malloc(1024);
    if(argc >= 2)
        strcpy(p, argv[1]);
    free(q);
    free(p);
    return 0;
}
```

```
+----------------+
|                |
|    App data    |
|                |    Allocated memory block
+----------------+
|      size      |
+----------------+


+----------------+
|      size      |
+----------------+

|   ...empty...  |
+----------------+
|     bkwd ptr   |
+----------------+
|     fwd ptr    |    Free memory block
+----------------+
|      size      |
+----------------+ 
```

原理是某块 malloc 的空间发生 overflow 后，可以覆盖相邻块空间的 size 字段，这样当堆管理系统 free 这些空间时，由于有两个 free，会将相邻的两块空闲空间合并：

```c
p = get_free_block_struct(size);//size 是被控制的，那么 p 也是可以构造的了，也就是下面两个 write 可以被控制来往任意地址写入两个 pointer 长度的数据
bck = p->bk;
fwd = p->fd;
fwd->bk = bck; //Writes memory!
bck->fd = fwd; //Writes memory!
```

（上述过程相当于将空闲 block 的双向链表中某个节点去掉，并重新连通）

但实际上如果不精心构造，大部分情况都是非法地址写入，产生段错误。

## 缓解措施 2. 边界检查

- 总体目标：

通过检查指针是否在边界内来阻止指针误用

- 难点：

首先 c 语言里面本来就很难判断一个指针本身（指针类型）是否是合法还是非法，同样是 char * 指针，用来访问 string 类型的 char 数组是没问题的，但是如果该 char 数组是网络数据包，那就不行了（显然要用对应结构体的指针）

更典型的是利用联合体指针访问内存对象，根据实际内存的不同，可能 valid 也可能是 invalid

造成上述状况的根本原因在于，c 语言中指针本身并不包含使用意图语义。

所以，要完全解决是很难的，所以退而求其次，我只要求指针访问的内存，是在边界内即可，具体语义是：指针派生的指针，后者只能解引用属于前者的合法的内存区域

即使是退而求其次，也是很有意义的，能够防止内存的随意覆盖：程序只能践踏实际分配的、属于自己的内存。这已经的 c 语言世界里的一大改进了！

### Electric fences

在每个堆对象的边界插一个 guard page，再通过 page tables 来确保对该 page 的访问会立即产生错误。

```
+---------+
|  Guard  |
|         |  ^
+---------+  | Overflows cause a page exception
|  Heap   |  |
|  obj    |  |
+---------+
```

也是一种非常好用的 debugg 技术，只要发生 over flow，立即导致 crash 而不是静悄悄或拖到未来才发现。

最大的优点：使用非常方便。无需修改源代码、无需编译器支持，只要修改 malloc 库函数以实现 Electric fences 就行。  
缺点：耗费空间，特别是存储特别小的堆对象时。所以生产环境很少使用。

### Fat Pointers

思想：修改普通指针，使其除了地址以外，包含首尾边界信息。编译器自动增加额外的代码，使得当访问 fat 指针使其地址更新时，fat 指针的边界也被强制检查

```
+-----------------+
| 4-byte address  |
+-----------------+

Fat pointer (96 bits)
+-----------------+----------------+---------------------+
| 4-byte obj_base | 4-byte obj_end | 4-byte curr_address |
+-----------------+----------------+---------------------+
```

缺点1：每次指针解引用前都执行边界检查，是一项昂贵的开支，对于 C 语言这种追求性能的语言来说，更是如此。 

缺点2：与现成的大量软件不兼容：
- 现成的库未经修改的话，不接受 fat 指针
- 包含 fat 指针的数据结构，sizeof 大小会改变（所以，原来哪些使用硬编码数据结构大小的代码，会出问题）
- fat 指针的更新不是原子化的，而有些程序假设指针写入是原子化的。fat 指针不再是 1 word，而含是多个 word（在 32bit 机器上，32bit 即 1 word 写入是原子化的），其更新操作包含多个步骤。

同样的，有令人恶心的副作用，所以生成环境也少用。

### 使用 shadow data structures 来跟踪边界信息

> 论文：Use shadow data structures to keep track of bounds information (Jones and Kelly, Baggy).

#### 基本思想

对于每一个分配的对象，同时额外地（使用 shadow data structure）存储该对象的大小信息。举例：存储 malloc 时传递的大小值：

```c
char *p = malloc(mem_size);
```

对于大小值是静态的，值由编译器确定：

```c
char p[256];
```

然后，对于这样的每个指针，干预两个操作来实现边界检查。

- 指针运算： `char *q = p + 256`
- 指针解引用： `char ch = *q;`

为什么一定要两个操作都干预，只使用某一项不行吗？在指针运算判断的非法指针不一定都是 bug，比如所谓的 “哨兵” 指针（比如数组的最后一项 +1 项），可以用来作为循环终止条件，所以这时候就需要干预解引用了；而指针解引用则是要依赖指针运算阶段获取到 OOB （out of bound）位，没有 OOB 以及当前指针所在的位置，那么解引用时是否越界也就无从判断了。

#### 如何实现？

挑战 1：怎么存储边界信息呢？即指针地址 -> 边界信息 这个映射关系。效率是一个重要问题。

- 简单方案 1：使用哈希表或者树，存储映射关系。好处：节省空间（只存使用中的指针，而不是所有可能的指针）；坏处：寻找速度慢（每次查找需要访问多处内存，即使是 Hash 表，也有溢出）
    注：Jones and Kelly, Baggy 的[论文](https://github.com/YuZhang/Security-Courseware/blob/master/buffer-overflow/supplements/baggy-bound-checking-USENIX2009.pdf)就提出了一种非常高效的数据结构，来跟踪这些边界信息，使得边界检查变得非常快。

- 简单方案 2：使用数组来存储每个内存地址的边界信息，速度快但占用内存。

挑战 2：如何强制 OOB 指针解引用失败？

- 简单方案：检测每个指针解引用，优点，可行，缺点是很高的内存消耗

后面介绍了 [宽松边界检查（Baggy Bounds Checking）](https://www.usenix.org/legacy/events/sec09/tech/full_papers/akritidis.pdf) 的原理，简单说就通过引入一种新的内存分配机制，使得指针边界计算、边界信息存储（通过数组）、边界信息索引变得非常高效，并且利用 VM（虚拟内存）系统自身的内存保护特性，把要解引用的非法指针的关键 bit 设置到非法内存区域，让 paging 硬件报错，这样就不用对解引用采取额外的阻断操作了。