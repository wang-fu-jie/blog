---
title:       自制操作系统 - 用户程序加载与执行
subtitle:    "用户程序加载与执行"
description: "用户程序最终是被编译为ELF文件，然后被操作系统加载执行。本节我们将使用系统对ELF文件进行解析，并通过系统调用进行执行。我们并不使用ELF文件的节头，它一般被用于编译器、调试器等。"
excerpt:     "用户程序最终是被编译为ELF文件，然后被操作系统加载执行。本节我们将使用系统对ELF文件进行解析，并通过系统调用进行执行。我们并不使用ELF文件的节头，它一般被用于编译器、调试器等。"
date:        2024-03-15T11:04:04+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/94/2a/smoke_smoking_silhouette_sitting_dark-22193.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-user-program-load"
categories:  [ "操作系统" ]
---

## 一、ELF文件解析
ELF文件是 Executable and Linking Format 的缩写，可以分为三种类型：可执行文件、可重定位文件 .o、共享的目标文件(动态链接库) .so。 可执行文件的内容主要分成三个部分：
* text: 代码段，存储运行时的代码
* data: 数据端，内容不为 0 的数据，需要存储到可执行文件中；
* bss: bss 段，未初始化的数据，内容全部为 0，没有必要存储到文件中，只需要标记占用的大小；

ELF文件有两种视角，分别是链接视图和执行视图。链接视图是从编译器/链接器角度出发，方便将代码组织成可重定位文件或者可执行文件。执行视图，从加载器/运行时角度出发，方便将 ELF 文件加载到内存中执行之。其中包括：其中包括：ELF 文件头、 程序头表：程序加载器使用的内容、 若干段：.text .data .bss、 节头表：编译器/汇编器/链接器使用的内容。

### 1.1、操作系统执行ELF文件
首先我们需要在操作系统中，生成一个ELF文件，并编译后写入到磁盘，源码如下：
```asm
[bits 32]
section .text
global _start

_start:
    ; write(stdout, message , sizeof(message))  这里调用write系统调用
    mov eax, 4; write  4号系统调用               
    mov ebx, 1; stdout  第一个参数1，代表标准输出
    mov ecx, message
    mov edx, message.end - message - 1
    int 0x80

    ; exit(0);
    mov eax, 1; nr_exit
    mov ebx, 0; code
    int 0x80

section .data

message:
    db "hello onix!!!", 10, 0
    .end:

section .bss
buffer: resb 1024; 预留 1024 个字节的空间
```
如上所示，是一个在评估打印字符的功能实现。在makefile中将这段代码编译为并写入磁盘。接下来我们通过readelf读取下这个可执行文件，如下：
```
wfj@wfj:/opt/onix/build/builtin$ readelf -e hello.out 
ELF 头：
  Magic：   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  类别:                              ELF32
  数据:                              2 补码，小端序 (little endian)
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI 版本:                          0
  类型:                              EXEC (可执行文件)
  系统架构:                          Intel 80386
  版本:                              0x1
  入口点地址：               0x1001000
  程序头起点：          52 (bytes into file)
  Start of section headers:          8740 (bytes into file)
  标志：             0x0
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         3
  Size of section headers:           40 (bytes)
  Number of section headers:         11
  Section header string table index: 10

节头：
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        01001000 001000 000022 00  AX  0   0 16
  [ 2] .data             PROGBITS        01002000 002000 00000f 00  WA  0   0  4
  [ 3] .bss              NOBITS          01002010 00200f 000400 00  WA  0   0  4
  [ 4] .debug_aranges    PROGBITS        00000000 00200f 000020 00      0   0  1
  [ 5] .debug_info       PROGBITS        00000000 00202f 00005a 00      0   0  1
  [ 6] .debug_abbrev     PROGBITS        00000000 002089 00001d 00      0   0  1
  [ 7] .debug_line       PROGBITS        00000000 0020a6 000046 00      0   0  1
  [ 8] .symtab           SYMTAB          00000000 0020ec 000090 10      9   5  4
  [ 9] .strtab           STRTAB          00000000 00217c 000046 00      0   0  1
  [10] .shstrtab         STRTAB          00000000 0021c2 000061 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  D (mbind), p (processor specific)

程序头：
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000000 0x01000000 0x01000000 0x00094 0x00094 R   0x1000
  LOAD           0x001000 0x01001000 0x01001000 0x00022 0x00022 R E 0x1000
  LOAD           0x002000 0x01002000 0x01002000 0x0000f 0x00410 RW  0x1000

 Section to Segment mapping:
  段节...
   00     
   01     .text 
   02     .data .bss 
```
ELF文件头标识了文件类型，系统架构、入口地址、程序头表偏移、节头表偏移、程序头数量、节数量等信息。节头标识的是链接视图、描述了编译器和调试器关心的逻辑组织，程序运行时不会直接用节头，只用于编译/链接/调试阶段。例如节头的 .bss 大小可以看到是1024字节（0x400）。程序头描述了内存中加载映射的结构，是操作系统加载 ELF 的关键。程序的加载基地址为 0x01000000；.text 段加载在 0x01001000；.data + .bss 段加载在 0x01002000；.bss 的文件中大小为 0（NOBITS），但内存中保留了空间。

