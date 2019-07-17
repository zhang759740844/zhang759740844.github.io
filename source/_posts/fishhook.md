title: fishhook 源码解析
date: 2019/7/7 14:07:12  
categories: iOS
tags: 

 - 源码解析
	
---

提起 iOS 中的 hook 手段，我们最先想到的是 Method Swizzling，这是 hook OC 方法最重要的手段。fishhook 则提供了 hook 系统 c 函数的手段。

<!--more-->

## 动态链接

### 为什么要有动态链接

为了减少应用的体积，加速应用的启动速度。苹果系统会将很多系统库设计为动态库。动态库的实际地址在应用编译的时候是未知的。一些符号会在应用**启动的时候链接**。但是如果这样的符号过多就会拖慢应用的启动速度。因此另一些非必要符号会在**第一次使用的时候绑定**，也就是 Lazy Binding。

### 从例子看 lazy Binding

我们来通过一个系统方法 NSLog 来验证动态库符号的绑定过程。我们在 main.m 中输入两个打印语句，然后加上断点：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook1.png?raw=true)

我们来看一下它的汇编执行过程，需要先设置：debug -> debug workflow -> always show disassembly。断点情况下的汇编代码如下：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook2.png?raw=true)

我们看到第14行，`bl 0x10470ebf0； symbol stub for: NSLog` 就是执行打印的方法。这个方法对应于可执行文件的哪个段呢？我们通过 MachOView 查看它的可执行文件：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook4.png?raw=true)

在 `__Text,__stubs` 中有一个对应于 NSLog 的方法。它的 offset 是 00006BF0。但是我们实际的地址是 0x10 470ebf0，这两个地址显然是不匹配的。这是因为 **MachOView 中的 offset 是相对于 `__Text` 段开始的 offset.(Header 和 Load Commands 也是属于 `__Text` 段的 )**。如果要换算到实际地址，就需要加上 `__Text` 段的实际地址。以下是地址的换算关系：

> 虚拟地址 + ASLR = 实际地址
>
> `__Text` 的虚拟地址 + ASLR = `__Text` 的实际地址
>
> `__Text` 的虚拟地址 + 对象的 offset = 对象的虚拟地址
>
> `__Text` 的实际地址 + 对象的 offset = 对象的实际地址

我们可以通过 lldb 中执行 `image list` 查看 `__Text` 段的实际地址：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook5.png?raw=true)

如上图所示，`image list` 能列举出当前应用加载的所有动态库。其中第一个就是当前应用的地址，也就是 0x0000000104708000。即

>  0x00006BF0 + 0x104708000 = 0x10470ebf0

> 由于 `__Text` 虚拟地址默认是 0x100000000，因此，我们还能计算得到  ASLR 地址:
>
> 0x104708000 - 0x100000000 = 0x4708000
>
> 当然，ASLR 地址并不是我们关注的重点，只是略带提及。
>
> 如果要直接获取 ASLR 地址，可以使用 `image list -o -f` 指令，就会默认减去 `__Text` 段的虚拟地址

那么这个地址上的方法是什么呢？可以在 lldb 中通过 `dis` 命令反汇编输出：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook3.png?raw=true) 



在 MachOView 中这个地址的数据为 `1F2003D5F0A0005800021FD6` 。这是一段16进制的机器码。我们可以验证一下它和反汇编的结果是否一致。如果我们自己查表会非常费事，有一个 [ARM to Hex 的网站](http://armconverter.com/)，可以帮助我们将汇编代码转换为 Hex。我们只验证第二句汇编语句的正确性：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook7.png?raw=true)

通过对比可以发现，MachOView 中相应 offset 的 Data 反汇编后就是 `dis` 生成的汇编代码。

重新看一下反汇编后的第二三条汇编代码。他们的作用是加载相对于当前地址，偏移量为 `0x141c` 的内存到寄存器 x16 中，然后执行。实际执行的地址的计算过程如下：

> 0x141c + 0x6BF4 = 0x8010
>
> 第一条汇编代码 offset 为 0x6BF0，那么第二条汇编代码 offset 即为 0x6BF4

在 MachOView 中体现为 `__DATA,__la_symbol_ptr`：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook9.png?raw=true)

