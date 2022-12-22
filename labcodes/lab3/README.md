## 练习0：填写已有实验

## 练习1：给未被映射的地址映射上物理页（需要编程）

首先看一下跟vmm相关的部分结构体：
```c++
// the control struct for a set of vma using the same PDT
struct mm_struct {
    list_entry_t mmap_list;        // linear list link which sorted by start addr of vma
    struct vma_struct *mmap_cache; // current accessed vma, used for speed purpose
    pde_t *pgdir;                  // the PDT of these vma
    int map_count;                 // the count of these vma
    void *sm_priv;                   // the private data for swap manager
};

// the virtual continuous memory area(vma), [vm_start, vm_end), 
// addr belong to a vma means  vma.vm_start<= addr <vma.vm_end 
struct vma_struct {
    struct mm_struct *vm_mm; // the set of vma using the same PDT 
    uintptr_t vm_start;      // start addr of vma      
    uintptr_t vm_end;        // end addr of vma, not include the vm_end itself
    uint32_t vm_flags;       // flags of vma
    list_entry_t list_link;  // linear list link which sorted by start addr of vma
};
```

## 物理内存分配 与 虚拟内存分配的关系

我们会看到，在内核中常常有`kmalloc`和`kfree`函数的出现，这些函数的定义如下：
```c
void *
kmalloc(size_t n) {
    void * ptr=NULL;
    struct Page *base=NULL;
    assert(n > 0 && n < 1024*0124);
    int num_pages=(n+PGSIZE-1)/PGSIZE;
    base = alloc_pages(num_pages);
    assert(base != NULL);
    ptr=page2kva(base);
    return ptr;
}

void 
kfree(void *ptr, size_t n) {
    assert(n > 0 && n < 1024*0124);
    assert(ptr != NULL);
    struct Page *base=NULL;
    int num_pages=(n+PGSIZE-1)/PGSIZE;
    base = kva2page(ptr);
    free_pages(base, num_pages);
}
//alloc_pages - call pmm->alloc_pages to allocate a continuous n*PAGESIZE memory 
struct Page *
alloc_pages(size_t n) {
    struct Page *page=NULL;
    bool intr_flag;
    
    while (1)
    {
         local_intr_save(intr_flag);
         {
              page = pmm_manager->alloc_pages(n);
         }
         local_intr_restore(intr_flag);

         if (page != NULL || n > 1 || swap_init_ok == 0) break;
         
         extern struct mm_struct *check_mm_struct;
         //cprintf("page %x, call swap_out in alloc_pages %d\n",page, n);
         swap_out(check_mm_struct, n, 0);
    }
    //cprintf("n %d,get page %x, No %d in alloc_pages\n",n,page,(page-pages));
    return page;
}

//free_pages - call pmm->free_pages to free a continuous n*PAGESIZE memory 
void
free_pages(struct Page *base, size_t n) {
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        pmm_manager->free_pages(base, n);
    }
    local_intr_restore(intr_flag);
}

```

会发现，这些函数都是调用了`alloc_pages`和`free_pages`。而这些函数是直接使用的物理内存管理器`pmm_manager`直接分配的物理内存。

这些返回的物理页面`Page`，通过`ptr=page2kva(base)`函数，从物理页面变成了内核虚拟地址。这是为什么呢？

内存的分配图如下：
```c
/* *
 * Virtual memory map:                                          Permissions
 *                                                              kernel/user
 *
 *     4G ------------------> +---------------------------------+
 *                            |                                 |
 *                            |         Empty Memory (*)        |
 *                            |                                 |
 *                            +---------------------------------+ 0xFB000000
 *                            |   Cur. Page Table (Kern, RW)    | RW/-- PTSIZE
 *     VPT -----------------> +---------------------------------+ 0xFAC00000
 *                            |        Invalid Memory (*)       | --/--
 *     KERNTOP -------------> +---------------------------------+ 0xF8000000
 *                            |                                 |
 *                            |    Remapped Physical Memory     | RW/-- KMEMSIZE
 *                            |                                 |
 *     KERNBASE ------------> +---------------------------------+ 0xC0000000
 *                            |                                 |
 *                            |                                 |
 *                            |                                 |
 *                            ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 * (*) Note: The kernel ensures that "Invalid Memory" is *never* mapped.
 *     "Empty Memory" is normally unmapped, but user programs may map pages
 *     there if desired.
 *
 * */
```
而我们在`pmm_init`就建立了一个映射