这里我们在makefile指定了程序加载的地址为0x01001000，是因为我们的内存布局 0x01000000 是代码段开始的位置，但是第一页需要存放ELF文件头和节头。

### 1.2、读取ELF文件
在生成了可执行的ELF文件后，操作系统需要读取、加载并执行ELF文件，首先需要定义ELF文件头格式，如下：
```cpp
typedef u32 Elf32_Word;
typedef u32 Elf32_Addr;
typedef u32 Elf32_Off;
typedef u16 Elf32_Half;

// ELF 文件标记
typedef struct ELFIdent
{
    u8 ei_magic[4];    // 内容为 0x7F, E, L, F
    u8 ei_class;       // 文件种类 1-32位，2-64 位
    u8 ei_data;        // 标记大小端 1-小端，2-大端
    u8 ei_version;     // 与 e_version 一样，必须为 1
    u8 ei_pad[16 - 7]; // 占满 16 个字节
} _packed ELFIdent;

// ELF 文件头
typedef struct Elf32_Ehdr
{
    ELFIdent e_ident;       // ELF 文件标记，文件最开始的 16 个字节
    Elf32_Half e_type;      // 文件类型，见 Etype
    Elf32_Half e_machine;   // 处理器架构类型，标记运行的 CPU，见 EMachine
    Elf32_Word e_version;   // 文件版本，见 EVersion
    Elf32_Addr e_entry;     // 程序入口地址
    Elf32_Off e_phoff;      // program header offset 程序头表在文件中的偏移量
    Elf32_Off e_shoff;      // section header offset 节头表在文件中的偏移量
    Elf32_Word e_flags;     // 处理器特殊标记
    Elf32_Half e_ehsize;    // ELF header size ELF 文件头大小
    Elf32_Half e_phentsize; // program header entry size 程序头表入口大小
    Elf32_Half e_phnum;     // program header number 程序头数量
    Elf32_Half e_shentsize; // section header entry size 节头表入口大小
    Elf32_Half e_shnum;     // section header number 节头表数量
    Elf32_Half e_shstrndx;  // 节字符串表在节头表中的索引
} Elf32_Ehdr;

// ELF 文件类型
enum Etype
{
    ET_NONE = 0,        // 无类型
    ET_REL = 1,         // 可重定位文件
    ET_EXEC = 2,        // 可执行文件
    ET_DYN = 3,         // 动态链接库
    ET_CORE = 4,        // core 文件，未说明，占位
    ET_LOPROC = 0xff00, // 处理器相关低值
    ET_HIPROC = 0xffff, // 处理器相关高值
};

// ELF 机器(CPU)类型
enum EMachine
{
    EM_NONE = 0,  // No machine
    EM_M32 = 1,   // AT&T WE 32100
    EM_SPARC = 2, // SPARC
    EM_386 = 3,   // Intel 80386
    EM_68K = 4,   // Motorola 68000
    EM_88K = 5,   // Motorola 88000
    EM_860 = 7,   // Intel 80860
    EM_MIPS = 8,  // MIPS RS3000
};

// ELF 文件版本
enum EVersion
{
    EV_NONE = 0,    // 不可用版本
    EV_CURRENT = 1, // 当前版本
};
```
定义完结构，就可以来读取ELF文件了。
```cpp
int sys_execve(char *filename, char *argv[], char *envp[])
{
    fd_t fd = open(filename, O_RDONLY, 0);
    if (fd == EOF)
        return EOF;

    Elf32_Ehdr *ehdr = (Elf32_Ehdr *)alloc_kpage(1);   // ELF 文件头
    lseek(fd, 0, SEEK_SET);
    read(fd, (char *)ehdr, sizeof(Elf32_Ehdr));

    LOGK("ELF ident %s\n", ehdr->e_ident.ei_magic);
    LOGK("ELF class %d\n", ehdr->e_ident.ei_class);
    ... // 一些打印。

    Elf32_Phdr *phdr = (Elf32_Phdr *)alloc_kpage(1);   // 段头
    lseek(fd, ehdr->e_phoff, SEEK_SET);
    read(fd, (char *)phdr, ehdr->e_phentsize * ehdr->e_phnum);
    LOGK("ELF segment size mem %d\n", ehdr->e_phentsize * ehdr->e_phnum);

    Elf32_Phdr *ptr = phdr;
    char *content = (char *)alloc_kpage(1);       // 内容
    for (size_t i = 0; i < ehdr->e_phnum; i++)
    {
        memset(content, 0, PAGE_SIZE);
        lseek(fd, ptr->p_offset, SEEK_SET);
        read(fd, content, ptr->p_filesz);
        LOGK("segment vaddr 0x%p paddr 0x%p\n", ptr->p_vaddr, ptr->p_paddr);
        ptr++;
    }

    free_kpage((u32)ehdr, 1);
    free_kpage((u32)phdr, 1);
    free_kpage((u32)content, 1);
    return 0;
}
```
我们通过一个系统调用 execve 来读取 ELF文件，execve系统调用的实现如上所示。我们这里进行了ELF文件的读取和解析，理论跳转到代码段即可执行程序，但是此时我们还处于内核态，最终执行代码是要切换回用户态的。