在 0x8010 的地址上的 data 为 0x0000000100006C2C。前面说到， `__Text` 虚拟地址默认是 0x100000000，这在 TEXT 段的 Load Commands 中有所体现：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook10.png?raw=true)

所以实际的偏移地址为 0x6C2C。它的地址在 `__Text,__stub_helper` 中：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook8.png?raw=true)

`__Text,__stub_helper` 会调用 `dyld_stu_binder` 方法计算并绑定 `NSLog` 函数的真实地址。

让程序走过这个方法，再次通过 `dis` 查看汇编代码：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook6.png?raw=true)

此次执行的方法地址不再是 0x10470ec2c，而是 0x2065176e6。也就是说进过了绑定之后， `__Text,__stub` 的方法指向发生了改变，指向了系统的动态库的方法。至此完成了 lazy binding 的过程。

### 从例子看非 lazy binding

非 lazy binding 的符号会在应用启动的时候就 binding 完成。我们简单验证一下。首先看 `__DATA,_nl_symbol_ptr` 段在 MachOView 中的符号信息，在 MachOView 中的非 lazy binding 符号指向的地址都是 0x0：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook_13.png?raw=true)

当程序运行并加载成功后，我们再看相应位置的 data：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook_14.png?raw=true)

可以看到相应位置已经不再是 0x0，而是具体的链接完成后的地址了。

## fishhook

通过上面的两个例子的分析，我们能知道，动态链接的符号不是位于程序的 `__Text` 段的，而是存在于 `__DATA` 段中。位于 `__Text` 段的符号是只读的，而位于 `__DATA` 段的符号是可读可写的

这种懒绑定的方式其实叫做 PIC(Position Independent Code 地址无关代码)，fishhook 能够帮助我们修改这部分符号的地址

### 使用

fishhook 的使用需要创建一个 `rebinding` 结构体，结构体中需要包含要 hook 的函数的名称，要替换的方法实现，被 hook 的方法的容器：

```objc

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
 
    // 定义rebinding 结构体
    struct rebinding rebind = {};
  	// 需要hook的函数名称
    rebind.name = "NSLog";
  	// 新函数的地址
    rebind.replacement = hookNSLog;
  	// 原始函数地址的指针
    rebind.replaced = (void *)&sys_NSLog;
    //将上面的结构体 放入 reb结构体数组中
    struct rebinding rebindObj[]  = {rebind};
    
    /*
     * arg1 : 结构体数据组
     * arg2 : 数组的长度
     */
    rebind_symbols(rebindObj, 1);
}

//定义一个函数指针 用于指向原来的NSLog函数
static void (*sys_NSLog)(NSString *format, ...);

void hookNSLog(NSString *format, ...){
    format = [format stringByAppendingString:@"被勾住了"];
    sys_NSLog(format);
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    funcDlog(@"原有NSLog函数");
}

@end

```

### 源码解析

#### `rebind_symbols` 方法

先从调用方法 `rebind_symbols` 方法入手：

```c
struct rebindings_entry {
    struct rebinding *rebindings;
    size_t rebindings_nel;
    struct rebindings_entry *next;
};

static struct rebindings_entry *_rebindings_head;

int rebind_symbols(struct rebinding rebindings[], size_t rebindings_nel) {
    // 维护一个 rebindings_entry 的结构
    // 将 rebinding 的多个实例组织成一个链表
    int retval = prepend_rebindings(&_rebindings_head, rebindings, rebindings_nel);
    // 判断是否 malloc 失败，失败会返回 -1
    if (retval < 0) {
        return retval;
    }
    // _rebindings_head -> next 是第一次调用的标志符，NULL 则代表第一次调用
    if (!_rebindings_head->next) {
        // 第一次调用，将 _rebind_symbols_for_image 注册为回调
        _dyld_register_func_for_add_image(_rebind_symbols_for_image);
    } else {
        // 先获取 dyld 镜像数量
        uint32_t c = _dyld_image_count();
        for (uint32_t i = 0; i < c; i++) {
            // 根据下标依次进行重绑定过程
            _rebind_symbols_for_image(_dyld_get_image_header(i), _dyld_get_image_vmaddr_slide(i));
        }
    }
    // 返回状态值
    return retval;
}
```

