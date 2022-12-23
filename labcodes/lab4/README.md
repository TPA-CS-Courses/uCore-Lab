## 练习0：填写已有实验


## 练习1：分配并初始化一个进程控制块（需要编码）

我们看`proc_init`函数，该函数的主要作用是：
- 将当前状态构造一个`idleproc`（这也是第一个内核线程）
- 构造一个`initproc`内核线程，跑`init_main`函数

感觉这部分和我之前看C++实现的协程服务器比较类似，无论是协程还是线程，对于第一个协程/线程，都是凭空将当前状态构造出一个协程/线程，这个协程/线程的作用往往是idle，也就是进行调度的。
```c++
void
cpu_idle(void) {
    while (1) {
        if (current->need_resched) {
            schedule();
        }
    }
}
```

对于协程而言：一个线程中的协程分成两类协程，一类只有一个，就是调度协程，一类有多个，是需要处理的任务协程。

对于内核线程而言：一个OS中有两类线程，一类只有一个，就是调度线程，一类有多个，是需要处理的任务线程。

综上，无论是线程调度、协程调度，本质上都是三个部分：
- 任务队列：存放可调度任务的队列
- 调度任务：用来进行调度的任务
- 执行模块：用来执行任务的模块，该模块可能执行任务队列中的任务，也可能执行调度任务。

**任务队列**中存放着能够被调度执行的任务，**执行模块**执行完当前任务后，会切换到**调度任务**，**调度任务**在**任务队列**中取出下一个任务，送给**执行模型**去执行。

以ucore为例：
- 任务队列：`proc_list`，是`process`链表
- 调度任务：`idleproc`，idle线程，进行调度
- 执行模块：就是跑线程的CPU，`current`，记录当前正在执行的线程，可能是`idleproc`，也可能是其他任务线程。

以协程服务器为例：
- 任务队列：协程队列
- 调度任务：调度协程
- 执行模块：就是跑协程的线程。


我们来看看`proc_struct`：
```c
struct proc_struct {
    enum proc_state state;                      // Process state
    int pid;                                    // Process ID
    int runs;                                   // the running times of Proces
    uintptr_t kstack;                           // Process kernel stack
    volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?
    struct proc_struct *parent;                 // the parent process
    struct mm_struct *mm;                       // Process's memory management field
    struct context context;                     // Switch here to run process
    struct trapframe *tf;                       // Trap frame for current interrupt
    uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)
    uint32_t flags;                             // Process flag
    char name[PROC_NAME_LEN + 1];               // Process name
    list_entry_t list_link;                     // Process link list 
    list_entry_t hash_link;                     // Process hash list
};
```
重点看`context`和`trapframe`。
其中`context`的结构如下：
```c
struct context {
    uint32_t eip;
    uint32_t esp;
    uint32_t ebx;
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;
};
```
里面存了一堆寄存器的值，没有段寄存器哦。


再看看`trapframe`的结构：
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
这里面就存了段寄存器。