## 二、ELF符号解析
为了支持一些语言特性，比如 C++ 的异常机制，需要在运行时存储了一些调用帧信息，以便在发生异常是使用，这些信息通常存储在 eh_frame , eh_framehdr 节中。其中的信息与 .debug_frame 的信息类似，是 DWARF 6格式的信息。节头中还有符号表，如下为节头的结构：
```cpp
typedef struct Elf32_Shdr
{
    Elf32_Word sh_name;      // 节名
    Elf32_Word sh_type;      // 节类型，见 SectionType
    Elf32_Word sh_flags;     // 节标记，见 SectionFlag
    Elf32_Addr sh_addr;      // 节地址
    Elf32_Off sh_offset;     // 节在文件中的偏移量
    Elf32_Word sh_size;      // 节大小
    Elf32_Word sh_link;      // 保存了头表索引链接，与节类型相关
    Elf32_Word sh_info;      // 额外信息，与节类型相关
    Elf32_Word sh_addralign; // 地址对齐约束
    Elf32_Word sh_entsize;   // 子项入口大小
} Elf32_Shdr;

enum SectionType
{
    SHT_NULL = 0,            // 不可用
    SHT_PROGBITS = 1,        // 程序信息
    SHT_SYMTAB = 2,          // 符号表
    SHT_STRTAB = 3,          // 字符串表
    SHT_RELA = 4,            // 有附加重定位
    SHT_HASH = 5,            // 符号哈希表
    SHT_DYNAMIC = 6,         // 动态链接信息
    SHT_NOTE = 7,            // 标记文件信息
    SHT_NOBITS = 8,          // 该节文件中无内容
    SHT_REL = 9,             // 无附加重定位
    SHT_SHLIB = 10,          // 保留，用于非特定的语义
    SHT_DYNSYM = 11,         // 符号表
    SHT_LOPROC = 0x70000000, // 以下与处理器相关
    SHT_HIPROC = 0x7fffffff,
    SHT_LOUSER = 0x80000000,
    SHT_HIUSER = 0xffffffff,
};

enum SectionFlag
{
    SHF_WRITE = 0x1,           // 执行时可写
    SHF_ALLOC = 0x2,           // 执行时占用内存，有些节执行时可以不在内存中
    SHF_EXECINSTR = 0x4,       // 包含可执行的机器指令，节里有代码
    SHF_MASKPROC = 0xf0000000, // 保留，与处理器相关
};

/////    符号表
typedef struct Elf32_Sym
{
    Elf32_Word st_name;  // 符号名称，在字符串表中的索引
    Elf32_Addr st_value; // 符号值，与具体符号相关
    Elf32_Word st_size;  // 符号的大小
    u8 st_info;          // 指定符号类型和约束属性，见 SymbolBinding
    u8 st_other;         // 为 0，无意义
    Elf32_Half st_shndx; // 符号对应的节表索引
} Elf32_Sym;

// 通过 info 获取约束
#define ELF32_ST_BIND(i) ((i) >> 4)
// 通过 info 获取类型
#define ELF32_ST_TYPE(i) ((i)&0xF)
// 通过 约束 b 和 类型 t 获取 info
#define ELF32_ST_INFO(b, t) (((b) << 4) + ((t)&0xf))

// 符号约束
enum SymbolBinding
{
    STB_LOCAL = 0,   // 外部不可见符号，优先级最高
    STB_GLOBAL = 1,  // 外部可见符号
    STB_WEAK = 2,    // 弱符号，外部可见，如果符号重定义，则优先级更低
    STB_LOPROC = 13, // 处理器相关低位
    STB_HIPROC = 15, // 处理器相关高位
};

// 符号类型
enum SymbolType
{
    STT_NOTYPE = 0,  // 未定义
    STT_OBJECT = 1,  // 数据对象，比如 变量，数组等
    STT_FUNC = 2,    // 函数或可执行代码
    STT_SECTION = 3, // 节，用于重定位
    STT_FILE = 4,    // 文件，节索引为 SHN_ABS，约束为 STB_LOCAL，
                     // 而且优先级高于其他 STB_LOCAL 符号
    STT_LOPROC = 13, // 处理器相关
    STT_HIPROC = 15, // 处理器相关
};
```
这里在execve的系统调用了也读取一下节头并进行打印操作。


