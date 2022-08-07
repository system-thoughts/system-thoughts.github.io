---
title: 开机过程分析--BIOS阶段
date: 2021-12-09T21:27:09+08:00
tags: [boot, BIOS]
categories: boot
---

无论是Linux还是其他操作系统，开机启动最开始的流程由BIOS完成。当电脑上电后，BIOS首先会初始化BIOS内部状态、外部接口、检测并setup硬件。这个最开始的阶段称为开机自检(POST, Power On Self Test)。随后BIOS进入启动阶段，会检查启动介质，找到bootloader，将其加载至内存并跳转至bootloader执行。
本文以[SeaBIOS](https://github.com/coreboot/seabios)为例，介绍x86架构下BIOS在启动流程所做的工作。SeaBIOS是16bit x86 BIOS的开源实现，也是qemu和kvm默认的BIOS。

## 实模式
x86实模式出现于Intel 8088时期，8088 CPU公有20位地址总线，8个16进制通用寄存器以及4个段寄存器(CS DS SS ES)以及3个专用寄存器(IP SP FLAGS)。要通过16位寄存器寻址20位地址空间，8088引入了分段的方法：
> 物理地址 = 段寄存器 << 4 + 段内偏移

如此，通过16bit的段寄存器和段内偏移便能寻址20位地址空间，计算出的地址便是实际物理地址，这也是“实”模式的由来。

## First Instruction
系统上电后，BIOS最初便在16bit实模式下工作。x86-64环境下，BIOS的第一条指令位于CS(0xF000h):IP(0xFFF0h)所指向的地址。刚才提及，上电之初，BIOS在16bit实模式下工作，最大寻址1MB，按照实模式下的寻址规则：`(0xF000h << 4) + (0xFFF0h) = 0xFFFF0h`。那么开机启动之后，第一条指令会在`0xFFFF0h`执行？其实不然，开机执行的第一条指令在`0xfffffff0h`处执行。分段模式下的每个段寄存器寻址的实际物理地址缓存在`段描述符高速缓冲寄存器`中，这个寄存器对于程序员是不可见的，段描述符高速缓冲寄存器就是通过硬件加速分段模式下的寻址速度。`CS`段寄存器对应的段描述符高速缓冲寄存器存储的是物理地址`0xFFFFFFF0h`，该物理地址称为[Reset vector](https://en.wikipedia.org/wiki/Reset_vector)。

执行`qemu-kvm -monitor stdio -S`命令查看机器启动执行的第一条指令：
```shell
[root@192 lab1]# qemu-kvm -monitor stdio -S
QEMU 2.12.0 monitor - type 'help' for more information
(qemu) VNC server running on ::1:5901

(qemu) info registers 
EAX=00000000 EBX=00000000 ECX=00000000 EDX=000006d3
ESI=00000000 EDI=00000000 EBP=00000000 ESP=00000000
EIP=0000fff0 EFL=00000002 [-------] CPL=0 II=0 A20=1 SMM=0 HLT=0
ES =0000 00000000 0000ffff 00009300
CS =f000 ffff0000 0000ffff 00009b00
SS =0000 00000000 0000ffff 00009300
DS =0000 00000000 0000ffff 00009300
FS =0000 00000000 0000ffff 00009300
GS =0000 00000000 0000ffff 00009300
LDT=0000 00000000 0000ffff 00008200
TR =0000 00000000 0000ffff 00008b00
GDT=     00000000 0000ffff
IDT=     00000000 0000ffff
CR0=60000010 CR2=00000000 CR3=00000000 CR4=00000000
...
```
`CS`寄存器为`0xf000`，`EIP`寄存器为`0000fff0`，`CS`寄存器对应的段基址是`0xffff0000`，该段最能容纳`0x0000ffff`字节。如此，起始物理地址便是`0xfffffff0`，便是`Reset vector`。

但是`0xFFFFFFF0h`仍然高于实模式下的最大物理地址空间1MB，为啥BIOS可以访问到了？`0xFFFFFFF0h`实际是[BIOS ROM映射到地址空间的地址](https://www.coreboot.org/Developer_Manual/Memory_map)，而非在实模式寻址RAM的物理地址：
> 0xFFFE_0000 - 0xFFFF_FFFF: 128 kByte ROM mapped into address space

查看SeaBIOS源码，BIOS第一条指令会执行什么：
```asm
        ORG 0xfff0 // Power-up Entry Point
        .global reset_vector
reset_vector:
        ljmpw $SEG_BIOS, $entry_post
```
`ORG`是一个[宏](https://wiki.osdev.org/Opcode_syntax)：
```asm
        // Specify a location in the fixed part of bios area.
        .macro ORG addr
        .section .fixedaddr.\addr
        .endm
```
`ORG 0xfff0`会定义一个名为`.fixedaddr.0xfff0`的ELF section，构建阶段，`layoutrom.py`脚本会检测名称中包含".fixedaddr."的section，并在最终的链接脚本中指定该section加载到的物理地址。
可见，第一条指令会执行一个长跳转（普通跳转只会更新EIP寄存器，长跳转还会更新CS寄存器）至`0xf000`:`0xe05b`
```asm
src/config.h：
// Important real-mode segments
#define SEG_IVT      0x0000
#define SEG_BDA      0x0040
#define SEG_BIOS     0xf000

src/romlayout.S:
        ORG 0xe05b
entry_post:
        cmpl $0, %cs:HaveRunPost                // Check for resume/reboot
        jnz entry_resume
        ENTRY_INTO32 _cfunc32flat_handle_post   // Normal entry point
```

第一条指令便是跳转到1MB以下的`0xfe05b`处执行，如[下图](https://read.seas.harvard.edu/~kohler/class/08w-aos/lab1.html)所示，该地址处于BIOS ROM区域。BIOS ROM内存区域是BIOS shadow内存区域，当前该内存区域是BIOS在ROM/flash实际存储中的一份只读拷贝。由于CPU访问flash/ROM的速度/带宽要慢于CPU访问RAM的速度，BIOS shadow可以加快CPU访问BIOS程序的速度。

![](physical_memory_map.PNG)

首先通过`HaveRunPost`全局变量判断是否经历过POST阶段，若经过POST阶段，则跳过POST阶段执行`resume`。否则切换到32bit保护模式执行`handle_post`函数进入POST阶段。

```c
// Indicator if POST phase has been started (and if it has completed).
int HaveRunPost VARFSEG;
```

## POST Phase

### 切换到32bit保护模式
我们首先看下进入32bit保护模式是如何实现的：
```c
src/entryfuncs.S:
        // Reset stack, transition to 32bit mode, and call a C function.
        .macro ENTRY_INTO32 cfunc
        xorw %dx, %dx 
        movw %dx, %ss 
        movl $ BUILD_STACK_ADDR , %esp
        movl $ \cfunc , %edx
        jmp transition32
        .endm

src/config.h：
// Various memory addresses used by the code.
#define BUILD_STACK_ADDR          0x7000
```
首先设置堆栈段寄存器`ss`为0，BIOS ROM的程序堆栈处于low memory地址空间(0x7000)。这里的`cfunc`是函数Label，被传递到`edx`寄存器，`transition32`切换到32bit保护模式之后，会跳转到该地址执行。

`transition32`将CPU转换成32bit：
```asm
src/entryfuncs.S:
        // Declare a function
        .macro DECLFUNC func
        .section .text.asm.\func
        .global \func
        .endm


src/romlayout.S:
// Place CPU into 32bit mode from 16bit mode.
// %edx = return location (in 32bit mode)
// Clobbers: ecx, flags, segment registers, cr0, idt/gdt
        DECLFUNC transition32
        .global transition32_nmi_off
transition32: 
        // Disable irqs (and clear direction flag)
        cli
        cld
        
        // Disable nmi
        movl %eax, %ecx
        movl $CMOS_RESET_CODE|NMI_DISABLE_BIT, %eax
        outb %al, $PORT_CMOS_INDEX
        inb $PORT_CMOS_DATA, %al
        
        // enable a20
        inb $PORT_A20, %al
        orb $A20_ENABLE_BIT, %al
        outb %al, $PORT_A20
        movl %ecx, %eax
        
transition32_nmi_off:
        // Set segment descriptors
        lidtw %cs:pmode_IDT_info
        lgdtw %cs:rombios32_gdt_48

        // Enable protected mode
        movl %cr0, %ecx
        andl $~(CR0_PG|CR0_CD|CR0_NW), %ecx
        orl $CR0_PE, %ecx
        movl %ecx, %cr0

        // start 32bit protected mode code
        ljmpl $SEG32_MODE32_CS, $(BUILD_BIOS_ADDR + 1f)

        .code32
        // init data segments
1:      movl $SEG32_MODE32_DS, %ecx
        movw %cx, %ds
        movw %cx, %es
        movw %cx, %ss
        movw %cx, %fs
        movw %cx, %gs

        jmpl *%edx        
```
`DECLFUNC`宏定义名为`.text.asm.transition32`的section，`transition32`以及`transition32_nmi_off`都在该section，这两个符号通过`.global`对链接器可见。

首先关中断并清空方向标志位。过去，为了节省存储空间，很多功能都被合并集成在一个有“空间”的芯片上，控制NMI中断使能、CMOS控制器、RTC时钟都放在CMOS中。
```c
src/hw/rtc.h:
#define PORT_CMOS_INDEX        0x0070
#define PORT_CMOS_DATA         0x0071

// PORT_CMOS_INDEX nmi disable bit
#define NMI_DISABLE_BIT 0x80
...
#define CMOS_RESET_CODE          0x0f
```
关闭NMI中断的[步骤](https://wiki.osdev.org/CMOS#Writing_to_the_CMOS)如下：
1. 向`0x70`端口，写入NMI diable bit(0x80)以及选择CMOS寄存器，接着要通过`0x71`端口从同一选择CMOS寄存器读取才算完成关闭NMI操作：
```asm
outb (0x70, (NMI_disable_bit << 7) | (selected CMOS register number));
```
2. 从`0x71`端口读取选择CMOS寄存器完成关闭NMI中断：
```asm
inb (0x71, (NMI_disable_bit << 7) | (selected CMOS register number));
```
随后开启A20总线以便访问1MB以上的物理地址空间。8086/8088实模式下，最大能够访问的物理地址是`0xFFFF:0xFFFF`即`0x10FFEF`。访问该物理地址需要21根地址线，然而8086/8088仅有20根地址总线，因此改地址会被截断成`0x0FFEF`。后续80x86 CPU为了兼容这种回环现象，设计了A20总线，当A20总线为1，第21位以及更高位都有效；反之，高位为0，兼容回环现象。
```c
// PORT_A20 bitdefs
#define PORT_A20 0x0092
#define A20_ENABLE_BIT 0x02
```
可见通过往A20 `0x92`I/O端口置位第2 bit位开启A20总线。
随后通过`lidtw`和`lgdtw`两条命令加载`idtr`和`gdtr`寄存器，设定中断描述符表和全局描述符表。32bit保护模式下的分段寻址和实模式有较大差异。偏移值同实模式一样，只不过变成了32bit。段值仍然放在以前的16bit寄存器，不过寄存器存放的不是段基址，而是段选择符。通过段选择符不仅能够索引全局/局部描述符表获取段描述符中的基址，还能够描述当前访问操作的特权级实现段保护。
![](segment_selector.PNG)

段描述符表`rombios32_gdt_48`如代码所示，对比段描述符结构，则一目了然。
```c
// Dummy IDT that forces a machine shutdown if an irq happens in
// protected mode.
u8 dummy_IDT VARFSEG;

// Protected mode IDT descriptor
struct descloc_s pmode_IDT_info VARFSEG = {
    .length = sizeof(dummy_IDT) - 1,
    .addr = (u32)&dummy_IDT,
};

// GDT
u64 rombios32_gdt[] VARFSEG __aligned(8) = {
    // First entry can't be used.
    0x0000000000000000LL,
    // 32 bit flat code segment (SEG32_MODE32_CS)
    GDT_GRANLIMIT(0xffffffff) | GDT_CODE | GDT_B,
    // 32 bit flat data segment (SEG32_MODE32_DS)
    GDT_GRANLIMIT(0xffffffff) | GDT_DATA | GDT_B,
    // 16 bit code segment base=0xf0000 limit=0xffff (SEG32_MODE16_CS)
    GDT_LIMIT(BUILD_BIOS_SIZE-1) | GDT_CODE | GDT_BASE(BUILD_BIOS_ADDR),
    // 16 bit data segment base=0x0 limit=0xffff (SEG32_MODE16_DS)
    GDT_LIMIT(0x0ffff) | GDT_DATA,
    // 16 bit code segment base=0xf0000 limit=0xffffffff (SEG32_MODE16BIG_CS)
    GDT_GRANLIMIT(0xffffffff) | GDT_CODE | GDT_BASE(BUILD_BIOS_ADDR),
    // 16 bit data segment base=0 limit=0xffffffff (SEG32_MODE16BIG_DS)
    GDT_GRANLIMIT(0xffffffff) | GDT_DATA,
};

// GDT descriptor
struct descloc_s rombios32_gdt_48 VARFSEG = { 
    .length = sizeof(rombios32_gdt) - 1,
    .addr = (u32)rombios32_gdt,
};
```
![](segment_descriptor.PNG)

随后通过设置`CR0`寄存器：关闭分页模式、使能CPU cache、设置CPU cache Write-through、并开启保护模式：
```c
src/x86.h:
// CR0 flags
#define CR0_PG (1<<31) // Paging
#define CR0_CD (1<<30) // Cache disable
#define CR0_NW (1<<29) // Not Write-through
#define CR0_PE (1<<0)  // Protection enable
```

最终，通过长跳转`ljmpl`跳转至32bit保护模式，`CS`寄存器存储的段选择符为`01000B`,即`RPL = 0`,执行在特权级0上；`TI = 0`，段选择符索引GDT；`Index = 1`，索引GDT中的第一个段描述符。段描述符中基址是`0x0`，`Segment Limit`是`0xfffff`，`G(Granularity)`flag置位，表示该段的范围是`0x100000 * 4KB = 4GB`。
```c
src/config.h:
// Segment definitions in protected mode (see rombios32_gdt in misc.c)
#define SEG32_MODE32_CS    (1 << 3)
#define SEG32_MODE32_DS    (2 << 3)
```

这里的1f要区分清楚。指的是前方第一个标签为“1”的位置，而不是代表十六进制数0x1F。下一个标签“1”就是这个指令的下一条。所以，看起来这个跳转是没有价值的。实际上，在cr0寄存器被设定好之前，下一条指令已经被放入流水线。而再放入的时候这条指令还是在实模式下的。所以这个ljmp指令是为了清空流水线，确保下一条指令在保护模式下执行。[1]

随后初始化所有的数据段寄存器`ss`、`ds`、`es`、`fs`、`gs`为`$SEG32_MODE32_DS`。段描述符中基址是`0x0`，`Segment Limit`是`0xfffff`，`G(Granularity)`flag置位，段的大小都是4GB。进入实模式后，所有段寻址的地址空间都是`[0, 4G)`,当前地址空间不再是分段的，而是完整的一大块，即“平坦模型”。

随后跳转到32bit保护模式下的`handle_post`函数。

### handle_post

```c
src/post.c:
// Entry point for Power On Self Test (POST) - the BIOS initilization
// phase.  This function makes the memory at 0xc0000-0xfffff
// read/writable and then calls dopost().
void VISIBLE32FLAT
handle_post(void)
{
    if (!CONFIG_QEMU && !CONFIG_COREBOOT)
        return;

    serial_debug_preinit();
    debug_banner();

    // Check if we are running under Xen.
    xen_preinit();

    // Allow writes to modify bios area (0xf0000)
    make_bios_writable();

    // Now that memory is read/writable - start post process.
    dopost();
}
```
post阶段最开始的启动串口调试等细节我们就忽略，着重关注`make_bios_writable`和`dopost`。

前面提到BIOS ROM内存区域是BIOS shadow内存区域，该内存区域是BIOS ROM/flash在内存中的只读拷贝。`make_bios_writable`就是用于让这段内存可写，以便后续更改一些静态分配的全局变量。

```c
// Setup for code relocation and then relocate.
void VISIBLE32INIT
dopost(void)
{
    code_mutable_preinit();

    // Detect ram and setup internal malloc.
    qemu_preinit();
    coreboot_preinit();
    malloc_preinit();
    
    // Relocate initialization code and call maininit().
    reloc_preinit(maininit, NULL);
}
```
`code_mutable_preinit`通过设置全局变量`HaveRunPost`为1，防止重复进入`POST`阶段。

根据SeaBIOS官方文档[2]描述，POST阶段可以分为如下四个子阶段：
* preinit: 代码重定位之前的初始化操作
* init: 初始化内部变量和接口
* setup: setup硬件和驱动
* preboot: 所有接口初始化工作完成，准备启动

SeaBIOS的代码重定位是为了释放部分占用的BIOS ROM空间，将原本很小的内存空间留给Option ROM、BIOS table以及运行时的存储空间。直接看`maininit`的最后代码部分`startBoot`准备启动：
```c
// Begin the boot process by invoking an int0x19 in 16bit mode.
void VISIBLE32FLAT
startBoot(void)
{
    // Clear low-memory allocations (required by PMM spec).
    memset((void*)BUILD_STACK_ADDR, 0, BUILD_EBDA_MINIMUM - BUILD_STACK_ADDR);

    dprintf(3, "Jump to int19\n");
    struct bregs br;
    memset(&br, 0, sizeof(br));
    br.flags = F_IF;
    call16_int(0x19, &br);
}
```
bootloader工作在16bit的实模式，`call16_int`切换至实模式并调用`int 0x19h`软件中断(software-generated interrupt)，进入BIOS的BOOT阶段。
> The INT n instruction permits interrupts to be generated from within software by supplying an interrupt vector number as an operand.

软件（生成）中断源自CPU主动执行特定的指令（x86下的int指令）产生中断，而非源自CPU接收的外部硬件产生的中断。

```c
src/stacks.h:
#define call16_int(nr, callregs) do {                           \
        extern void irq_trampoline_ ##nr (void);                \
        __call16_int((callregs), (u32)&irq_trampoline_ ##nr );  \
    } while (0)

src/romlayout.S:
// IRQ trampolines
        .macro IRQ_TRAMPOLINE num 
        DECLFUNC irq_trampoline_0x\num
        irq_trampoline_0x\num :
        int $0x\num
        lretw
        .endm

        IRQ_TRAMPOLINE 02
        IRQ_TRAMPOLINE 05
        IRQ_TRAMPOLINE 10
        IRQ_TRAMPOLINE 13
        IRQ_TRAMPOLINE 15
        IRQ_TRAMPOLINE 16
        IRQ_TRAMPOLINE 18
        IRQ_TRAMPOLINE 19
        IRQ_TRAMPOLINE 1b
        IRQ_TRAMPOLINE 1c
        IRQ_TRAMPOLINE 4a
```
可见`irq_trampoline_0x19`实际执行:
```asm
int $0x19
lretw
```
`maininit`->`interface_init`->`ivt_init`初始化中断向量表：
```c
src/post.c:
    // Initialize software handlers.
    SET_IVT(0x02, FUNC16(entry_02));
    SET_IVT(0x05, FUNC16(entry_05));
    SET_IVT(0x10, FUNC16(entry_10));
    SET_IVT(0x11, FUNC16(entry_11));
    SET_IVT(0x12, FUNC16(entry_12));
    SET_IVT(0x13, FUNC16(entry_13_official));
    SET_IVT(0x14, FUNC16(entry_14));
    SET_IVT(0x15, FUNC16(entry_15_official));
    SET_IVT(0x16, FUNC16(entry_16));
    SET_IVT(0x17, FUNC16(entry_17));
    SET_IVT(0x18, FUNC16(entry_18));
    SET_IVT(0x19, FUNC16(entry_19_official));
    SET_IVT(0x1a, FUNC16(entry_1a_official));
    SET_IVT(0x40, FUNC16(entry_40));

src/romlayout.S:
        // int 18/19 are special - they reset stack and call into 32bit mode.
        DECLFUNC entry_19
entry_19:
        ENTRY_INTO32 _cfunc32flat_handle_19
...
        .global entry_19_official
entry_19_official:
        jmp entry_19
```
`0x19`软件中断会执行`handle_19`，BIOS进入了BOOT阶段。

## BOOT Phase
```c
src/boot.c:
// Determine next boot method and attempt a boot using it.
static void 
do_boot(int seq_nr)
{
    if (! CONFIG_BOOT)
        panic("Boot support not compiled in.\n");

    if (seq_nr >= BEVCount)
        boot_fail();

    // Boot the given BEV type.
    struct bev_s *ie = &BEV[seq_nr];
    switch (ie->type) {
    case IPL_TYPE_FLOPPY:
        printf("Booting from Floppy...\n");
        boot_disk(0x00, CheckFloppySig);
        break;
    case IPL_TYPE_HARDDISK:
        printf("Booting from Hard Disk...\n");
        boot_disk(0x80, 1);
        break;
    case IPL_TYPE_CDROM:
        boot_cdrom((void*)ie->vector);
        break;
    case IPL_TYPE_CBFS:
        boot_cbfs((void*)ie->vector);
        break;
    case IPL_TYPE_BEV:
        boot_rom(ie->vector);
        break;
    case IPL_TYPE_HALT:
        boot_fail();
        break;
    }    

    // Boot failed: invoke the boot recovery function
    struct bregs br;
    memset(&br, 0, sizeof(br));
    br.flags = F_IF;
    call16_int(0x18, &br);
}

// INT 19h Boot Load Service Entry Point
void VISIBLE32FLAT
handle_19(void)
{
    debug_enter(NULL, DEBUG_HDL_19);
    BootSequence = 0; 
    do_boot(0);
}
```
`do_boot`从第一个启动项进行启动，以常见的硬盘(`IPL_TYPE_HARDDISK`)为例:
```c
src/boot.c
// Boot from a disk (either floppy or harddrive)
static void 
boot_disk(u8 bootdrv, int checksig)
{
    u16 bootseg = 0x07c0;

    // Read sector
    struct bregs br;
    memset(&br, 0, sizeof(br));
    br.flags = F_IF;
    br.dl = bootdrv;
    br.es = bootseg;
    br.ah = 2; 
    br.al = 1; 
    br.cl = 1; 
    call16_int(0x13, &br);

    if (br.flags & F_CF) {
        printf("Boot failed: could not read the boot disk\n\n");
        return;
    }    

    if (checksig) {
        struct mbr_s *mbr = (void*)0;
        if (GET_FARVAR(bootseg, mbr->signature) != MBR_SIGNATURE) {
            printf("Boot failed: not a bootable disk\n\n");
            return;
        }    
    }    

    tpm_add_bcv(bootdrv, MAKE_FLATPTR(bootseg, 0), 512);

    /* Canonicalize bootseg:bootip */
    u16 bootip = (bootseg & 0x0fff) << 4;
    bootseg &= 0xf000;

    call_boot_entry(SEGOFF(bootseg, bootip), bootdrv);
}
```
`int 0x13H`从硬盘读取第一个扇区(MBR)至`ES:BX`,`BX`初始化为0，因此MBR的内容读取到`0x07c0:0x0`。 `int 0x13H`相关寄存器参数[3]:
![](int13.PNG)
`GET_FARVAR(bootseg, mbr->signature)`便是校验MBR签名`0xaa55`，确认读取到的内容就是MBR扇区。
最后的规范化(Canonicalize)操作为了让后续调用bootloader时，段地址:偏移地址为`0x0:0x7c00`而非`0x07c0:0x0`。
最后调用`call_boot_entry`跳转至`0x0:0x7c00`地址，转移至bootloader执行，此时系统仍处于16bit实模式。

我们写一个简单的bootloader，看看BIOS启动跳转到bootloader的效果：
```asm
boot.S:

.global _start
.text
.code16
_start:
	mov $0x21, %al
	mov $0x0e, %ah
	mov $0x00, %bh
	mov $0x07, %bl
	int $0x10

	jmp .

.space 510-(.-_start)
.word 0xaa55
```
构建[4]并运行：
```
# gcc -Wl,--oformat=binary -Wl,-Ttext=0x7c00 -Wl,--build-id=none -nostartfiles -nostdlib -m32 -o boot.bin boot.S
# qemu-kvm -nographic -drive format=raw,file=boot.bin
```
![](mbr.PNG)
bootloader程序很简单，就是向屏幕输出`!`。

## Reference
[1] [SeaBIOS实现简单分析](https://www.cnblogs.com/gnuemacs/p/14287120.html)
[2] [Execution_and_code_flow](https://github.com/coreboot/seabios/blob/master/docs/Execution_and_code_flow.md)
[3] [INT 13H Wikipedia](https://en.wikipedia.org/wiki/INT_13H)
[4] [Calculating padding length with GAS AT&T directives for a boot sector?](https://stackoverflow.com/questions/47859273/calculating-padding-length-with-gas-att-directives-for-a-boot-sector)