```c
    // map all physical memory to linear memory with base linear addr KERNBASE
    //linear_addr KERNBASE~KERNBASE+KMEMSIZE = phy_addr 0~KMEMSIZE
    //But shouldn't use this map until enable_paging() & gdt_init() finished.
    boot_map_segment(boot_pgdir, KERNBASE, KMEMSIZE, 0, PTE_W);
```

也就是将地址0开始的物理地址，映射到虚拟地址空间中的`[KERNBASE, KERNBASE + KMEMSIZE)`中
也就是上图中的`Remapped Physical Memory`

这也是为什么`ptr=page2kva(base)`函数实现方式是下面这样的了(`__m_pa + KERNBASE`)
```c
static inline void *
page2kva(struct Page *page) {
    return KADDR(page2pa(page));
}
/* *
 * KADDR - takes a physical address and returns the corresponding kernel virtual
 * address. It panics if you pass an invalid physical address.
 * */
#define KADDR(pa) ({                                                    \
            uintptr_t __m_pa = (pa);                                    \
            size_t __m_ppn = PPN(__m_pa);                               \
            if (__m_ppn >= npage) {                                     \
                panic("KADDR called with invalid pa %08lx", __m_pa);    \
            }                                                           \
            (void *) (__m_pa + KERNBASE);                               \
        })
```

综上，我们在内核中，使用`kmalloc`，`kfree`分配的物理内存，是有虚拟地址的，是在虚拟地址空间中的`Remapped Physical Memory`这个部分。


## swap_fifo

首先看一下`swap_manager`的实现类`swap_manager_fifo`，
```c
struct swap_manager swap_manager_fifo =
{
     .name            = "fifo swap manager",
     .init            = &_fifo_init,
     .init_mm         = &_fifo_init_mm,
     .tick_event      = &_fifo_tick_event,
     .map_swappable   = &_fifo_map_swappable,
     .set_unswappable = &_fifo_set_unswappable,
     .swap_out_victim = &_fifo_swap_out_victim,
     .check_swap      = &_fifo_check_swap,
};
```
实现的是先入先出的调度方式。

这个页面替换管理器主要负责内存页面与磁盘页面的调度。

- init            
- init_mm         
- tick_event  
- map_swappable   
- set_unswappable 
- swap_out_victim 
- check_swap      


每一个被实际使用的物理页面，按其被使用的先后顺序形成队列。

当物理页面都被使用了，而又需要申请新的物理页面时，就从这个队列首部取出物理页面，将该页面的内容保存到磁盘中去，并更新对应的页表项，源页表项存的内容是该页面的物理地址，现在存的是该页面在磁盘中的位置。最后，该物理页面被用来存其他东西了。

swap_in 就是添加队列。
swap_out 取出队列。

虚拟地址空间中的页的几种情况：
- 该虚拟地址块存在
    - 有对应的物理页面
        - 对应的物理页面在内存中：正常访问
        - 对应的物理页面swap out在磁盘中：page fault
    - 没有对应的物理页面：page fault

- 该虚拟地址块不存在


看一下与PRA相关的结构
```c
struct Page {
    int ref;                        // page frame's reference counter
    uint32_t flags;                 // array of flags that describe the status of the page frame
    unsigned int property;          // the num of free block, used in first fit pm manager
    list_entry_t page_link;         // free list link
    list_entry_t pra_page_link;     // used for pra (page replace algorithm)
    uintptr_t pra_vaddr;            // used for pra (page replace algorithm)
};
// the control struct for a set of vma using the same PDT
struct mm_struct {
    list_entry_t mmap_list;        // linear list link which sorted by start addr of vma
    struct vma_struct *mmap_cache; // current accessed vma, used for speed purpose
    pde_t *pgdir;                  // the PDT of these vma
    int map_count;                 // the count of these vma
    void *sm_priv;                   // the private data for swap manager
};
list_entry_t pra_list_head;
```

`pra_list_head`：所有可以换出的页面组成的链表的头

`pra_page_link`：

`sm_priv`：每个mm_strcut都有一个这个，其值就是`&pra_list_head`