先看 `prepend_rebindings` 方法，它会把传入的 `rebindings` 数组串成一个链表，链表的头部用 `_rebindings_head` 保存：

```c
/**
 * prepend_rebindings 用于 rebindings_entry 结构的维护
 * struct rebindings_entry **rebindings_head - 对应的是 static 的 _rebindings_head
 * struct rebinding rebindings[] - 传入的方法符号数组
 * size_t nel - 数组对应的元素数量
 */
static int prepend_rebindings(struct rebindings_entry **rebindings_head,
                              struct rebinding rebindings[],
                              size_t nel) {
    // 声明 rebindings_entry 一个指针，并为其分配空间
    struct rebindings_entry *new_entry = (struct rebindings_entry *) malloc(sizeof(struct rebindings_entry));
    if (!new_entry) {
        return -1;
    }
    // 为链表中元素的 rebindings 实例分配指定空间
    new_entry->rebindings = (struct rebinding *) malloc(sizeof(struct rebinding) * nel);
    if (!new_entry->rebindings) {
        free(new_entry);
        return -1;
    }
    // 将 rebindings 数组中 copy 到 new_entry -> rebingdings 成员中
    memcpy(new_entry->rebindings, rebindings, sizeof(struct rebinding) * nel);
    // 为 new_entry -> rebindings_nel 赋值
    new_entry->rebindings_nel = nel;
    // 为 new_entry -> next 赋值，维护链表结构
    new_entry->next = *rebindings_head;
    // 移动 head 指针，指向表头
    *rebindings_head = new_entry;
    return 0;
}
```

在进行过了可能的多次 `prepend_rebindings` 方法后，会形成如下链表：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook_11.png?raw=true)

当然由于最开始 `rebindgs_header` 是 null，所以 `next_entry->next = *rebindings_headr` 就是 null，也就是说 `*rebinding_head = new_entry` 后， `*rebinding_head->next` 为 null。在 `rebind_symbolds` 方法中会执行 `_dyld_register_func_for_add_image(_rebind_symbols_for_image);` 方法。

`_dyld_register_func_for_add_image` 方法是 dyld 注册回调函数的方法，当镜像被加载的时候，就会主动触发注册的回调方法。

> 一个可执行文件会加载非常多的动态库，每个动态库的成功加载都会触发注册的回调方法。每个动态库镜像都会根据设置重绑定符号

此处注册了 `_rebind_symbols_for_image`  方法：

```c
static void _rebind_symbols_for_image(const struct mach_header *header,
                                      intptr_t slide) {
    rebind_symbols_for_image(_rebindings_head, header, slide);
}
```

`_rebind_symbols_for_image` 方法非常的朴实，它会受到 dyld 加载成功时候传入的两个参数 `mach_header *header` 和 `intptr_t slide`。这两个参数分别是当前可执行文件的内存地址和 ASLR 偏移量。也就是说 `mach_header *header` 就是通过 `image list` 获取到的地址，如下图：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook_12.png?raw=true)

另外可以发现 `mach_header *header` 和 `intptr_t slide` 相差就是 0x100000000，也侧面印证了 `mach_header *header` 就是 `__Text` 段的实际起始地址

#### `rebind_symbols_for_image` 方法

进入 `rebind_symbols_for_image` 方法，这是一个非常重要的方法，我们可以将其分为几个阶段来看。

##### 获取动态静态符号表位置以及 `linkedit_segment` 的 load command 位置

这一部分其实很简单，就是通过遍历 load commands 找到 `symtab_command` 和 `dysymtab_command` 以及 `linkedit_segment` 的位置：

