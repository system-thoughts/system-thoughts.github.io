---
title: 程序员的自我修养 -- ELF剖析
date: 2022-07-14T19:24:32+08:00
tags: ELF
categories: compilation
---
编译生成的中间目标文件、可执行文件都是按照特定的目标文件格式进行组织。各个系统的目标文件格式不太一样，如Unix a.out格式、Windows可移植可执行程序(Portable Executable, PE)、MacOS-X使用Mach-O格式。现代Linux/Unix都是用可执行可链接格式(Eexcutable Linkable Format, ELF)。PE和ELF格式都是COFF(Common File Format)格式的变种。
Linux上不仅中间目标文件、可执行文件采用ELF格式。采用ELF格式的文件类型可分为下表中的四类：
| ELF文件类型 | 说明 | 举例 |
| ---- | ----  | ---- |
| 可重定位文件</br>(Relocatable File) | 编译产生的中间目标文件，可以被用来链接成可执行文件或者共享目标文件，静态链接库也属于这一类 | Linux的.o文件</br>Windows中的.obj文件 |
| 可执行文件</br>(Executable File) | 静态/动态链接的可执行程序 | Linux中的可执行程序无后缀</br>Windows中的.exe |
| 共享目标文件</br>(Shared Object File) | 链接器可以使用这类文件和其他可重定位文件和共享目标文件链接生成新的共享目标文件；与可执行文件进行动态链接，作为进程映像的一部分 | Linux的.so</br>Windows的DLL |
| 核心转储文件</br>(Core Dump File) | 程序意外终止时，系统保留进程地址空间的内容以及终止时的一些其他信息到核心转储文件 | Linux下的core dump |

通过`file`命令查看相应的文件格式，上面的几种文件在file命令中会显示出相应类型：
```bash
# file hello.o
hello.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
# file hello
hello: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=5c4bc53a7f316d2e95cbb40b395ff72b2f1e87fb, not stripped
# file /lib64/ld-2.28.so 
/lib64/ld-2.28.so: ELF 64-bit LSB shared object, x86-64, version 1 (GNU/Linux), dynamically linked, BuildID[sha1]=523b1c63a3f4b12da75660bf483d63560694d81f, not stripped
# file core-test_c-11-0-0-98197-1658125608 
core-test_c-11-0-0-98197-1658125608: ELF 64-bit LSB core file, x86-64, version 1 (SYSV), SVR4-style, from './test_c', real uid: 0, effective uid: 0, real gid: 0, effective gid: 0, execfn: './test_c', platform: 'x86_64'
```

ELF文件格式有两种不同的视角：参与链接的`Linking view`；加载、运行ELF文件的`Execution view`。
![](ELF_format.png)

ELF文件头部是`ELF header`，作为`road map`描述ELF文件组织。链接视角下，ELF文件中的信息按照`section`来组织；执行视角下，ELF文件中的信息按照`segment`来组织。一般来说，将`section`翻译成“节”，将`segment`翻译成“段”，但是在处理ELF文件过程中，往往将二者都翻译成“段”，可以根据实际的视角上下文，便能区分出“段”指的是`section`还是`segment`。不论`section`还是`segment`，都是ELF将相同属性的数据组织到一起的方式。

我们以`simpleElf.c`编译出的可重定位文件`simpleElf.o`开启ELF结构剖析之旅。
```c
# cat simpleElf.c 
int printf(const char *format, ...);

int global_init_var = 8;
int global_uninit_var;

void func(int i)
{
	printf("hello %d\n", i);
}

int main(void)
{
	static int static_init_var = 4;
	static int static_uninit_var;

	int a = 1;
	int b;

	func(static_init_var + static_uninit_var + a + b);
	return 0;
}
# gcc -c simpleElf.c -o simpleElf.o
```

## ELF header
ELF header结构体及相关参数定义在`/usr/include/elf.h`，由于ELF文件在32bit和64bit下都通用，因此存在32bit版本和64bit版本，两个版本的内容一致，只是某些成员大小不一样。`elf.h`使用typedef定义了一套自己的变量体系，如表所示：
| 自定义类型 | 描述 | 原始类型 | 长度(字节) |
| --- | --- | --- | --- |
| Elf32_Half | 32bit版本的无符号短整形 | uint16_t | 2 |
| Elf32_Word | 32bit版本无符号整形 | uint32_t | 4 |
| Elf32_Sword | 32bit版本有符号整形 | int32_t | 4 |
| Elf32_Xword | 32bit版本64bit无符号整形 | uint64_t | 8 |
| Elf32_Sxword | 32bit版本64bit有符号整形 | int64_t | 8 |
| Elf32_Addr | 32bit版本地址 | uint32_t | 4 |
| Elf32_Off | 32bit版本偏移 | uint32_t | 4 |
| Elf64_Half | 64bit版本的无符号短整形 | uint16_t | 2 |
| Elf64_Word | 64bit版本无符号整形 | uint32_t | 4 |
| Elf64_Sword | 64bit版本有符号整形 | int32_t | 4 |
| Elf64_Xword | 64bit版本64bit无符号整形 | uint64_t | 8 |
| Elf64_Sxword | 64bit版本64bit有符号整形 | int64_t | 8 |
| Elf64_Addr | 64bit版本地址 | uint64_t | 8 |
| Elf64_Off | 64bit版本偏移 | uint64_t | 8 |

