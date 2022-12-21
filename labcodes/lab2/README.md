## 练习0：填写已有实验
    gdb调试为什么不会停在断点上？
    
## 练习1：实现 first-fit 连续物理内存分配算法（需要编程）

看一下调用关系：
- 首先是`init.c`中`kern_init()`函数会调用`pmm_init()`
- `pmm_init()`是对物理内存管理进行初始化
- `pmm_init()`中首先对物理内存管理器进行初始化，`pmm_manager->init()`
- `pmm_init()`中进行对物理内存的探测：`page_init()`
- `pmm_init()`中接下来是创建页目录表（PDT）


首先看`default_pmm.h`，这个里面就提供了一个默认物理内存管理器：`default_pmm_manager`
```c
extern const struct pmm_manager default_pmm_manager;
```
然后`default_pmm.c`中以一种面向对象的方式提供了这个管理器的实现：
```c
const struct pmm_manager default_pmm_manager = {
    .name = "default_pmm_manager",
    .init = default_init,
    .init_memmap = default_init_memmap,
    .alloc_pages = default_alloc_pages,
    .free_pages = default_free_pages,
    .nr_free_pages = default_nr_free_pages,
    .check = default_check,
};
```

pmm使用空闲链表法进行物理内存管理
其中，空闲块的结构如下：
```c
/* free_area_t - maintains a doubly linked list to record free (unused) pages */
struct list_entry {
    struct list_entry *prev, *next;
};
typedef struct list_entry list_entry_t;
typedef struct {
    list_entry_t free_list;         // the list header
    unsigned int nr_free;           // # of free pages in this free list
} free_area_t;
free_area_t free_area;
```

list的一些操作：
```c
list_init(list_entry_t *elm) {
    elm->prev = elm->next = elm;
}
```
将list的前后指针都指向自己，意思为初始化一个elem
 
`default_init_memmap`的作用：新增一个空闲物理内存块。


`pa2page`：给一个物理地址，求出管理这个物理地址的page


这个物理空闲链表的逻辑是这样的：
1. 每个物理页面都会有一个Page对应，使用`pa2page`就能完成物理地址到Page的转变
2. 每个Page中内嵌了一个`free list link`
3. 如果连续的多个Page都是空闲的，那么第一个Page的`property`就会记录有多少个空闲块
4. 并且第一个Page的`free list link` 将会添加至空闲链表
5. `free_list`就是空闲链表头，表头不存空闲块，只存一个`nr_free`，用来记录整个空闲链表中空闲块的数目


这里主要是要将空闲链表按起始地址排序，源代码没有排，我们需要修改一下。


练习2：实现寻找虚拟地址对应的页表项（需要编程）

就是个二级页表
![pic](https://img-blog.csdnimg.cn/img_convert/b184e33db8c42f78c5cebe10160092c1.png)

- 页目录表（PDT）
    - 每个段只有一个PDT，就一个页那么大
    - 页表就是数组，PDT就是PDE的数组
    - `boot_pgdir = boot_alloc_page()`
    - 
    - 页目录项目（PDE）
- 页表（PT）
    - 页表项（PTE）

- 全局描述表（GDT）
    - 又叫段描述符表
    - 寄存器GDTR存GDT的起始地址
    - 会发现，段描述符项目的段起始地址和段限长都一样，都是0x0 与 0xFFFFFFFF，这样就是Linux对x86-32段机制的使用（其实就是忽略）

地址：
- 逻辑地址
- 线性地址
- 物理地址

在 Intel 平台下，逻辑地址(logical address)是 selector:offset 这种形式，selector 是 CS 寄存器的值，offset 是 EIP 寄存器的值。如果用 selector 去 GDT( 全局描述符表 ) 里拿到 segment base address(段基址) 然后加上 offset(段内偏移)，这就得到了 linear address

如果再把 linear address 切成四段，用前三段分别作为索引去PGD、PMD、Page Table里查表，最终就会得到一个页表项(Page Table Entry)，那里面的值就是一页物理内存的起始地址，把它加上 linear address 切分之后第四段的内容(又叫页内偏移)就得到了最终的 physical address。我们把这个过程称作页式内存管理。

问题来了，为什么没提到 virtual address，这是个什么东西？其实在 Intel IA-32 手册里并没有提到这个术语，但是在内核的确是用到了这个概念，比如__va和__pa这两个宏定义。经过我的考证，virtual address就是linear address的别名，俩词汇是一个意思，内核代码和我们编程中喜欢用virtual address这个术语，而Intel手册里只用linear address这个术语。

```c
pde_t *pgdir
```
这个pgdir是数组，是页目录表
pgdir[0]是页目录表中第0项目

给定一个线性地址la，可以通过宏拆分成三个部分：
- `PDX(la)`：得到的是该线性地址在页目录表（PDT）中的索引编号
- `PTX(la)`：得到的是该线性地址在页表（PT）中的索引编号
- `PGOFF(la)`：得到的是该线性地址在页中的偏移

```c
// 页表项的类型
typedef uintptr_t pte_t;  
// 页目录项的类型
typedef uintptr_t pde_t;
```

PTE和PDE，这两个项，是32位的，由于都存的是一个页的地址，所以低12位都是0，可以用来保存额外信息。

## 练习3：释放某虚地址所在的页并取消对应二级页表项的映射（需要编程）