```c
static void rebind_symbols_for_image(struct rebindings_entry *rebindings, const struct mach_header *header, intptr_t slide) {
		...

    // 声明几个查找量:
    // linkedit_segment, symtab_command, dysymtab_command
    segment_command_t *cur_seg_cmd;
    segment_command_t *linkedit_segment = NULL;
    struct symtab_command* symtab_cmd = NULL;
    struct dysymtab_command* dysymtab_cmd = NULL;

    // 初始化游标
    // header = 0x100000000 + ASLR 偏移
    // sizeof(mach_header_t) = 默认 0x20 (Mach-O 头部大小)
    // 首先需要跳过 Mach-O Header
    uintptr_t cur = (uintptr_t)header + sizeof(mach_header_t);
    // 遍历每一个 Load Command，游标每一次偏移每个命令的 Command Size 大小
    // header -> ncmds: Load Command 加载命令数量
    // cur_seg_cmd -> cmdsize: Load 大小
    for (uint i = 0; i < header->ncmds; i++, cur += cur_seg_cmd->cmdsize) {
        // 取出当前的 Load Command
        cur_seg_cmd = (segment_command_t *)cur;
        // Load Command 的类型是 LC_SEGMENT
        if (cur_seg_cmd->cmd == LC_SEGMENT_ARCH_DEPENDENT) {
            // 比对一下 Load Command 的 name 是否为 __LINKEDIT
            if (strcmp(cur_seg_cmd->segname, SEG_LINKEDIT) == 0) {
                // 检索到 __LINKEDIT
                linkedit_segment = cur_seg_cmd;
            }
        }
        // 判断当前 Load Command 是否是 LC_SYMTAB 类型
        // LC_SEGMENT - 代表当前区域链接器信息
        else if (cur_seg_cmd->cmd == LC_SYMTAB) {
            // 检索到 LC_SYMTAB
            symtab_cmd = (struct symtab_command*)cur_seg_cmd;
        }
        // 判断当前 Load Command 是否是 LC_DYSYMTAB 类型
        // LC_DYSYMTAB - 代表动态链接器信息区域
        else if (cur_seg_cmd->cmd == LC_DYSYMTAB) {
            // 检索到 LC_DYSYMTAB
            dysymtab_cmd = (struct dysymtab_command*)cur_seg_cmd;
        }
    }
}
```

前面提到过，在 Mach-O 加载进内存后，`__Text` 段的起始位置时 0x100000000，并且所有 offset 都是以 `__Text` 段为基准的。`header` 和 `load commands` 也是属于 `__Text` 段的一部分。上面的代码正印证了这个观点。

##### 计算静态符号表和动态符号表以及字符串表的位置

前面拿到了静态符号表以及动态符号表的 load command，现在就可以根据 load command 中的信息计算得到静态符号表,动态符号表以及字符串表的位置了：

```c
static void rebind_symbols_for_image(struct rebindings_entry *rebindings, const struct mach_header *header, intptr_t slide) {
  	...
      
		// slide: ASLR 偏移量
    // vmaddr: SEG_LINKEDIT 的虚拟地址
    // fileoff: SEG_LINKEDIT 地址偏移
    // 式①：base = SEG_LINKEDIT真实地址 - SEG_LINKEDIT地址偏移
    // 式②：SEG_LINKEDIT真实地址 = SEG_LINKEDIT虚拟地址 + ASLR偏移量
    // 将②代入①：Base = SEG_LINKEDIT虚拟地址 + ASLR偏移量 - SEG_LINKEDIT地址偏移
    uintptr_t linkedit_base = (uintptr_t)slide + linkedit_segment->vmaddr - linkedit_segment->fileoff;
    // 通过 base + symtab 的偏移量 计算 symtab 表的首地址，并获取 nlist_t 结构体实例
    nlist_t *symtab = (nlist_t *)(linkedit_base + symtab_cmd->symoff);
    // 通过 base + stroff 字符表偏移量计算字符表中的首地址，获取字符串表
    char *strtab = (char *)(linkedit_base + symtab_cmd->stroff);
    // 通过 base + indirectsymoff 偏移量来计算动态符号表的首地址
    uint32_t *indirect_symtab = (uint32_t *)(linkedit_base + dysymtab_cmd->indirectsymoff);
}
```