本文以64bit版本的相关数据结构进行介绍，相应ELF header的数据结构是`Elf64_Ehdr`：
```c
#define EI_NIDENT (16)

typedef struct
{
  unsigned char e_ident[EI_NIDENT];     /* Magic number and other info */
  Elf64_Half    e_type;                 /* Object file type */
  Elf64_Half    e_machine;              /* Architecture */
  Elf64_Word    e_version;              /* Object file version */
  Elf64_Addr    e_entry;                /* Entry point virtual address */
  Elf64_Off     e_phoff;                /* Program header table file offset */
  Elf64_Off     e_shoff;                /* Section header table file offset */
  Elf64_Word    e_flags;                /* Processor-specific flags */
  Elf64_Half    e_ehsize;               /* ELF header size in bytes */
  Elf64_Half    e_phentsize;            /* Program header table entry size */
  Elf64_Half    e_phnum;                /* Program header table entry count */
  Elf64_Half    e_shentsize;            /* Section header table entry size */
  Elf64_Half    e_shnum;                /* Section header table entry count */
  Elf64_Half    e_shstrndx;             /* Section header string table index */
} Elf64_Ehdr;
```
使用`readelf -h`读取ELF文件的ELF header信息：
```bash
# readelf -h simpleElf.o
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          1072 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         13
  Section header string table index: 12
  ```
ELF格式可以支持32bit/64bit处理器，也支持不同字节序(大端/小端)处理器。ELF header开头的`ELF Identification`(`e_ident`)字段指定了底层硬件信息：
![](e_ident.png)
最开始的4字节是ELF文件的标识码，也叫做`magic number`，分别是0x7f、0x45、0x4c、0x46，第一个字节对应的是ASCII字符集中的`DEL`控制符，其余三个字节刚好是ELF这三个字母的ASCII编码。

字节`e_ident[EI_CLASS]`表示`file class`，区分ELF文件是64bit还是32bit。1表示32bit，2表示64bit。
字节`e_ident[EI_DATA]`表示数据编码方式，1表示补码的小端表示，2表示补码的大端表示。
字节`e_ident[EI_VERSION]`表示ELF文件主版本，当前为1，即`EV_CURRENT`，若ELF规范继续演进，不排除`EV_CURRENT`会取更大的值，该字节和后面的`e_version`字段一致。
字节`e_ident[EI_VERSION]`、`e_ident[EI_VERSION]`表示OS/ABI ELF扩展，0表示未定义，看来默认遵循Unix System V ABI标准。

接下来的`e_type`字段区分ELF文件类型：
![](e_type.png)

`e_machine`字段指定了ELF文件的平台架构属性，该字段的详细取值范围参考[1]，`EM_X86_64`表示x86_64平台、`EM_AARCH64`表示ARM64平台。
`e_entry`字段指定的是系统加载ELF文件后，系统跳转执行的第一条命令，可以理解为程序的入口。可重定位文件不会被加载执行，所以此处为0。
`e_flags`字段指定ELF文件处理器相关的flag，当前没有flag被定义，此处为0。
`e_ehsize`字段保存ELF header的大小，可见simpleElf.o的ELF header大小为64 byte。
`e_shstrndx`字段表示section header string table在section header table中的索引，section header string table是段名表：
```bash
# readelf -p .shstrtab simpleElf.o

String dump of section '.shstrtab':
  [     1]  .symtab
  [     9]  .strtab
  [    11]  .shstrtab
  [    1b]  .rela.text
  [    26]  .data
  [    2c]  .bss
  [    31]  .rodata
  [    39]  .comment
  [    42]  .note.GNU-stack
  [    52]  .rela.eh_frame
```
第一列表示段名在`section header string table`数组中的索引，第二列表示相应的段名。`section header string table`数组中的字符串以'\0'分隔，第0位也是'\0'：
| offset | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| +0 | \0 | . | s | y | m | t | a | b | \0 | . |
| +10 | s | t | r | t | a | b | \0 | . | s | h |

`readelf -p .shstrtab simpleElf.o`显示的`.shstrtab`段的内容中包含10个段的名字，少了两个段`.text`和`.eh_frame`，是不是工具的问题呢？并不是，这两个段的名称复用了".rela.text"、".rela.eh_frame"字符串，`.text`和`.eh_frame`的section header的`sh_name`字段指向的是上述段名中间的"."的偏移🙃

## section header table
ELF header中有三个字段与section header table相关：
* `e_shoff`：section header table在文件中的偏移(单位：byte)，如果不存在section header table，则为0
* `e_shentsize`：section header table表项大小(单位：byte)
* `e_shnum`：section header table表项数目，如果不存在section header table，此项为0

