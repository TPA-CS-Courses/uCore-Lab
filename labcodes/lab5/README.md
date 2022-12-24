来梳理一下ucore中几种切换机制：
- 中断上下文切换
- 进程上下文切换
- 函数切换

## 中断上下文切换

我们在`kern_init`时会调用`idt_init`函数，对中断向量表进行初始化：
```c
void
idt_init(void) {
    extern uintptr_t __vectors[];
    int i;
    for (i = 0; i < sizeof(idt) / sizeof(struct gatedesc); i ++) {
        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
    }
	// set for switch from user to kernel
    SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
	// load the IDT
    lidt(&idt_pd);
}
```
这个`__vectors[]`数组（中断向量表）中就存有中断处理程序的地址，

我们在`vectors.S`中就能看到这个中断向量表中存的内容如下：
```c
# vector table
.data
.globl __vectors
__vectors:
  .long vector0
  .long vector1
  .long vector2
  .long vector3
  .long vector4
  .long vector5
  .long vector6
  .long vector7
// ....
```
接着，我们能看到这些`vector0, vector1`的地址下的指令是下面这样的。
```asm
.globl vector0
vector0:
  pushl $0
  pushl $0
  jmp __alltraps
.globl vector1
vector1:
  pushl $0
  pushl $1
  jmp __alltraps
.globl vector2
vector2:
  pushl $0
  pushl $2
  jmp __alltraps
.globl vector3
```
会往栈中压入0和中断编号，然后跳转到`__alltraps`处。
而，`__alltraps`处就是进行中断上下文切换的地方。
```asm
# vectors.S sends all traps here.
.text
.globl __alltraps
__alltraps:
    # push registers to build a trap frame
    # therefore make the stack look like a struct trapframe
    pushl %ds
    pushl %es
    pushl %fs
    pushl %gs
    pushal

    # load GD_KDATA into %ds and %es to set up data segments for kernel
    movl $GD_KDATA, %eax
    movw %ax, %ds
    movw %ax, %es

    # push %esp to pass a pointer to the trapframe as an argument to trap()
    pushl %esp

    # call trap(tf), where tf=%esp
    call trap

    # pop the pushed stack pointer
    popl %esp

    # return falls through to trapret...
.globl __trapret
__trapret:
    # restore registers from stack
    popal

    # restore %ds, %es, %fs and %gs
    popl %gs
    popl %fs
    popl %es
    popl %ds

    # get rid of the trap number and error code
    addl $0x8, %esp
    iret

.globl forkrets
forkrets:
    # set stack to this new process's trapframe
    movl 4(%esp), %esp
    jmp __trapret

```

能够看到，它实际上是往栈中压入一系列寄存器，组成trapframe结构（这里面存的就是发生中断之前的寄存器状况），将之前的状态备份之后，再将寄存器的内容切换为内核中的段，至此，段已经是咱内核的段啦
```asm
    # load GD_KDATA into %ds and %es to set up data segments for kernel
    movl $GD_KDATA, %eax
    movw %ax, %ds
    movw %ax, %es

```
然后，调用`call trap`，这样就执行到了我们内核的程序`trap()`中了，至此，中断上下文的切换就完成了。
总结一下，在中断之前，**段是用户态的段，程序是用户的程序**，发生中断后，我们将用户的段保存好，组成一个trapframe结构体，然后`call trap`，调用内核中的`trap`函数，这样，段完成了切换，程序也跳转到了内核中的`trap`函数。


流程：
- `int`指令，`int`指令就像个特殊的函数调用，后面会跟一个中断号，如`int 0x80`
- 根据中断，查看中断描述符表`idt[]`（在内核中`idt_init`时，使用`__vectors`初始化好的）
- 比较CS段和`idt[]`中断描述符的DPL，判断是否能切换`idt[]`中保存的`__vectors`地址，切换内核栈
- `__alltraps`完成中断上下文的切换，构造trapframe
- `call trap`
- `iret`指令

详细的流程：
硬件中断处理过程1（起始）：从CPU收到中断事件后，打断当前程序或任务的执行，根据某种机制跳转到中断服务例程去执行的过程。其具体流程如下：
- CPU在执行完当前程序的每一条指令后，都会去确认在执行刚才的指令过程中中断控制器（如：8259A）是否发送中断请求过来，如果有那么CPU就会在相应的时钟脉冲到来时从总线上读取中断请求对应的中断向量；
- CPU根据得到的中断向量（以此为索引）到IDT中找到该向量对应的中断描述符，中断描述符里保存着中断服务例程的段选择子；
- CPU使用IDT查到的中断服务例程的段选择子从GDT中取得相应的段描述符，段描述符里保存了中断服务例程的段基址和属性信息，此时CPU就得到了中断服务例程的起始地址，并跳转到该地址；
- CPU会根据CPL和中断服务例程的段描述符的DPL信息确认是否发生了特权级的转换。比如当前程序正运行在用户态，而中断程序是运行在内核态的，则意味着发生了特权级的转换，这时CPU会从当前程序的TSS信息（该信息在内存中的起始地址存在TR寄存器中）里取得该程序的内核栈地址，即包括内核态的ss和esp的值，并立即将系统当前使用的栈切换成新的内核栈。这个栈就是即将运行的中断服务程序要使用的栈。紧接着就将当前程序使用的用户态的ss和esp压到新的内核栈中保存起来；
- CPU需要开始保存当前被打断的程序的现场（即一些寄存器的值），以便于将来恢复被打断的程序继续执行。这需要利用内核栈来保存相关现场信息，即依次压入当前被打断程序使用的eflags，cs，eip，errorCode（如果是有错误码的异常）信息；
- CPU利用中断服务例程的段描述符将其第一条指令的地址加载到cs和eip寄存器中，开始执行中断服务例程。这意味着先前的程序被暂停执行，中断服务程序正式开始工作。



接着，我们看`trapframe`这个结构体：
```c
struct trapframe {
    struct pushregs tf_regs;
    uint16_t tf_gs;
    uint16_t tf_padding0;
    uint16_t tf_fs;
    uint16_t tf_padding1;
    uint16_t tf_es;
    uint16_t tf_padding2;
    uint16_t tf_ds;
    uint16_t tf_padding3;
    uint32_t tf_trapno;
    /* below here defined by x86 hardware */
    uint32_t tf_err;
    uintptr_t tf_eip;
    uint16_t tf_cs;
    uint16_t tf_padding4;
    uint32_t tf_eflags;
    /* below here only when crossing rings, such as from user to kernel */
    uintptr_t tf_esp;
    uint16_t tf_ss;
    uint16_t tf_padding5;
} __attribute__((packed));
```
这里面存了一系列的寄存器的值，可以看到，`ds, es, fs, gs`这四个寄存器的值是在`__alltraps`程序中完成入栈的，而此之后的`eip, cs, eflags, esp, ee`等，就如注释所说：below here defined by x86 hardware，也就是由指令集定义，例如使用`int`指令时，就会将`cs, ip`等值入栈。

综上，`trapframe`里就保存了中断发生之前寄存器的样子（包括`int`指令为我们保存，以及我们手动保存的）。

## 进程上下文切换