这里通过 `linkedit` 计算基地址，所有的偏移量都是以基地址为参照的。计算公式上面也有写到。获取到基地址后，就可以通过前面获取到的静态符号表以及动态符号表的 load commands 中保存的偏移量计算得到符号表 `symtab`，字符串表 `strtab` 以及动态符号表 `indirect_symtab` 的位置了。

##### 跳转重绑定

之后，重新遍历 load commands，获取 `__DATA` 段中的 `__nl_symbol_ptr` 和 `__la_symbol_ptr` 两个 section 的信息，然后执行真正的重绑定方法 `perform_rebinding_with_section`：

```c
static void rebind_symbols_for_image(struct rebindings_entry *rebindings, const struct mach_header *header, intptr_t slide) {
  	...
      
		// 归零游标
    cur = (uintptr_t)header + sizeof(mach_header_t);
    // 再次遍历 Load Commands
    for (uint i = 0; i < header->ncmds; i++, cur += cur_seg_cmd->cmdsize) {
        cur_seg_cmd = (segment_command_t *)cur;
        // Load Command 的类型是 LC_SEGMENT
        if (cur_seg_cmd->cmd == LC_SEGMENT_ARCH_DEPENDENT) {
            // 查询 Segment Name。不是 __DATA 或者 __DATA_CONST 的直接 return
            if (strcmp(cur_seg_cmd->segname, SEG_DATA) != 0 &&
                strcmp(cur_seg_cmd->segname, SEG_DATA_CONST) != 0) {
                continue;
            }
            // 遍历 Segment 中的 Section
            for (uint j = 0; j < cur_seg_cmd->nsects; j++) {
                // 取出 Section
                section_t *sect = (section_t *)(cur + sizeof(segment_command_t)) + j;
                // flags & SECTION_TYPE 通过 SECTION_TYPE 掩码获取 flags 记录类型的 8 bit
                // 如果 section 的类型为 S_LAZY_SYMBOL_POINTERS
                // 这个类型代表 lazy symbol 指针 Section
                if ((sect->flags & SECTION_TYPE) == S_LAZY_SYMBOL_POINTERS) {
                    // 进行 rebinding 重写操作
                    perform_rebinding_with_section(rebindings, sect, slide, symtab, strtab, indirect_symtab);
                }
                // 这个类型代表 non-lazy symbol 指针 Section
                if ((sect->flags & SECTION_TYPE) == S_NON_LAZY_SYMBOL_POINTERS) {
                    perform_rebinding_with_section(rebindings, sect, slide, symtab, strtab, indirect_symtab);
                }
            }
        }
    }
}
```

#### 重绑定

这是执行方法替换的最关键方法，我们先看一下代码：