显而易见，section header table是数组类型，数组项(entry)描述相应的section，section header table entry类型的数据结构`Elf64_Shdr`，也称作段描述符：
```c
typedef struct
{
  Elf64_Word    sh_name;                /* Section name (string tbl index) */
  Elf64_Word    sh_type;                /* Section type */
  Elf64_Xword   sh_flags;               /* Section flags */
  Elf64_Addr    sh_addr;                /* Section virtual addr at execution */
  Elf64_Off     sh_offset;              /* Section file offset */
  Elf64_Xword   sh_size;                /* Section size in bytes */
  Elf64_Word    sh_link;                /* Link to another section */
  Elf64_Word    sh_info;                /* Additional section information */
  Elf64_Xword   sh_addralign;           /* Section alignment */
  Elf64_Xword   sh_entsize;             /* Entry size if section holds table */
} Elf64_Shdr;
```
使用readelf查看section header table:
```bash
# readelf -S simpleElf.o
There are 13 section headers, starting at offset 0x430:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       0000000000000057  0000000000000000  AX       0     0     1
  [ 2] .rela.text        RELA             0000000000000000  00000320
       0000000000000078  0000000000000018   I      10     1     8
  [ 3] .data             PROGBITS         0000000000000000  00000098
       0000000000000008  0000000000000000  WA       0     0     4
  [ 4] .bss              NOBITS           0000000000000000  000000a0
       0000000000000004  0000000000000000  WA       0     0     4
  [ 5] .rodata           PROGBITS         0000000000000000  000000a0
       000000000000000a  0000000000000000   A       0     0     1
  [ 6] .comment          PROGBITS         0000000000000000  000000aa
       000000000000002d  0000000000000001  MS       0     0     1
  [ 7] .note.GNU-stack   PROGBITS         0000000000000000  000000d7
       0000000000000000  0000000000000000           0     0     1
  [ 8] .eh_frame         PROGBITS         0000000000000000  000000d8
       0000000000000058  0000000000000000   A       0     0     8
  [ 9] .rela.eh_frame    RELA             0000000000000000  00000398
       0000000000000030  0000000000000018   I      10     8     8
  [10] .symtab           SYMTAB           0000000000000000  00000130
       0000000000000180  0000000000000018          11    11     8
  [11] .strtab           STRTAB           0000000000000000  000002b0
       000000000000006c  0000000000000000           0     0     1
  [12] .shstrtab         STRTAB           0000000000000000  000003c8
       0000000000000061  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
```
section header table的第一项是无效的段描述符，默认值为0。第一项的`sh_size`字段在一种情况下也是有意义的：ELF文件段数目大于等于`SHN_LORESERVE`(0xff00)时，ELF header中的`e_shnum`字段存储`SHN_UNDEF`(0)，实际的段数目保存在无效段的`sh_size`字段中。由于`e_shnum`字段是`Elf64_Half`类型，`sh_size`字段的类型为`Elf64_Xword`类型，能够保存的数据范围大的多。这种实际索引超过当前字段类型的表示范围时，ELF规范会在当前字段存放保留值(reserved value)，并不表示实际大小。如符号表项的`st_shndx`成员，ELF header中的`e_shstrndx`：当.shstrtab字符串表中的索引大于等于`SHN_LORESERVE`(0xff00)，`e_shstrndx`保存`SHN_XINDEX`(0xffff)，实际的索引保存在无效段的`sh_link`字段中。