## 三、程序的加载与执行
完成了ELF文件的解析，就可以将用户程序加载到用户内存并执行。当前内存布局如下：
![图片加载失败](/post_images/os/{{< filename >}}/3-01.png)
因此用户程序的ELF头加载到0x1000000的位置，我们看代码实现，首先在shell中内建exec命令，用来执行ELF文件。
```cpp
void builtin_exec(int argc, char *argv[])
{
    if (argc < 2)
    {
        return;
    }

    int status;
    pid_t pid = fork();   // fork 一个子进程来执行用户程序
    if (pid)
    {
        pid_t child = waitpid(pid, &status);   // 父进程waitd等待子进程的执行
        printf("wait pid %d status %d %d\n", child, status, time());
    }
    else
    {
        int i = execve(argv[1], NULL, NULL);    // execve 除非文件不合法，否则不会返回
        exit(i);
    }
}
```
如上索索，当我们的在操作系统shell中执行 exec /hello.out 时。会获取到执行的文件，下发给 execve 系统调用来执行ELF文件。
```cpp
int sys_execve(char *filename, char *argv[], char *envp[])
{
    inode_t *inode = namei(filename);    // 获取ELF文件的inode
    int ret = EOF;
    if (!inode)   // 判断inode是否有效
        goto rollback;
    if (!ISFILE(inode->desc->mode))     //判断inode是否是普通文件
        goto rollback;
    if (!permission(inode, P_EXEC))     // 判断文件是否有执行权限
        goto rollback;

    task_t *task = running_task();       // 获取正在运行的进程
    strncpy(task->name, filename, TASK_NAME_LEN);   // 拷贝文件名给进程名

    task->end = USER_EXEC_ADDR;    // 首先释放原程序的堆内存， 也就是父进程的堆内存，因为子进程这里是从父进程fork来的。
    sys_brk(USER_EXEC_ADDR);

    u32 entry = load_elf(inode);     // 加载程序， 加载到用户内存。
    if (entry == EOF)
        goto rollback;

    sys_brk((u32)task->end);      // 设置堆内存地址
    iput(task->iexec);
    task->iexec = inode;

    intr_frame_t *iframe = (intr_frame_t *)((u32)task + PAGE_SIZE - sizeof(intr_frame_t));
    iframe->eip = entry;
    iframe->esp = (u32)USER_STACK_TOP;
    // ROP 技术，直接从中断返回  通过 eip 跳转到 entry 执行
    asm volatile(
        "movl %0, %%esp\n"
        "jmp interrupt_exit\n" ::"m"(iframe));

rollback:
    iput(inode);
    return ret;
}
```
可以看到在 sys_execve 通过函数 load_elf 加载程序， load_elf中会对第一页做映射，并读取文件头和程序端头表。ELF文件加载到内存之后，通过ROP技术跳转到ELF文件的代码段去执行。执行完后用户代码调用了exit()系统调用进行进程退出，主动释放相关资源，并交出CPU执行权。

最后说明一下。内存布局，ELF、.data、.txt、.bss、堆内存、用户映射内存、 用户态栈。这几个都属于用户态内存，每个用户进程都是独立的，在使用时进行映射。堆内存使用brk系统调用申请， 用户映射内存使用mmap系统调用申请。都是用户态可以申请进行使用的。实际上，glibc 的 malloc 同时使用两者：小内存（通常 <128KB） → 用 brk() 从堆分配；大内存（>128KB） → 用 mmap() 分配匿名映射。这里我们就可以对应C语言的内存模型了。局部变量位于栈，函数调用通过栈进行变量传递。