```c
static void perform_rebinding_with_section(struct rebindings_entry *rebindings, section_t *section, intptr_t slide, nlist_t *symtab, char *strtab, uint32_t *indirect_symtab) {
    // 在 Indirect Symbol table 中检索到 __la_symbol_ptr 或者 __nl_symbol_ptr 起始的位置
    uint32_t *indirect_symbol_indices = indirect_symtab + section->reserved1;
    // 获取 _DATA.__nl_symbol_ptr(或__la_symbol_ptr) Section
    // 已知其 value 是一个指针类型，整段区域用二阶指针来获取
    void **indirect_symbol_bindings = (void **)((uintptr_t)slide + section->addr);
    // 用 size / 一阶指针来计算 _DATA.__nl_symbol_ptr(或__la_symbol_ptr) 中符号的个数，遍历整个 Section
    for (uint i = 0; i < section->size / sizeof(void *); i++) {
        // 通过下标来获取每一个 Indirect Address 的 Value
        // 这个 Value 也是外层寻址时需要的下标
        uint32_t symtab_index = indirect_symbol_indices[i];
        if (symtab_index == INDIRECT_SYMBOL_ABS || symtab_index == INDIRECT_SYMBOL_LOCAL ||
            symtab_index == (INDIRECT_SYMBOL_LOCAL   | INDIRECT_SYMBOL_ABS)) {
            continue;
        }
        // 获取符号名在字符表中的偏移地址
        uint32_t strtab_offset = symtab[symtab_index].n_un.n_strx;
        // 获取符号名
        char *symbol_name = strtab + strtab_offset;
        // 取出 rebindings 结构体实例数组，开始遍历链表
        struct rebindings_entry *cur = rebindings;
        while (cur) {
            // 对于链表中每一个 rebindings 数组的每一个 rebinding 实例
            // 依次在 String Table 匹配符号名
            for (uint j = 0; j < cur->rebindings_nel; j++) {
                // 符号名与方法名匹配
                if (strcmp(&symbol_name[1], cur->rebindings[j].name) == 0) {
                    // 如果是第一次对跳转地址进行重写
                    if (cur->rebindings[j].replaced != NULL &&
                        indirect_symbol_bindings[i] != cur->rebindings[j].replacement) {
                        // 记录原始跳转地址
                        *(cur->rebindings[j].replaced) = indirect_symbol_bindings[i];
                    }
                    // 重写跳转地址
                    indirect_symbol_bindings[i] = cur->rebindings[j].replacement;
                    // 完成后不再对当前 Indirect Symbol 处理
                    // 继续迭代到下一个 Indirect Symbol
                    goto symbol_loop;
                }
            }
            // 链表遍历
            cur = cur->next;
        }
    symbol_loop:;
    }
}
```

##### 获取 `__la_symbol_ptr` 或者 `__nl_symbol_ptr` 的符号在动态符号表的位置

```c
uint32_t *indirect_symbol_indices = indirect_symtab + section->reserved1;
```

`indirect_symtab` 是 Dynamic Symbol Table 段：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook_16.png?raw=true)

它汇集了 `__Text,__stubs` ,`__DATA,__nl_symbol_ptr`,`__DATA,__got`， `__DATA,__la_symbol_ptr` 这几个段的所有动态链接符号的所有信息。

section 就是上面获取到的在 Load Command是中的 `__la_symbol_ptr` 以及 `__nl_symbol_ptr` 段的信息，它的 `reserved1` 就是该段在 Dynamic Symbol Table 中的位置。在 MachOView 中显示为 `Indirect Sym Index`:

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fishhook_17.png?raw=true)

由于 `indirect_symtab` 是 uint32 类型的指针，所以一个指针占用4个字节。因此偏移 `reserved1`即偏移 `reserved1` * 4 的地址。

##### 获取 `__la_symbol_ptr` 或者 `__nl_symbol_ptr`的实际地址

```c
void **indirect_symbol_bindings = (void **)((uintptr_t)slide + section->addr);
```

之前我们说过一个公式：

>  ASLR偏移 + 段的虚拟地址 = 段的实际地址

`section->addr` 就是段的虚拟地址，`indirect_symbol_bindings` 就是指向段的实际地址。由于  `__la_symbol_ptr` 或者 `__nl_symbol_ptr` 段内保存的是一个个指向实际方法的指针，因此 `indirect_symbol_bindings` 就被声明为一个二级指针。

##### 遍历 `__la_symbol_ptr` 或者 `__nl_symbol_ptr`替换符号实现方法

```c
for (uint i = 0; i < section->size / sizeof(void *); i++) {
        uint32_t symtab_index = indirect_symbol_indices[i];
        
        uint32_t strtab_offset = symtab[symtab_index].n_un.n_strx;
        char *symbol_name = strtab + strtab_offset;
        struct rebindings_entry *cur = rebindings;
        while (cur) {
            for (uint j = 0; j < cur->rebindings_nel; j++) {
                if (strcmp(&symbol_name[1], cur->rebindings[j].name) == 0) {
                    if (cur->rebindings[j].replaced != NULL &&
                        indirect_symbol_bindings[i] != cur->rebindings[j].replacement) {
                        *(cur->rebindings[j].replaced) = indirect_symbol_bindings[i];
                    }
                    indirect_symbol_bindings[i] = cur->rebindings[j].replacement;
                    goto symbol_loop;
                }
            }
            cur = cur->next;
        }
    symbol_loop:;
}
```