section header table entry各个字段含义如下表所示:
| Name | Description |
| --- | --- |
| sh_name | 段名，存储在.shstrtab字符串表中的偏移 |
| sh_type | 段类型，见[Section type章节](#Section-type) |
| sh_flags | 段的标志位，见[Section flag章节](#Section-flag) |
| sh_addr | 若该段被加载，字段表示段在进程虚拟地址空间的起始地址，否则为0。由于可充定位目标文件不会被加载，所有段的该字段都为0 |
| sh_offset | 该段在ELF文件中的偏移，SHT_NOBITS段不占文件空间，该字段是个概念值，无意义 |
| sh_size | 段的文件长度，即便SHT_NOBITS段不占文件空间，也会有个非0值 |
| sh_link | 保存section header table entry索引，取决于段类型，见[sh_link and sh_info章节](sh_link-and-sh_info) |
| sh_info | 和sh_link一起使用，取决于段类型，见[sh_link and sh_info章节](sh_link-and-sh_info) |
| sh_addralign | 段地址对齐(2^sh_addralign)，取值0或1，例如该段无对齐要求。假设该段最开始是doule变量，则该段必须按照8字节对齐 |
| sh_entsize | 有些段是由固定大小的项组成，比如符号表。sh_entsize表示每项大小，0表示该段并不包含固定大小的项 |

### Section type
下表列举了`simpleElf.o`的段类型，更为详细的描述参考[2]：
| Name | Value | Description |
| --- | --- | --- |
| SHT_NULL | 0 | 无效段 |
| SHT_PROGBITS | 1 | 段保存的信息由程序定义，如代码段、数据段 |
| SHT_SYMTAB\|SHT_DYNSYM | 2 |  符号表 |
| SHT_STRTAB | 3 | 字符串表 |
| SHT_RELA | 4 | 重定位表，该段包含重定位信息 |
| SHT_NOBITS | 8 | 段在ELF文件中不占存储空间，其他方面类似于SHT_PROGBITS段，如.bss段 |

SHT_SYMTAB和SHT_DYNSYM同作为符号表类型，区别如下:
> These sections hold a symbol table. Currently, an object file may have only one section of each type, but this restriction may be relaxed in the future. Typically, SHT_SYMTAB provides symbols for link editing, though it may also be used for dynamic linking. As a complete symbol table, it may contain many symbols unnecessary for dynamic linking. Consequently, an object file may also contain a SHT_DYNSYM section, which holds a minimal set of dynamic linking symbols, to save space.

简要总结下：
* SHT_SYMTAB包含的符号比较全，包含静态链接(link editing)和动态链接(dynamic linking)
* SHT_DYNSYM包含的符号用于动态链接(dynamic linking)，符号表较小

### Section flag
下表列举了`simpleElf.o`的段的标志位，`readelf -S`输出的最后一行是各flag的简称。更为详细的描述参考[2]：
| Name | Value | Description |
| --- | --- | --- |
| SHF_WRITE | 0x1 | 该段在进程地址空间中可写 |
| SHF_ALLOC | 0x2 | 该段会在进程地址空间中分配空间，有些包含指示或者控制信息的段无需在进程地址空间中分配空间。代码段、数据段、.bss段会包含这个flag |
| SHF_EXECINSTR | 0x4 | 该段在进程地址空间中可执行 |
| SHF_MERGE | 0x10 | 仅有可重定位目标文件SHT_PROGBITS类型的section可以使用该flag，用于静态链接时合并相同数据[3] |
| SHF_STRINGS | 0x20 | 该段包含以空字符结尾的字符串，空字符只能作为字符串结束符，而不能出现在任何字符串的中间位置 |
| SHF_INFO_LINK | 0x40 | 该段对应的section header table entry的sh_info字段保存了section header table index | 

> The data in the section may be merged to eliminate duplication. Unless the SHF_STRINGS flag is also set, the data elements in the section are of a uniform size. The size of each element is specified in the section header's sh_entsize field. If the SHF_STRINGS flag is also set, the data elements consist of null-terminated character strings. The size of each character is specified in the section header's sh_entsize field.
>  Each element in the section is compared against other elements in sections with the same name, type and flags.

GNU ld当前仅支持`SHF_MERGE|SHF_STRINGS`都设置的段的合并。

### sh_link and sh_info
`sh_link`与`sh_info`存储的内容取决于section类型：
![](sh_link.png)

由上表可见，`sh_link`和`sh_info`存储的都是section header index，只是根据section类型，存储的是不同的section的索引。对于，不在上表的section type，其`sh_link`和`sh_info`字段为`SHN_UNDEF`(0)。

## 符号
链接过程中，目标文件之间相互拼合实际上是目标文件之间对地址的引用，即对函数和变量的引用。每个函数或者变量都有自己独特的名字，才能避免链接过程中不同变量和函数之间的混淆。在链接过程中，我们将变量和函数统称为符号(Symbol)，函数名或者变量名就是符号名(Symbol Name)。

链接过程很关键的一部分是符号的管理，每一个目标文件都会有一个相应的符号表(symbol table)，这个表中记录了目标文件用到的所有符号。每个定义的符号有一个对应的值，叫做符号值(symbol value)，对于变量和函数来说，符号值就是它们的地址。使用`readelf -s`查看ELF文件的符号表：
```bash
# readelf -s simpleElf.o

Symbol table '.symtab' contains 16 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS simpleElf.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
     6: 0000000000000004     4 OBJECT  LOCAL  DEFAULT    3 static_init_var.1965
     7: 0000000000000000     4 OBJECT  LOCAL  DEFAULT    4 static_uninit_var.1966
     8: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 
     9: 0000000000000000     0 SECTION LOCAL  DEFAULT    8 
    10: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
    11: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    3 global_init_var
    12: 0000000000000004     4 OBJECT  GLOBAL DEFAULT  COM global_uninit_var
    13: 0000000000000000    34 FUNC    GLOBAL DEFAULT    1 func
    14: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND printf
    15: 0000000000000022    53 FUNC    GLOBAL DEFAULT    1 main
```
符号表项的数据结构`Elf64_Sym`：
```c
typedef struct
{
  Elf64_Word    st_name;                /* Symbol name (string tbl index) */
  unsigned char st_info;                /* Symbol type and binding */
  unsigned char st_other;               /* Symbol visibility */
  Elf64_Section st_shndx;               /* Section index */
  Elf64_Addr    st_value;               /* Symbol value */
  Elf64_Xword   st_size;                /* Symbol size */
} Elf64_Sym;
```
符号表项的各个字段含义如下：
| Name | Description |
| --- | --- |
| st_name | 符号名在字符串表.strtab中的索引 |
| st_info | [符号类型与绑定属性](#符号类型与绑定属性) |
| st_other | 符号的可视性(visibility)，C语言编译的重定位文件/动态库/可执行文件，该项都是STV_DEFAULT |
| st_shndx | 符号所在段在section header table的索引，参考[符号所在段](#符号所在段) |
| st_value | 符号对应的值与符号有关，参考[符号值](#符号值) |
| st_size | 符号大小，对于包含数据的符号，如数据对象或者函数，表示该对象的大小。大小为0，表示符号不存在大小或者大小未知 |

可见，符号表中的第一个符号，即下标为0的符号，永远是一个未定义的符号。

### 符号类型与绑定属性
成员低4位表示符号类型(symbol type)，符号高4位表示符号绑定属性(symbol binding)，相应提取/组合的C代码：
```c
#define ELF64_ST_BIND(i)   ((i)>>4)
#define ELF64_ST_TYPE(i)   ((i)&0xf)
#define ELF64_ST_INFO(b,t) (((b)<<4)+((t)&0xf))
```
符号类型如下表所示，完整的symbol type见[4]
| Name | Value | Description |
| --- | --- | --- |
| STT_NOTYPE | 0 | 未知类型符号 |
| STT_OBJECT | 1 | 符号关联的是数据对象，如变量或者数组等 |
| STT_FUNC | 2 | 符号关联的是函数或者其他可执行代码 |
| STT_SECTION | 3 | 符号关联的是段，绑定属性必须是STB_LOCAL，用于重定位场景，能够通过此种形式制定符号：“修改段xxx偏移yyy处的值”[5] |
| STT_FILE | 4 | 符号关联的是当前ELF文件的源文件名，该符号的绑定属性必须是STB_LOCAL，并且它的st_shndx一定是SHN_ABS |
| STT_COMMON | 5 | 该符号标记一个未初始化的公共块 |

符号绑定属性如下表所示，完整的symbol binding见[4]
| Name | Value | Description |
| --- | --- | --- |
| STB_LOCAL | 0 | 局部符号，仅在定义符号的目标文件中可见 |
| STB_GLOBAL | 1 | 全局符号，所有目标文件都可见 |
| STB_WEAK | 2 | 弱符号类似于全局符号，当有同名的弱符号和全局符号，全局符号的优先级更高 |

### 符号所在段
如果符号在当前ELF文件中，`st_shndx`存储符号在section header table中的索引，对于一些特殊符号，`st_shndx`的取值有些特殊：
* SHN_ABS(0xfff1): 简写为`ABS`，表示符号包含了一个绝对值，不会因为重定位改变。源文件名符号就是该类型
* SHN_COMMON(0xfff2): 简写为`COM`，表示符号是尚未分配的公共块(commom block)，该符号值中存储的是对齐约束，链接器将按照`st_value`对齐分配地址，SHN_COMMON类型的符号仅出现在可重定位文件中
* SHN_UNDEF(0): 简写为`UND`,该类型的符号在本ELF文件中引用，定义在其他目标文件中
* SHN_XINDEX(0xffff)：表示索引值太大，实际的section header table索引存储在SHT_SYMTAB_SHNDX段中

SHT_SYMTAB_SHNDX段的结构：
> The section is an array of Elf32_Word values. Each value corresponds one to one with a symbol table entry and appear in the same order as those entries. The values represent the section header indexes against which the symbol table entries are defined. Only if the corresponding symbol table entry's st_shndx field contains the escape value SHN_XINDEX will the matching Elf32_Word hold the actual section header index; otherwise, the entry must be SHN_UNDEF (0).

### 符号值
不同的符号的`st_value`有不同含义：
* 可重定位文件中，符号所在段为SHN_ABS的符号，st_value中保存的是符号对齐要求
* 可重定位文件中，st_value保存定义符号的段偏移(section offset)，符号所在的段由st_shndx确定
* 可执行文件、动态库中，st_value保存符号的虚拟地址

### 符号总结
上述小节介绍了ELF文件的符号表，对`simpleElf.o`的符号总结如下：
* 文件名(FILE)类型符号，局部绑定属性，SHN_ABS表示符号包含的是绝对值，即该符号不会被重定位，其符号值保存其对齐要求，此处为0，无对齐要求；符号不存在大小，故符号大小为0。
* 段类型(SECTION)符号用来引用段本身，局部绑定属性，根据st_shndx排查段头表，仅`.rela.text`、`.rela.eh_frame`、`.symtab`、`.strtab`、`.shstrtab`段无响应的符号，这些段有个特点：辅助重定位，段内的信息并不会被重定位。符号不存在大小，故符号大小为0。
* 数据类型(OBJECT)符号表示目标文件中的静态变量和全局变量，符号大小都是int类型大小(4)，静态变量是局部绑定属性，全局变量是全局绑定属性。初始化的静态变量和全局变量保存在.data段，未初始化的全局变量保存在.bss段，未初始化的全局变量保存在common block。其符号值表示符号的段内偏移。这些符号名与其定义在函数中的变量名并不相同，特别是静态符号，不仅有`static_`前缀，更有`.<num>`后缀，用于解决符号名冲突问题，即程序中的符号名必须唯一，这种行为成为名字改编(name mangling)[6]。静态变量的改编规则更为复杂也是因为静态变量的局部绑定属性允许不同目标模块存在同名变量，因此通过更为复杂的改编规则保证符号名的唯一。
* 函数类型(FUNC)符号对应目标文件中定义的两个函数，它们都在.text段，相应的符号值表示段内偏移，即`func`在段首，`main`在段内偏移`0x22`字节处，它们都是全局绑定属性，两个函数的大小分别是34(0x22)byte和53(0x35)byte，下面的objdump对.text段的反汇编可以验证。
* 未知类型符号(NOTYPE)`printf`，SHN_UNDEF表示其不是在当前目标文件中定义，其绑定属性为全局，引用符号的符号值和符号大小均未定义

```bash
objdump -d -j .text simpleElf.o

simpleElf.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <func>:
   0:	55                   	push   %rbp
   1:	48 89 e5             	mov    %rsp,%rbp
   4:	48 83 ec 10          	sub    $0x10,%rsp
   8:	89 7d fc             	mov    %edi,-0x4(%rbp)
   b:	8b 45 fc             	mov    -0x4(%rbp),%eax
   e:	89 c6                	mov    %eax,%esi
  10:	bf 00 00 00 00       	mov    $0x0,%edi
  15:	b8 00 00 00 00       	mov    $0x0,%eax
  1a:	e8 00 00 00 00       	callq  1f <func+0x1f>
  1f:	90                   	nop
  20:	c9                   	leaveq 
  21:	c3                   	retq   

0000000000000022 <main>:
  22:	55                   	push   %rbp
  23:	48 89 e5             	mov    %rsp,%rbp
  26:	48 83 ec 10          	sub    $0x10,%rsp
  2a:	c7 45 fc 01 00 00 00 	movl   $0x1,-0x4(%rbp)
  31:	8b 15 00 00 00 00    	mov    0x0(%rip),%edx        # 37 <main+0x15>
  37:	8b 05 00 00 00 00    	mov    0x0(%rip),%eax        # 3d <main+0x1b>
  3d:	01 c2                	add    %eax,%edx
  3f:	8b 45 fc             	mov    -0x4(%rbp),%eax
  42:	01 c2                	add    %eax,%edx
  44:	8b 45 f8             	mov    -0x8(%rbp),%eax
  47:	01 d0                	add    %edx,%eax
  49:	89 c7                	mov    %eax,%edi
  4b:	e8 00 00 00 00       	callq  50 <main+0x2e>
  50:	b8 00 00 00 00       	mov    $0x0,%eax
  55:	c9                   	leaveq 
  56:	c3                   	retq  
```

## 强符号与弱符号
我们经常碰到符号重复定义，多个目标文件中包含同名全局符号的定义，这些目标文件链接时会报符号重复定义的错误：
```bash
# cat lib.c
void foo(int *p)
{
	*p = 1;
}
# cat main.c
void foo(double *p)
{
	*p = 2.0;
}

int main()
{
	return 0;
}
# gcc -c lib.c 
# gcc -c main.c 
# gcc main.o lib.o -o main
lib.o: In function `foo':
lib.c:(.text+0x0): multiple definition of `foo'
main.o:main.c:(.text+0x0): first defined here
collect2: error: ld returned 1 exit status
```
可见，gcc在处理C语言的函数符号时，链接器是不支持识别符号类型的，即两个`foo`函数的返回值不同。然而在C++中，这两个函数的函数签名(Function Signature)是不同的，函数签名包含了一个函数的信息，包含函数名、它的参数类型，其所在的类和命名空间等其他信息。C++的名字改编机制保证不同的函数签名对应不同的符号，因此这两个`foo`函数在C++中是两个不同的符号：
```bash
# g++ -c main.c
# g++ -c lib.c
# g++ main.o lib.o -o main
# readelf -s lib.o | grep foo
     8: 0000000000000000    21 FUNC    GLOBAL DEFAULT    1 _Z3fooPi
# readelf -s main.o | grep foo
     9: 0000000000000000    27 FUNC    GLOBAL DEFAULT    1 _Z3fooPd
```
可见，使用C++编译器编译、链接上述文件，由于`foo`函数的符号不同，可以链接成功。

言归正传，上面的链接报错由于定义了两个同名的符号，而且这两个符号都是强符号(strong symbol)，与之相应的也有弱符号(weak symbol)。弱符号是在链接ELF目标文件过程中使用的特殊注释(specially annotated)符号，默认情况下都是强符号。链接期间，强符号可以覆盖同名弱符号，比如库中定义的弱符号可以被用户自定义的强符号所覆盖，从而使用自定义版本的库函数[7]。弱符号并不在C/C++规范中，本文主要讨论gcc扩展的弱符号。
链接过程中会按照下面的规则处理同名强/弱符号：
* 不允许强符号被多次定义，否则链接器报告重复定义错误
* 不同目标文件中定义同名的强符号、弱符号，链接器选择强符号

CSAPP、《程序员的自我修养》将未初始化的全局变量称为弱符号，我认为这种说法是不严谨的。前面的`global_uninit_var`变量是全局绑定属性的符号，该符号属于common block。更为严格的区分，`global_uninit_var`被称为"common symbol"而非weak symbol[8]。我也更加倾向将绑定属性为global的符号称为强符号，将绑定属性为weak的符号称为弱符号，若将common symbol也称为弱符号，符号的绑定属性的意义就会比较模糊。

gcc中定义弱符号有两种方式：`#pragma`预处理和gcc扩展weak属性`__attribute__((weak))`。简要介绍线下`#pragma`预处理：
```c
// function declaration
#pragma weak power2

int power2(int x);
```
我们主要关注gcc扩展weak属性定义弱符号的方式：
1. 以内核的`arch_report_meminfo`函数为例，在架构无关的文件`fs/proc/meminfo.c`中定义了arch_report_meminfo的弱符号，是空函数。在具体架构如powerpc、x86中实际实现了arch_report_meminfo函数。这些架构相关的arch_report_meminfo是强符号，从而完成了特定架构函数的实现，也能保证无关架构链接arch_report_meminfo时不出错（空函数）。
```c
fs/proc/meminfo.c:
void __attribute__((weak)) arch_report_meminfo(struct seq_file *m) 
{
}

arch/x86/mm/pat/set_memory.c:
void arch_report_meminfo(struct seq_file *m)
{
        seq_printf(m, "DirectMap4k:    %8lu kB\n",
                        direct_pages_count[PG_LEVEL_4K] << 2);
#if defined(CONFIG_X86_64) || defined(CONFIG_X86_PAE)
        seq_printf(m, "DirectMap2M:    %8lu kB\n",
                        direct_pages_count[PG_LEVEL_2M] << 11);
#else
        seq_printf(m, "DirectMap4M:    %8lu kB\n",
                        direct_pages_count[PG_LEVEL_2M] << 12);
#endif
        if (direct_gbpages)
                seq_printf(m, "DirectMap1G:    %8lu kB\n",
                        direct_pages_count[PG_LEVEL_1G] << 20);
}
```

2. 以glibc中的`strdup`库函数的实现为例，也定义了`strdup`的弱符号，允许用户实现同名强符号，覆盖库函数的弱符号：
```c
string/strdup.c:
char *
__strdup (const char *s) 
{
  size_t len = strlen (s) + 1;
  void *new = malloc (len);

  if (new == NULL)
    return NULL;

  return (char *) memcpy (new, s, len);
}
libc_hidden_def (__strdup)
weak_alias (__strdup, strdup)

include/libc-symbols.h:
/* Define ALIASNAME as a weak alias for NAME.
   If weak aliases are not available, this defines a strong alias.  */
# define weak_alias(name, aliasname) _weak_alias (name, aliasname)
# define _weak_alias(name, aliasname) \
  extern __typeof (name) aliasname __attribute__ ((weak, alias (#name)));
```

这个例子主要介绍weak attribute结合weak attribute的情况。先看alias attribute格式:
```c
# cat alias.c
int a[8] = {0};
extern __attribute__((alias ("a"))) int b[8];

void bar(int *p)
{
    *p = 1;
}
extern __attribute__((alias ("bar"))) void foo(int *p);

# gcc -c alias.c
# readelf -s alias.o

Symbol table '.symtab' contains 12 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS alias.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    2 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     8: 0000000000000000    32 OBJECT  GLOBAL DEFAULT    3 a
     9: 0000000000000000    32 OBJECT  GLOBAL DEFAULT    3 b
    10: 0000000000000000    21 FUNC    GLOBAL DEFAULT    1 bar
    11: 0000000000000000    21 FUNC    GLOBAL DEFAULT    1 foo
```
> the attribute alias tells the compiler that the declared symbol provides an alias, or alternate identity, for the symbol being named. The named symbol is known as the alias target. The target must be defined in the same translation unit as the alias; the alias itself can only be declared, it cannot be defined. The alias declaration is a definition even when also declared extern. [9]

简要总结下alias属性：
* alias属性用来给已经定义的符号提供一个别名符号(alias target)，将已经定义的符号称为target
* target和alias target必须在同一编译单元，且target必须在该编译单元定义。注意上面例子中的数组a显示初始化，如果显示不初始化，编译报错：`Error: b' can't be equated to common symbol a'`，我猜想未初始化的全局变量a属于common block，编译器认为这种情况属于“未定义”
* alias target只能以声明的方式出现，而不能被定义，如上面例子中的`extern`的声明方式。由于alias target是target的别名，两者实际是一样的，因此alias target已经定义
* 由readelf输出结果可知，target和alias target都是符号，除了符号名不同，其他符号属性都相同

`weak_alias (__strdup, strdup)`预处理之后是`extern __typeof(__strdup) strdup __attribute__ ((weak, alias (__strdup)));`，这条语句定义了别名符号`strdup`，其target是`strdup`符号，而且别名符号`strdup`是弱符号。

当前，我们看到的链接过程在查找符号引用，未找到符号定义时会报错，这种被称为强引用(strong reference)。相对应也存在弱引用(weak reference)，链接处理弱引用和强引用的过程几乎一样，只是对于未定义的符号，链接器处理弱引用不会报错，将未定义的符号值默认为0。
gcc的扩展弱引用属性有两种表示方式：
* `static __attribute__ ((weakref ("target"))) alias_target`，声明的弱引用符号为target，在当前程序中使用alias_target符号引用target。别名的weakref必须采用static修饰符，否则会编译报错：
```
# cat lib.c 
#include <stdio.h>

void bar()
{
	printf("%s\n",__func__);
}
# cat main.c
static __attribute__((weakref("bar"))) void foo();

int main()
{
	if (foo) foo();
	return 0;
}
# gcc main.c -o main && ./main 
# echo $?
0
# gcc -c main.c lib.c
# gcc -o main main.o lib.o && ./main 
bar
# readelf -s main.o

Symbol table '.symtab' contains 10 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS main.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
     8: 0000000000000000    31 FUNC    GLOBAL DEFAULT    1 main
     9: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND bar
```
可见，main.c定义了弱引用符号`bar`，无论`bar`是否定义，程序链接都不会出错。
* `__attribute__((weakref)) symbol`，无target的情况，即非别名的情况下，weakref也可用weak表示[10]：
```c
# cat pthread.c
#include <stdio.h>
#include <pthread.h>

int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                          void *(*start_routine) (void *), void *arg)
			__attribute__((weak));

int main()
{
	if (pthread_create) {
		printf("This is a multi-thread version\n");
	} else {
		printf("This is a single-thread version\n");
	}
	return 0;
}
# gcc pthread.c -o pt
# ./pt
This is a single-thread version
# gcc pthread.c -lpthread -o pt
# ./pt
This is a multi-thread version
```
有无链接libpthread.so，链接过程都不会报错。`pthread_create`在当前程序中是弱符号，弱链接了libpthread.so，则有符号`pthread_create`实际定义。

弱符号和弱引用在工程实现中具有如下作用：
* 库中定义的弱符号，可以被具体实现(用户定义/架构定义)的强符号覆盖，无强符号的情况下，弱符号的定义具有通用性，有强符号的情况下又具有特定性
* 程序可以将扩展功能模块的引用定义为弱引用，当扩展模块和程序链接在一起时，扩展功能可以正常使用，反之则缺少扩展功能

## common block
符号的`st_shndx`字段为`SHN_COMMON`是common symbol，有时候，可以通过符号的`st_type`字段为`STT_COMMON`判断符号为common symbol。并不要求common symbol同时满足`SHN_COMMON`、`STT_COMMON`。现在的编译器和链接器都支持common block机制，这种机制最早来源于Fortran。早期的Fortran没有动态分配空间的机制，程序员必须实现声明它所需空间大小。Fortran把这种空间叫做common block，当不同的目标文件需要的common block空间不一致时，以最大的空间为准。
未定义的全局变量是commcon symbol,前面章节提及的`global_uninit_var`就是一个典型例子。链接器链接同名的强符号、common符号、弱符号遵循如下规则：
> IWhen the link editor combines several relocatable object files, it does not allow multiple definitions of STB_GLOBAL symbols with the same name. On the other hand, if a defined global symbol exists, the appearance of a weak symbol with the same name will not cause an error. The link editor honors the global definition and ignores the weak ones. Similarly, if a common symbol exists (that is, a symbol whose st_shndx field holds SHN_COMMON), the appearance of a weak symbol with the same name will not cause an error. The link editor honors the common definition and ignores the weak ones.[11]

简而言之，符号优先级：`STB_GLOBAL > COMMON > STB_WEAK`，更多关于common symbol的介绍也请参考[11]。

gcc的`-fno-common`编译选项允许将所有未初始化的全局变量不以common block的形式处理，或者使用__attritute__扩展:
```c
int global __attribute__((nocommon));
```
此时这些未定义的全局变量便是强符号。GCC和Clang之前的版本默认使用`-fcommon`编译C文件，自GCC 10/Clang 11默认采用`-fno-common`[11]。


## Reference
[1] http://www.sco.com/developers/gabi/latest/ch4.eheader.html
[2] http://www.sco.com/developers/gabi/latest/ch4.sheader.html
[3] https://docs.oracle.com/cd/E26926_01/html/E25910/ggdlu.html
[4] http://www.sco.com/developers/gabi/latest/ch4.symtab.html
[5] https://blogs.oracle.com/solaris/post/inside-elf-symbol-tables
[6] https://zh.m.wikipedia.org/zh-hans/%E5%90%8D%E5%AD%97%E4%BF%AE%E9%A5%B0
[7] https://en.wikipedia.org/wiki/Weak_symbol
[8] https://stackoverflow.com/questions/3691835/why-uninitialized-global-variable-is-weak-symbol
[9] https://developers.redhat.com/blog/2020/06/03/the-joys-and-perils-of-aliasing-in-c-and-c-part-2#aliases_and_weak_symbols
[10] https://gcc.gnu.org/onlinedocs/gcc-4.3.5/gcc/Function-Attributes.html
[11] https://maskray.me/blog/2022-02-06-common-symbols
