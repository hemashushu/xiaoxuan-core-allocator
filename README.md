# XiaoXuan Allocator

一个高效的动态 _heap_ 分配器，主要为 [XiaoXuan Lang](https://www.github.com/hemashushu/xiaoxuan-lang) 而设计。也可用于一般的应用程序或者内核的动态 _heap_ 分配。

特点：

- 为 _XiaoXuan Lang_ 的语言特点而优化；
- 分配和释放内存的速度快；
- 有效降低内部碎片和外部碎片的产生。

- - -

目录

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=4 orderedList=false} -->

<!-- code_chunk_output -->

- [XiaoXuan Lang 的内存管理方式](#-xiaoxuan-lang-的内存管理方式)
  - [在 _heap_ 上创建结构体对象](#-在-_heap_-上创建结构体对象)
  - [按指针传递结构体对象](#-按指针传递结构体对象)
  - [优缺点](#-优缺点)
- [编译和测试](#-编译和测试)
- [API 接口](#-api-接口)
  - [init](#-init)
  - [alloc](#-alloc)
  - [free](#-free)
- [原理](#-原理)
  - [内存的结构](#-内存的结构)
  - [_Heap_ 的结构](#-_heap_-的结构)
    - [_Header_ 的结构](#-_header_-的结构)
    - [_Data area_ 的结构](#-_data-area_-的结构)
  - [_可变大小分配器_ 的工作过程](#-_可变大小分配器_-的工作过程)
    - [_Free block_ 的插入及合并](#-_free-block_-的插入及合并)
  - [固定大小内存分配器](#-固定大小内存分配器)
    - [_索引页_ 的结构](#-_索引页_-的结构)
    - [_数据页_ 的结构](#-_数据页_-的结构)
    - [_索引页_ 和 _数据页_ 示例](#-_索引页_-和-_数据页_-示例)
  - [_固定大小分配器_ 的工作过程](#-_固定大小分配器_-的工作过程)
    - [_数据页_ 的创建过程](#-_数据页_-的创建过程)
- [多核环境](#-多核环境)
- [与其它内存分配器的比较](#-与其它内存分配器的比较)
- [License](#-license)

<!-- /code_chunk_output -->

## XiaoXuan Lang 的内存管理方式

_XiaoXuan Lang_ 运行环境的内存管理方式是 _无 GC 自动内存管理_ 型的，因为 _XiaoXuan Lang_ 的数据不可变（immutable），对于诸如结构体等数据对象，只需简单标记数据对象的引用数量即可准确且及时释放，而无需 GC 管理器。_XiaoXuan Lang_ 的数据除了基本的数据类型（诸如 int, float）和元组（tuple）之外，其余的都是结构体，所以下面主要考察结构体在内存中的管理方式。

### 在 _heap_ 上创建结构体对象

_XiaoXuan Lang_ 的所有结构体对象都是在 _heap_ 上创建，这点跟诸如 C 语言不同，在 C 语言里，局部的结构体对象是在 _stack_ 上创建的。

例如：

```c
// file "main.c"
#include <stdio.h>
#include <stdint.h>

struct User {
    uint32_t number;
    uint32_t count;
} __attribute__((packed));

int main(void) {
    struct User u1 = {
        number: 0x11,
        count: 0x22
    };

    printf("number: %d, count: %d\n", u1.number, u1.count);
}
```

编译上面的源代码然后反汇编：

```bash
$ gcc -g -o main.out main.c
$ objdump -d main.out
```

函数 `main` 的部分汇编/指令如下：

```s
00000000000011c0 <main>:
    11c0:       55                      push   %rbp
    11c1:       48 89 e5                mov    %rsp,%rbp
    113d:       48 83 ec 10             sub    $0x10,%rsp
    1141:       c7 45 f8 11 00 00 00    movl   $0x11,-0x8(%rbp)
    1148:       c7 45 fc 22 00 00 00    movl   $0x22,-0x4(%rbp)
```

上面的指令显示：程序在当前栈帧开始位置（往低地址方向）依次存储进 32 bits 的 0x22 和 0x11。可见结构体对象 `u1` 是直接在 _stack_ 上创建的。

> 如果想简单地了解 x86 的 GNU assembly 语法，可以参考这篇 [x86 Assembly/GNU assembly syntax](https://en.wikibooks.org/wiki/X86_Assembly/GNU_assembly_syntax)，或者 [x86 Assembly Guide](http://flint.cs.yale.edu/cs421/papers/x86-asm/asm.html)。如果想了解在 C 语言代码里嵌入汇编，可以参考 [6.47 How to Use Inline Assembly Language in C Code](https://gcc.gnu.org/onlinedocs/gcc/Using-Assembly-Language-with-C.html)

下面是上面程序的等效 _XiaoXuan Lang_ 代码：

```js
// file: "main.an"
struct User {
    int number
    int count
}

function main(List<string> args) type int {
    let u1 = User {
        number: 0x11
        count: 0x22
    }

    Console.log(`number: {{u1.number}}, count: {{u1.count}}`)
}
```

将源码编译到 _XiaoXuan IR_（中间语言）：

```bash
$ anlc --ir -o main.anir main.an
```

函数 `main` 的部分代码如下：

```clojure
(defn main:i32
    (args:ptr)
    (do
        (let u1 (struct 0 8))
        (write_u32 u1 8 0x11)
        (write_u32 u1 12 0x22)
        ...
    )
)
```

_XiaoXuan IR_ 当中的 `struct` 函数用于在 _heap_ 创建指定成员总大小的内存段落。可见对于局部结构体对象，_XiaoXuan Lang_ 是在 _heap_ 而不是 _stack_ 上创建的。

> 如果想了解 _XiaoXuan IR_ 的语法，可以参阅 [XiaoXuan IR Spec](https://www.github.com/hemashushu/xiaoxuan-ir)

### 按指针传递结构体对象

对于结构体类型的参数，_XiaoXuan Lang_ 总是传递对象的指针。例如下面的代码：

```js
function main(List<string> args) type int {
    let u1 = User {
        number: 0x11
        count: 0x22
    }

    show(u1)
}

function show(User u) {
    Console.log(`number: {{u.number}}, count: {{u.count}}`)
}
```

其对应的中间代码如下：

```clojure
(defn main:i32
    (args:ptr)
    (do
        (let u1 (struct 0 8))
        (write_u32 u1 8 0x11)
        (write_u32 u1 12 0x22)
        (self::show u1)
    )
)

(defn show:i32
    (u:ptr)
    (do
        ...
    )
)
```

可见函数 `show` 的参数 `u` 的数据类型是指针类型。这点也跟 C 语言不同，C 语言的结构体参数传递类型（_parameter passing mechanism_）是按值传递（_call by value_）。为了避免结构体的复制，一般将结构体类型的参数定义为指针类型。因此跟上面代码等效的 C 语言代码如下：

```c
void show(const struct User* const u) {
    printf("number: %d, score: %d\n", u->number, u->count);
}

int main(void) {
    struct User u1 = {
        number: 0x11,
        count: 0x22
    };

    show(&u1);
}
```

语句 `show(&u1)` 对于的汇编如下：

```s
    11a0:       48 8d 45 f0             lea    -0x10(%rbp),%rax
    11a4:       48 89 c7                mov    %rax,%rdi
    11a7:       e8 9d ff ff ff          call   1149 <show>
```

可见它把结构体的地址，也就是第一个成员的地址（此地址位于 _stack_ 之中）作为参数值传递给了函数 `show`。

### 优缺点

_XiaoXuan Lang_ 的所有结构体对象都在 _heap_ 上创建的策略，能简化语言的同时也能简化编译器以及内存管理设计和实现。比如当需要创建一个非局部的结构体对象（即离开结构体对象的声明语句作用范围之后仍需要访问的对象），C 语言需要显式地使用函数 `malloc` 分配内存，而局部变量则直接声明及初始化，在语言层面会出现两种不同且不可替换的结构体对象实例化的方式。

所有结构体对象都在 _heap_ 上创建的缺点是，相对在 _stack_ 上创建结构体对象，在 _heap_ 上创建需要耗费更多时间。_XiaoXuan Lang_ 使用两个方法来减少这种策略所带来的时间损耗：其一是在语言层提供了 _元组_（_tuple_）数据类型，_元组_ 是值类型的数据，在某些需要创建简单的（比如只包含两个基本数据类型成员的）局部数据时，可以使用 _元组_ 代替结构体；其二是针对 _XiaoXuan Lang_ 语言特点专门设计一个动态 _heap_ 分配器，也就是本项目 _XiaoXuan Allocator_。

## 编译和测试

TODO::

## API 接口

### init

`fn init(heap_pos: u32, brk: u32) -> Result<(), AllocatorErr>`

### alloc

`fn alloc(size: u32) -> Result<u32, AllocatorErr>`

### free

`fn free(addr: u32)`

TODO::

## 原理

分配器的作用是将内存中的一段连续区域（以下简称为 _内存片段_）的 _开始位置（即地址）_ 和 _长度_ 通过一定的方式记录下来，并将 _内存片段_ 的开始位置返回给请求者，请求者可以随时读写这个内存片段。在请求者尚未 **声明不再使用这个内存片段** 之前，分配器需要保证这个内存片段不会被其它程序占用、修改或者移动。

_XiaoYu Allocator_ 由 _可变大小分配器_ 以及 _固定大小分配器_ 两部分组成。其中 _可变大小分配器_ 使用一个 _显式的空闲块双向链表_ 实现，而 _固定大小分配器_ 使用一个 _索引页_ 和多个 _空闲项单向链表页_ 实现。

### 内存的结构

下图是一个典型的应用程序的内存视图：

![Memory View](./images/memory-view.png)

分配器负责分配上图当中的 _heap_ 区域的内存，即从 `heap start` 到 `brk` 之间的区域。

### _Heap_ 的结构

_heap_ 由一个 _header_ 和一个 _数据区域_ 组成：

```text
| header | data area |
```

#### _Header_ 的结构

```text
| magic number | data area pos | index page pos | first free block pos | brk value |
```

- `magic number`：[u8; 8]，用于表示 _XiaoYu Allocator_ 的幻数，值为 HEX `78 79 61 6c 6c 6f 63 01`，对应字符串 `xyalloc`，最后一个字节 `0x01` 表示版本号。`magic number` 用于标识 _heap_ 是否已经初始化；
- `data area pos`：uint64，_data area_ 的开始地址；
- `index page pos`：uint64，_index page_ 的开始地址；
- `first free block pos`：uint64，第一个 _free block_ 的地址；
- `brk value`：uint64，`heap` 的结束位置。

#### _Data area_ 的结构

_Data area_ 由 _零个或多个已分配的内存片段_ 以及 _一个或多个未分配的内存片段_ 组成。

为了方便描述，根据不同的分配状态，以下使用不同的名称来称呼内存片段：

- 已分配的内存片段称为 _已分配块_ （_allocated block_）；
- 未分配的内存片段称为 _空闲块_ （_free block_）。

在分配器初始化之后，_data area_ 只存在一个 _free block_，这个 _free block_ 占据了整个 _data area_。

随着应用程序的运行和时间的推移，许多 _allocated block_ 将会被创建（从 _free block_ 分割或转换而得），同时也有许多 _allocated block_ 被释放，被释放的 _allocated block_ 会重新转换为 _free block_。这时 _data area_ 会存在着许多个被 _allocated block_ 间隔开来的 _free block_。

_可变大小分配器_ 会 **按照位置的顺序** 将所有 _free block_ 串联起来，形成一个**显式的有序双向链表**，称为 _free block linked list_。分配器使用该链表管理空闲的内存。

_free block linked list_ 是一个 _有序链表_，所有 _free block_ 按照从 `低端地址` 到 `高端地址` 的顺序排列并连接。当一个 _allocated block_ 被释放后，会根据它所在的位置插入到链表的相应位置。如果相邻的位置刚好也是 _free block_，则会跟它们合并。

下面的图演示了一个包含有 3 个 _allocated block_ 和 3 个 _free block_ 的 _data area_：

![Free block linked list](./images/free-block-linked-list.png)

> _free block linked list_ 总会有一个 _free block_ 位于链表的末尾。

##### _Free block_ 的结构

```text
| block size | next block pos | free memory | previous block pos | block size |
```

- `previous block pos` 和 `next block pos`：uint64，分别是前一个和后一个 _free block_ 的位置。
- `block size`：uint64，是当前 _free block_ 的总长度（即 **包括** `block offset` 和 `block size` 等字段在内的长度）。

`previous block pos` 和 `next block pos` 的值在下列情况下其值为 0：

- 当 _free block_ 是链表的第一个 _free block_ 时，`previous block pos` 的值为 0；
- 当 _free block_ 是链表的最后一个 _free block_ 时，`next block pos` 的值为 0；
- 当整个链表只包含一个 _free block_ 时，它的 `previous block pos` 和 `next block pos` 的值都是 0。

##### _Allocated block_ 的结构

```text
| block size | data | block size |
```

注意：

- `data` 的长度必须是 8 的倍数。
- 分配内存时，分配器返回给应用程序的是 `data` 字段的开始位置（内存指针），当应用程序释放 _allocated block_ 时，（通过参数）传递给分配器的也是 `data` 字段的位置。但分配器管理内存片段时使用的是该片段的开始位置（即 `block size` 字段的位置）。

##### 标记位

分配器要求内存片段的大小必须是 8 的倍数（同时也是为了实现内存的 8 bytes 对齐），所以 `block size` 和 `block size` 数值的低 4 bits 的值总是 0。分配器将这最低 2 bits 用作为标记位，其中最低位是 `type` flag，第 2 低位是 `status` flag，如下图所示：

```text
|       [63:2]     | [1:1]  | [0:0] |
| block size value | status | type  |
```

这两个字段的数值及含义如下：

`status flag`

- 0：表示 _free block_；
- 1：表示 _allocated block_。

`type flag`

- 0：表示可变大小内存片段；
- 1：表示固定大小内存片段。

### _可变大小分配器_ 的工作过程

当分配器初始化之后，_free block linked list_ 只包含一个 _free block_，该 _free block_ 占据了整个 _data area_。

当应用程序请求分配一个 _可变大小_ 的内存片段：

1. _可变大小分配器_ 从链表的第一个 _free block_ 开始遍历，如果找到的 _free block_ 容量比请求的容量小，则继续往后找；

2. 当找到一个容量等于请求的大小的 _free block_ 时，_free block_ 会被转换为 _allocated block_；

3. 当找到一个容量大于请求的大小的 _free block_ 时，分配器会从 _free block_ 的左边（低地址端）分割一部分出来并成为一个 _allocated block_ ，而 _free block_ 会缩小至剩下的部分。

4. 当分配器遍历到最后一个 _free block_ 且容量不足以分配时，分配器会尝试增加 _heap_ 的空间，即向高地址端移动 `brk`。如果扩充成功，则最后一个 _free block_ 先会增大容量，然后再进行分割。如果扩充失败，则分配失败。

当应用程序释放 _allocated block_ 时：

1. 分配器会把该 _allocated block_ 转换为 _free block_，并根据 _allocated block_ 的位置插入到 _free block linked list_ 相应的位置（详细见下一节）；

2. 当新的 _free block_ 插入到 _free block linked list_ 且相邻的位置也是 _free block_ 时，则新的 _free block_ 将会跟相邻的 _free block_ 合并（详细见下一节）。

> 显然在链表里，不存在两个紧密相邻的 _free block_，因为它们会被合并。

#### _Free block_ 的插入及合并

当一个新 _free block_（即被释放的 _allocated block_）插入到 _free block linked list_ 时，会有以下这几种情况：

1. 前后均为 _allocated block_

![Merge A](./images/merge-a.png)

这种情况不需要合并 _free block_，只需把 _allocated block_ 转换为 _free block_ 即可。具体需要更新的字段有：

- 位于当前 _allocated block_ 之前的第一个 _free block_ 的 `next block pos` 字段，更新它的值为当前 _allocated block_ 的位置。
  注：如果当前 _allocated block_ 的位置比 `first node pos` 小，则说明不存在位于它之前的 _free block_，这时需要更新 `first node pos` 字段，让它的值为当前 _allocated block_ 的位置。也就是说，当前 _allocated block_ 将会成为第一个 _free block_；
- 位于当前 _allocated block_ 之后的第一个 _free block_ 的 `previous block pos` 字段，更新它的值为当前 _allocated block_ 的位置；
- 当前 _allocated block_ 的 `status` flag 更新为 0；
- 当前 _allocated block_ 添加 `next block pos` 和 `previous block pos` 字段，分别指向下一个和前一个 _free block_ 的位置。注：如果当前 _allocated block_ 即将要成为链表的第一个 _free block_，则 `previous block pos` 的值应该为 0。

> 分配器采用遍历 _free block linked list_ 的方法来寻找当前 _allocated block_ 之前及之后的第一个 _free block_。即从第一个 _free block_ 开始遍历，找到第一个位置比当前 _allocated block_ 大的 _free block_，该 _free block_ 则为当前 _allocated block_ 的后一个 _free block_。读取该 _free block_ 的 `previous block pos`，则找到当前 _allocated block_ 的前一个 _free block_。

2. 前一个是 _free block_

![Merge B](./images/merge-b.png)

这种情况需要合并前一个 _free block_。具体需要更新的有：

- 前一个 _free block_ 的第一个 `block size` 字段，更新它的值为它们两者的总长度；
- 当前 _allocated block_ 的第二个 `block size` 字段，更新它的值为它们两者的总长度；
- 当前 _allocated block_ 添加 `previous block pos` 字段，其值为前一个 _free block_ 的位置。

3. 后一个是 _free block_

这种情况跟 _情况2_ 类似，不同的是需要合并后一个 _free block_。具体需要更新的有：

- 后一个 _free block_ 的第二个 `block size` 字段，更新它的值为它们两者的总长度；
- 当前 _allocated block_ 的第一个 `block size` 字段，更新它的值为它们两者的总长度；
- 当前 _allocated block_ 添加 `next block pos` 字段，其值为前一个 _free block_ 的位置。

4. 前后都是 _free block_

![Merge C](./images/merge-c.png)

这种情况需要合并前后两个 _free block_。具体需要更新的有：

- 前一个 _free block_ 的第一个 `block size` 字段，更新它的值为它们三者的总长度；
- 后一个 _free block_ 的第二个 `block size` 字段，更新它的值为它们三者的总长度。

### 固定大小内存分配器

_固定大小分配器_（_fixed size allocator_）是 _XiaoYu Allocator_ 的一个子系统，它运行于 _可变大小分配器_ 的基础之上。用于快速分配具有固定大小且容量较小（小于等于 256 bytes）的内存。

> 当程序请求分配或者释放一个容量大于 256 bytes 的内存片段时，则由 _可变大小分配器_ 程序管理。而小于等于 256 bytes 的内存片段则由 _固定大小分配器_ 程序管理。

> _XiaoYu Allocator_ 则是以 128 bytes 作为分界线。

_固定大小分配器_ 将内存片段按照长度每 8 bytes 分为一 _类_（_class_），比如 `1 ~ 8 bytes`（包括 1 和 8）作为第 1 类，`9 ~ 16 bytes`（包括 9 和 16）作为第 2 类，`17 ~ 24 bytes`（包括 17 和 24）作为第 3 类，如此类推。256 bytes 以内的长度共被分为 `256 / 8 = 32` 类。

固定大小分配器由 1 个 _索引页_（_index page_） 和多个 _数据页_（_data page_） 组成。_索引页_ 和 _数据页_ 均被包装在 _allocated block_ 里，如下图所示：

![Fixed size allocator view](./images/fixed-size-allocator.png)

#### _索引页_ 的结构

索引页包含有 32 个 uint64 整数（称为 _head item offset_），每一个整数指向一个 _free item linked list_ 的表头。

#### _数据页_ 的结构

一个数据页里存储着多个长度相同的内存片段，这些内存片段被称为 _内存项_（_memory item_）。根据分配状态的不同，下面使用不同的名称来称呼 _内存项_：

- 空闲的 _内存项_ 称为 _空闲项_ （_free item_）
- 已分配的 _内存项_ 称为 _已分配项_ (_allocated item_)

所有同一 _class_ 的 _free item_ 将会被连接起来形成一个单向链表，称为 _空闲项链表_ （_free item linked list_）。

在应用程序运行过程中，不断地有 _free item_ 被转变为 _allocated item_，_allocated item_ 会暂时脱离 _free item linked list_；同时也有 _allocated item_ 重新被转变为 _free item_，新的 _free item_ 会被插入到 _free item linked list_ 的头部，因此 _free item linked list_ 是一个 **无序单向链表**。

![Data page view](./images/data-page-view.png)

当 _free item linked list_ 所有 _free item_ 都被分配之后，分配器会新建一个同类的 _数据页_，并且将新的 _数据页_ 的第一个 _free item_ 连接到 _free item linked list_ 的末尾。

![Multiple data page](./images/multiple-data-page.png)

##### _Free item_ 的结构

```
| next free item offset | class idx | flags | free memory |
```

- `next free item offset`：下一个 _free item_ 位置相对 _index page_ 的偏移值；
- `class idx`：5 bits，当前项的 _class_ 的索引值，值的范围是从 0 到 31；
- `flags`：2 bits，最低端位的值为 1，表示固定大小的内存片段，第 2 低端位暂时是保留的，它的值会被忽略。

_memory item_ 以紧凑的方式排列，`next free item offset`、`class idx` 以及 `flags` 3 个字段共享一个 uint64，因为 `class idx` 和 `flags` 已占用低端的 `5 + 2 = 7` bits，所以 `next free item offset` 只能使用高端的 `64 - 7 = 57` bits。又因为内存段落以 8 bytes 对齐，所以 `next free item offset` 可以表达的最大值为 `2 ^ (57 + 4) = 2^61 bytes`，这是一个很大的数字。

##### _Allocated item_ 的结构

```
| padding | class idx | flags | data |
```

`padding` 字段实际上是 _free item_ 的 `next free item offset` 字段，因为 _内存项_ 被分配出去之后，原先 `next free item offset` 字段的值此已不再具有任何意义，该字段的值既可以清零，也可以保持不变。

注意，无论是 _allocated block_ 还是 _allocated item_，分配器返回给应用程序的都是其中的 `data` 字段的位置（内存指针）。当 _allocated block_ 或者 _allocated item_ 被释放时，比如应用程序调用了函数 `void free(int addr)`，通过参数传递给分配器的其实是 `data` 字段的位置。分配器需要根据这个地址往后 8 bytes 读取一个 uint64 整数，这个整数的低端 4 bits 就是 _标记位_。通过 _标记位_ 的 `type` flag 就可以区分该内存片段是 _allocated block_ 还是 _allocated item_，知道内存片段的类型之后再由相应的分配程序来执行具体的释放过程。

#### _索引页_ 和 _数据页_ 示例

![Index page and data page](./images/index-page-and-data-page.png)

### _固定大小分配器_ 的工作过程

当应用程序请求分配一个固定大小的内存片段时：

1. 根据请求的内存片段的大小计算 _class idx_，公式是 `truncate((n - 1) / 8)`，比如 16 bytes 的 `class idx` 是 `truncate((16 - 1) / 8) = 1`；

2. 在 _索引页_ 找到相应索引值的 `head item offset`，如果 `head item offset` 的值为 0，说明该 _class_ 对应的 _数据页_ 尚未创建。此时 _固定大小分配器_ 会先创建相应的 _数据页_ 并更新 `head item offset` 的值。

3. 将 `head item offset` 指向的 _free item_ 转换为 _allocated item_，然后更新 `head item offset`，让它的值为原链表的第二个 _free item_ 的位置；

当应用程序释放一个 _allocated item_ 时：

1. 读取被释放的 _allocated item_ 的 `class idx` 字段，然后获取该 _class_ 的 `head item offset` 值；

2. 更新被释放的 _allocated item_ 的 `next free item offset` 字段，让它的值等于 `head item offset`；

3. 更新 `head item offset` 字段，让它的值等于被释放的 _allocated item_ 的位置。这样一来，被释放的 _allocated item_ 又重新被插入到 _free item linked list_ 并成为链表的头。

> 虽然 _free item linked list_ 是一种链表结构，但工作过程更接近数据结构当中的 _栈_（_statck_）概念。可以认为 _free item linked list_ 是一个由一堆 _free item_ 组成的栈。当需要分配内存片段时，从该 _栈_ 弹出一项 _free item_；当内存片段被释放时，它会重新被压入 _栈_ 并成为新的栈顶。

_固定大小分配器_ 比起 _可变大小分配器_ 节省了分割与合并链表项、搜索前后 _free block_、遍历 _free block_ 以找到合适大小的 _free block_ 等等操作，因此 _固定大小分配器_ 的效率非常高，能胜任频繁的内存分配和释放任务。

#### _数据页_ 的创建过程

在应用程序请求分配固定大小内存片段时，如果 `head item offset` 的值为 0，或者 _free item linked list_ 只剩下一项 _free item_，则必须创建新的 _数据页_。创建 _数据页_ 的过程如下：

1. 向 _可变大小分配器_ 申请一个 _allocated block_，然后在里面初始化一个由多个 _free item_ 组成的 _free item linked list_，每一个 _free item_ 均指向下一个 _free item_，最后一个 _free item_ 的 `next free item offset` 值为 0。

2. 如果新建的 _数据页_ 是指定 _class_ 的第一个数据页，则更新 `head item offset`，让它的值为新 _数据页_ 第一个 `free item` 的位置的偏移值。

3. 如果新建的 _数据页_ 不是指定 _class_ 的第一个数据页，则更新该 _class_ 的 _free item linked list_ 的最后一个 _free item_ 的 `next free item offset` 字段，让它值为新 _数据页_ 第一个 `free item` 的位置的偏移值。

一个 _数据页_ 必须 **至少** 包含两个 _item_。比如长度为 256 bytes 的 _class_ 的 _item_，因为至少需要包含两项，所以需要申请长度为 `(256 + 8) * 2 = 528` bytes 的 _allocated block_。

如果 _XiaoYu Allocator_ 运行在内存比较充裕的环境，最低 _item_ 数量可以配置一个比较大的数字，比如要求至少 16 项。同时要求在创建 _数据页_ 时最小长度应该大于 4096 bytes。比如长度为 16 bytes 的 _class_ 的 _item_ 长度为 `16 + 8 = 24 bytes`（其中 8 bytes 是 _item_ 的 header 的长度），该 _class_ 的数据页的长度应该为 4104 bytes。其计算公式是：`(floor(4096 / 24) + 1) * 24 = 171 * 24 = 4104 bytes`。

## 多核环境

TODO::

## 与其它内存分配器的比较

目前已存在比较成熟且被广泛应用的内存分配器，比如集成到 glibc 的 `ptmalloc2`，以及 `jemalloc`、`TCMalloc`、`Hoard` 等等，显然内存分配是一项跟实际应用场景密切相关的工作，其中存在着几个相互矛盾的需求，分配器需要针对应用场景在各个方面进行平衡和取舍。_XiaoXuan Allocator_ 的主要实现方法也是受到主流分配器所启发，不过针对 _XiaoXuan Lang_ 程序的运行特点而进行了改进和优化。另外，跟其它以 _XiaoXuan_ 命名的项目一样，我希望用新的语言工具从零开始实现这些基本的组件（_XiaoXuan Script_ interpreter 版本的分配器使用 Rust 实现，_XiaoXuan VM_ 和 _XiaoXuan Native_ 则使用 _XiaoXuan IR_ 直接实现），于是就有了当前这个项目。

另外，[XiaoYu OS](https://www.github.com/hemashushu/xiaoyu-os) 内核采用的动态内存分配器 [XiaoYu Allocator](https://www.github.com/hemashushu/xiaoyu-allocator) 是当前这个分配器的 32 位版本，主要用于诸如 MCU 等内存资源非常有限的设备。

如果对其它分配器的原理感兴趣，可以参考：

- [Doug Lea's Malloc](https://gee.cs.oswego.edu/dl/html/malloc.html)
- Introduction to ptmalloc2 internals [Part 1](https://blog.k3170makan.com/2018/11/glibc-heap-exploitation-basics.html), [Part 2](https://blog.k3170makan.com/2018/12/glibc-heap-exploitation-basics.html), [Part 3](https://blog.k3170makan.com/2019/03/glibc-heap-exploitation-basics.html)
- [jemalloc](http://jemalloc.net/jemalloc.3.html)
- [TCMalloc](https://goog-perftools.sourceforge.net/doc/tcmalloc.html)
- [Hoard](https://people.cs.umass.edu/~emery/pubs/berger-asplos2000.pdf)
- [nedmalloc](https://www.nedprod.com/programs/portable/nedmalloc/)

## License

Copyright (c) 2023 Hemashushu, All rights reserved.

This Source Code Form is subject to the terms of the Mozilla Public License, v. 2.0. If a copy of the MPL was not distributed with this file, You can obtain one at http://mozilla.org/MPL/2.0/.