遍历 `__la_symbol_ptr` 或者 `__nl_symbol_ptr` 是为了将每一个动态符号和要替换的符号方法匹配，那么通过什么匹配的呢？通过符号名。也就是每个循环中的内容。

根据符号在 Dynamic Symbol Tabel 的位置，拿到它在 Symbol Tabel 的索引位置 `symtab_index`，然后在 Symbol Tabel 的相应位置拿到他在 String Tabel 的偏移 `strtab_offset`。获取到偏移后就加上 string table 的基地址，拿到符号的位置，也就获得了符号名 `symbol_name`。

符号名拿到后，就可以遍历自定义的要替换的符号数组 `rebindings`，一一匹配符号名和要替换的符号名是否匹配。如果匹配了，并且没有替换过(替没替换过只要判断 rebindings 结构体的 replaced 是否为空即可)，就让 `indirect_symbol_bindings` 相应的符号指向 rebindings 相应的 replacement 方法的地址。

至此，fishhook 整个替换流程就结束了。

## 总结

到这里你一定被各种跳转和各种偏移绕晕了。下面我来整理一个过程：

### lazy binding 过程

1. lazy binding 的符号指向 `__TEXT,__stubs` ，调用的时候会执行 `__TEXT,__stubs` 指向的方法。它会调用 `__DATA,__la_symbol_ptr` 指向的地址上的方法。
2. 未绑定时，`__DATA,__la_symbol_ptr` 指向的地址为 `___TEXT,stub_helper`，它会执行系统函数修改 `__DATA,__la_symbol_ptr` 的指向。
3. 绑定后， `__DATA,__la_symbol_ptr`  指向实际的函数的地址

### fishhook 替换过程

1. 通过注册系统回调 `_dyld_register_func_for_add_image` 获取 image 的起始地址和 ASLR 偏移。
2. 通过 image 的起始地址，加上 Header 的大小(Header 固定大小为 0x20)，得出 Load Commands 的起始地址
3. 遍历 Load Commands 拿到 `__DATA,__nl_symbol_ptr` 和 `__DATA,__la_symbol_ptr` 的各项信息，包括段的位置，段的大小，段在 Dynamic Symbol Table 的起始索引 `reserved1`。
4. 再次遍历 Load Commands 拿到静态符号表 `LC_SYMTAB`，获取 Symbol Table 和 String Table 的起始位置；拿到动态符号表 `LC_DYSYMTAB`  的起始位置;
5. 根据第四步获取的起始位置，在 `LC_DYSYMTAB` 中遍历 `__DATA,__nl_symbol_ptr` 或者 `__DATA,__la_symbol_ptr` 的各个符号，获取它在 `LC_SYMTAB` 的索引。
6. 根据这个索引，获取该符号在 `LC_SYMTAB`的信息，可以拿到它在 String Table 的 offset。这个 offset 保存着符号的名字。
7. 拿到这个符号的名字和我们要替换的各个符号名做对比，如果相同，那么把 `__DATA,__nl_symbol_ptr` 或者 `__DATA,__la_symbol_ptr` 相应位置的符号指向要替换的方法的地址。至此，fishhook 替换完成

## 参考

[深入理解fishhook](<https://www.jianshu.com/p/828ec78d4ae1>)

[巧用符号表 - 探求 fishhook 原理](<https://www.desgard.com/fishhook-1/>)

[ios逆向 - mach-o文件分析](<https://www.jianshu.com/p/37f10bb70c50>)

[深入理解fishhook](<https://www.jianshu.com/p/828ec78d4ae1>)