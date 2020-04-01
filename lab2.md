
# lab2

## mm

## pmap.c

### 全局变量

- **maxpa**(u_long): 最大可用物理内存(0x400_0000,64MB), 实验用的操作系统内存大小一共就64MB
- **npage**(u_long): 物理内存全部加在一起共划分为多少页(maxpa/页大小)
- **basemem**: 内存共有多少字节;基础内存(在lab2设置为64MB)
- **extmem**: 扩展内存共多少字节(0)

- **boot_pgdir**(Pde *) : unknown

- **pages**(struct Page *): 物理页面结构体
- **freemem**(static u_long )

C语言的指针本质是虚拟地址(即与下面频频出现的(内核)虚拟地址是统一的), 经过解引用可以得到其对应的物理地址的内容, 这应该是透彻理解下面函数的核心

### static Pte \**boot_pgdir_walk*

```C
static Pte *boot_pgdir_walk(Pde *pgdir, u_long va, int create)
{

    Pde *pgdir_entryp;
    Pte *pgtable, *pgtable_entry;

    /* Step 1: Get the corresponding page directory entry and page table. */
    /* Hint: Use KADDR and PTE_ADDR to get the page table from page directory
     * entry value. */

    pgdir_entryp = pgdir + PDX(va);
    pgtable = (Pte *)KADDR(PTE_ADDR(*pgdir_entryp));

    /* Step 2: If the corresponding page table is not exist and parameter `create`
     * is set, create one. And set the correct permission bits for this new page
     * table. */
    if((*pgdir_entryp&PTE_V)==0){
        if(create){
            pgtable = (Pte *)alloc(BY2PG,BY2PG,1);
            *pgdir_entryp = PADDR(pgtable)|PTE_V;
        } else {
            return (Pte) 0;
        }
    }

    /* Step 3: Get the page table entry for `va`, and return it. */
    pgtable_entry = pgtable + PTX(va);
    return pgtable_entry;
}
```

> 实现在 mm/pmap.c  
> 是一个私有函数  
> 用于在页表初始化之前的页表分配，返回对应的页表项虚地址  
Page Table/Directory Entry--PTE/PDE

#### 参数

- Pte \***pgdir**:    页目录页虚拟基地址(pgdir:page directory)
- u_long **va**:      待查询虚拟地址(va:virtual address)  
对于一个32位的虚存地址，其31-22位表示的是页目录项的索引，21-12位表示的是页表项的索引，11-0位表示的 是该地址在该页面内的偏移。
- int **create**:     如果待查询地址未分配物理页面是否进行分配操作

#### 内部变量

- Pde \***pgdir_entryp**:   va高十位去索引的页目录项的虚拟地址, 从页目录查询到的页表的入口(基地址, 虚拟地址)
- Pte \***pgtable** :       va所在的虚拟页面对应页表项所在页表页的基地址(虚拟地址)
- Pte \***pgtable_entry** : 物理页面入口(页表项的虚拟地址)

```C
pgdir_entryp = pgdir + PDX(va);
//pgdir_entryp = &pgdir[PDX(va)];
pgtable = (Pte *)KADDR(PTE_ADDR(*pgdir_entryp));
```

> PDX(va) 定义在 include/mm  
> **#define PDX(va) ((((u_long)(va))>>22) & 0x03FF)**  
> 含义是取va[31:22], 页目录项在页目录页的索引, 即高10字节

然后pgdir[PDX(va)] 也就是我们要找的页目录项  
注意，此处的pgdir是虚地址, 可以直接解引用(当数组来用), 但是内部存储的页表信息是物理地址+标志位  
pgdir_entryp是va对应的虚拟页面的页表所在页表页对应的页目录项的指针, *pgdir_entryp即待查询虚拟地址的所在虚页面对应的**页目录项**(页表可能不存在, 关系可能尚未建立, 需要分配之后填入页目录, 因此获取的是指针)

> include/mmu.h
> **#define PTE_ADDR(pte) ((u_long)(pte)&~0xFFF)**
> 用于将页表入口的低12位清空(因为这些地方存储着标志位), 获得**页表的物理基地址**
>
> include/mmu.h

```C
#define KADDR(pa)                       \
    ({                              \
        u_long ppn = PPN(pa);                   \
        if (ppn >= npage)                   \
            panic("KADDR called with invalid pa %08lx", (u_long)pa);\
        (pa) + ULIM;                    \
    })

#define PADDR(kva)                      \
    ({                              \
        u_long a = (u_long) (kva);              \
        if (a < ULIM)                   \
            panic("PADDR called with invalid kva %08lx", a);\
        a - ULIM;                   \
    })
```

> KADDR(pa)用于把物理地址pa转换成内核虚拟地址Kernel ADDress  
> PADDR(kva)用于把内核虚拟地址转换为物理地址Physical ADDress

```C
    if((*pgdir_entryp&PTE_V)==0){
        if(create){
            pgtable = (Pte *)alloc(BY2PG,BY2PG,1);
            *pgdir_entryp = PADDR(pgtable)|PTE_V;
        } else {
            return 0;
        }
    }
```

取出页表目录项的内容，也就是**页表入口物理地址+标志位**，然后与**PTE_V进行按位与**判断此位是否有效

> include/mmu.h
> **#define PTE_V       0x0200**
> 用于有效位判断(第十位为1, 其他为0, 按位与)

如果页目录项无效(该页表页还没有分配物理内存):

- 如果create参数为1,
  - 调用alloc, 分配一页大小的空间, 让pgtable(虚拟地址)指向刚分配的空间(这里分配出了一个页表页)
  - 将pgtable转换为物理地址, 抹去低位, 标志位仅赋有效位后写入pgdir_entryp指向的(虚拟)地址(这里是从c语言角度来看，在转换为汇编语言后应该是经过MMU翻译为物理地址), 从而页目录项安排好了
- 如果create参数为0, 不分配页表页
  - 则直接返回0

```C
    pgtable_entry = pgtable + PTX(va);
    //pgtable_entry = &pgtable[PTX(va)];
    return pgtable_entry;
```

> PTX(va) 定义在 include/mm  
> **#define PTX(va)     ((((u_long)(va))>>12) & 0x03FF)**  
> 用来取va[21:12], 即找页表项在页表页(看做大数组)中的编号

pgtable_entry: 物理页面入口(页表项的虚拟地址)  
通过PTX(va)找到页表项的编号，然后取得页表项虚地址(pgtable_entry)并返回

个人看法: 这里如果以及页目项不存在的话, 是没有建立页表项的, 只是返回了va所在页面对应的页表项在虚存的地址

### *int pgdir_walk*

```C
int
pgdir_walk(Pde *pgdir, u_long va, int create, Pte **ppte)
{
    Pde *pgdir_entryp;
    Pte *pgtable;
    struct Page *ppage;

    /* Step 1: Get the corresponding page directory entry and page table. */
    pgdir_entryp = &pgdir[PDX(va)];
    pgtable = (Pte *)KADDR(PTE_ADDR(*pgdir_entryp));

    /* Step 2: If the corresponding page table is not exist(valid) and parameter `create`
     * is set, create one. And set the correct permission bits for this new page
     * table.
     * When creating new page table, maybe out of memory. */
    if(((*pgdir_entryp&PTE_V)==0)&&create){
        if(page_alloc(&ppage)==-E_NO_MEM){
            *ppte = 0;
            return -E_NO_MEM;
        }
        ppage->pp_ref++;
        *pgdir_entryp = PADDR((Pte)page2kva(ppage)|PTE_R|PTE_V);
    }

    /* Step 3: Set the page table entry to `*ppte` as return value. */
    if((*pgdir_entryp) == 0){
        *ppte = 0;
    }else{
        pgtable=(Pte *)KADDR(PTE_ADDR(*pgdir_entryp));
        *ppte = &pgtable[PTX(va)];
    }
    return 0;
}
```

> 判断va所对应的页表项所在的页表页是否已经建立, *ppte接收页表项的虚拟地址(此处其实应该更多当指针用)

#### 参数

- Pte \***pgdir**:      页目录页虚拟基地址(pgdir:page directory)
- u_long **va**:        待查询虚拟地址(va:virtual address)  
对于一个32位的虚存地址，其31-22位表示的是页目录项的索引，21-12位表示的是页表项的索引，11-0位表示的 是该地址在该页面内的偏移。
- int **create**:       如果待查询地址未分配物理页面是否进行分配操作
- Pte \*\***ppte**:     ppte是C程序指向页表项的虚拟地址(Pte *类型)的指针, *ppte即是页表项的虚拟地址(为0是什么含义尚不清楚)

#### 内部变量

- Pde \***pgdir_entryp**:   va高十位去索引的页目录项的虚拟地址, 从页目录查询到的页表的入口(基地址, 虚拟地址)
- Pte \***pgtable** :       va所在的虚拟页面对应页表项所在页表页的基地址(虚拟地址)
- Pte \***pgtable_entry** : 物理页面入口(页表项的虚拟地址)

```C
pgdir_entryp = pgdir + PDX(va);
//pgdir_entryp = &pgdir[PDX(va)];
pgtable = (Pte \*)KADDR(PTE_ADDR(\*pgdir_entryp));
```

> PDX(va) 定义在 include/mm  
> **#define PDX(va) ((((u_long)(va))>>22) & 0x03FF)**  
> 含义是取va[31:22], 页目录项在页目录页的索引, 即高10字节

然后pgdir[PDX(va)] 也就是我们要找的页目录项  
注意，此处的pgdir是虚地址, 可以直接解引用(当数组来用), 但是内部存储的页表信息是物理地址+标志位  
pgdir_entryp是va对应的虚拟页面的页表所在页表页对应的页目录项的指针, *pgdir_entryp即待查询虚拟地址的所在虚页面对应的**页目录项**(页表可能不存在, 关系可能尚未建立, 需要分配之后填入页目录, 因此获取的是指针)

> include/mmu.h
> **#define PTE_ADDR(pte) ((u_long)(pte)&~0xFFF)**
> 用于将页表入口的低12位清空(因为这些地方存储着标志位), 获得**页表的物理基地址**
>
> include/mmu.h

```C
#define KADDR(pa)                       \
    ({                              \
        u_long ppn = PPN(pa);                   \
        if (ppn >= npage)                   \
            panic("KADDR called with invalid pa %08lx", (u_long)pa);\
        (pa) + ULIM;                    \
    })

#define PADDR(kva)                      \
    ({                              \
        u_long a = (u_long) (kva);              \
        if (a < ULIM)                   \
            panic("PADDR called with invalid kva %08lx", a);\
        a - ULIM;                   \
    })
```

> KADDR(pa)用于把物理地址pa转换成内核虚拟地址Kernel ADDRess  
> PADDR(kva)用于把内核虚拟地址转换为物理地址Physical ADDRess

```C
    if(((*pgdir_entryp&PTE_V)==0)&&create){
        if(page_alloc(&ppage)==-E_NO_MEM){
            *ppte = 0;
            return -E_NO_MEM;
        }
        ppage->pp_ref++;
        *pgdir_entryp = PADDR((Pte)page2kva(ppage)|PTE_R|PTE_V);
    }
```

取出页表目录项的内容，也就是**页表入口物理地址+标志位**，然后与**PTE_V进行按位与**判断此位是否有效

> include/mmu.h
> **#define PTE_V       0x0200**
> 用于有效位判断(第十位为1, 其他为0, 按位与)

如果页目录项无效且create参数为1(该页表页还没有分配物理内存):

- 调用page_alloc, 分配一页大小的空间, 如果没有内存了, page_alloc返回错误值
  - *ppte被赋为0, 函数返回-E_NO_MEM
- 如果有空闲, 则取空闲页面链表的第一个赋给ppage
  - ppage(不是页面地址，而是通过链表管理的结构体的指针)指向的页面的使用次数+1;
  - 通过page2kva找到ppage指向的页面的内核虚拟基地址(4KB对齐), 赋脏位和有效位, 再通过PADDR将该虚拟地址的物理地址(这里PADDR没有改变低位的性质, 因此可以先赋若干标志位)连同标志位(这样就形成了一个页目录项)写入*pgdir_entryp

```C
    if((*pgdir_entryp) == 0){
        *ppte = 0;
    }else{
        pgtable=(Pte \*)KADDR(PTE_ADDR(\*pgdir_entryp));
        *ppte = &pgtable[PTX(va)];
    }
    return 0;
```

> PTX(va) 定义在 include/mm  
> **#define PTX(va)     ((((u_long)(va))>>12) & 0x03FF)**  
> 用来取va[21:12], 即找页表项在页表页(看做大数组)中的编号

- 如果*pgdir_entryp为0(笔者思考如果create为1, *pgdir_entryp为0的话, 若是没有内存应该直接返回了, 那么到了这一步还能为0只有可能是本来为0, 即未分配页表页, 而且create为0(不创建)), *ppte被赋为0
- 否则
  - 页目录项(*pgdir_entryp)通过PTE_ADDR(低位与0)获得物理页面基地址, 再将该物理地址通过KADDR转换为内核虚拟地址, (作为指针(强调这个是因为C语言的指针机制))赋给pgtable
  - 通过PTX(va)获得页表页(看成以四字节为一元素大小的1024个元素的大数组)内的索引(编号), 通过索引页面页获得页表项(pgtable[PTX(va)]), 其虚拟地址(通过C语言的取址机制&pgtable[PTX(va)]获得, 其实相当于pgtable+PTX(va))返回给*ppte

### void boot_map_segment(Pde *pgdir, u_long va, u_long size, u_long pa, int perm)

```C
void boot_map_segment(Pde *pgdir, u_long va, u_long size, u_long pa, int perm)
{
    int i, va_temp;
    Pte *pgtable_entry;

    /* Step 1: Check if `size` is a multiple of BY2PG. */
    if(size%BY2PG != 0){
        printf("Boot map segment failed!\n");
        return ;
    }

    /* Step 2: Map virtual address space to physical address. */
    /* Hint: Use `boot_pgdir_walk` to get the page table entry of virtual address `va`. */
    for(i = 0; i<size; i += BY2PG){
        va_temp = va + i;
        pgtable_entry = boot_pgdir_walk(pgdir, va_temp, 1);
        *pgtable_entry = (pa+i)|perm|PTE_V;
    }
}
```

> 定义于mm/pmap.h
> 实现于mm/pamp.c  
> 用于将[va, va+size)的虚拟地址映射到物理地址[pa,pa+size)中, 并根据perm对这个地址标记位更新--地址以页为单位, 核心是建立(写入)页表项

参数:  

- Pde *pgdir:   页目录基地址(虚拟地址)
- u_long va:    虚拟地址(起始分配地址)
- u_long size:  分配尺寸
- u_long pa:    物理地址(起始分配地址)
- int perm:     标志位

局部变量:

- int i:                用于循环, 表示已经映射好了多少空间
- int va_temp:          用于循环, 当前待映射虚拟地址空间首地址
- Pte *pgtable_entry:   用于循环, 页表入口(页表项虚拟地址)

```C
if(size%BY2PG != 0){
        printf("Boot map segment failed!\n");
        return ;
    }
```

前置要求：size是BY2PG(一页的字节大小)的整数倍  
检查size是不是BY2PG(4096)的整数倍, 不是就报错并直接返回

```C
for(i = 0; i<size; i += BY2PG){
        va_temp = va + i;
        pgtable_entry = boot_pgdir_walk(pgdir, va_temp, 1);
        *pgtable_entry = (pa+i)|perm|PTE_V;
    }
```

- 通过`boot_pgdir_walk(pgdir,va_temp,1);`获取虚拟地址(va_temp)对应的页表项的虚拟地址并赋给pgtable_entry(其实就是页表项的指针)(, 如果还没建立页表项所在的页表页, 就建立页表页, 然后把页表项所在地址返回并赋给pgtable_entry);
- 将物理地址(提供实页号)赋标志位(perm)和有效位(PTE_V), 写入*pgtab_entry;

### int page_insert(Pde *pgdir, struct Page *pp, u_long va, u_int perm)

```C
int
page_insert(Pde *pgdir, struct Page *pp, u_long va, u_int perm)
{
    u_int PERM;
    Pte *pgtable_entry;
    PERM = perm | PTE_V;

    /* Step 1: Get corresponding page table entry. */
    pgdir_walk(pgdir, va, 0, &pgtable_entry);

    if (pgtable_entry != 0 && (*pgtable_entry & PTE_V) != 0) {
        if (pa2page(*pgtable_entry) != pp) {
            page_remove(pgdir, va);
        } else  {
            tlb_invalidate(pgdir, va);
            *pgtable_entry = (page2pa(pp) | PERM);
            return 0;
        }
    }

    /* Step 2: Update TLB. */

    /* hint: use tlb_invalidate function */


    /* Step 3: Do check, re-get page table entry to validate the insertion. */

    /* Step 3.1 Check if the page can be insert, if can’t return -E_NO_MEM */

    /* Step 3.2 Insert page and increment the pp_ref */

    return 0;
}
```

参数

- Pde *pgdir：          页目录基地址(虚拟地址)
- struct Page *pp：     包含页面信息的结构体的指针
- u_long va:            虚拟地址
- u_int perm:           标志位

内部变量:

- u_int PERM:           标志位进一步处理
- Pte *pgtable_entry:   页表项的虚拟地址(同时也看做指针————由C语言指针解引用机制, \*pgtable_entry直接就是页表项)

> 原型在include/pmap.h中
> 实现在mm/pmap.c中
> 用来将va虚拟地址和其要对应的物理页pp的映射关系以perm的权限设置加入页目录

```C
pgdir_walk(pgdir, va, 0, &pgtable_entry);

if (pgtable_entry != 0 && (*pgtable_entry & PTE_V) != 0) {
        if (pa2page(*pgtable_entry) != pp) {
            page_remove(pgdir, va);
        } else  {
            tlb_invalidate(pgdir, va);
            *pgtable_entry = (page2pa(pp) | PERM);
            return 0;
        }
    }
```

- 通过**pgdir_walk**给pgtable_entry赋va对应的页表项的虚拟地址(指针)(传入&pgtable_entry, 即pgtable_entry的指针, 即可在函数内部给其赋值), 判断va是否有对应的有效的页表项(va是否已经有了映射的物理地址), 如果有效(pgtable_entry不为0且页表项有效位是1)
  - 取pgtable_entry指向的页表项(\*pgtable_entry), 通过PPN(\*pgtable_entry)取实页号作为索引去在pages大数组里找, 返回该项指针(其实是虚拟地址, 但是考虑到实际含义, 此处应理解为取了指针备用),
    - 如果该实页面对应的页面结构体指针与传入的作为参数的指针不相等, 即va已经有了映射的页面物理且不为我们即将准备为其安排的物理页面的页面, 就解除va的页表项的映射关系(页表项置0, 减少物理页面的引用次数, 更新TLB),
    - 否则, 说明已经建立与实页面映射关系, 且就是我们希望的那个实页面, 则更新TLB, 页表项写入pp(结构体指针)所指向的物理页面的物理地址并赋标志位(其实这里有个对称性函数处理的特殊关系(p&~0xFFF=page2pa(pa2page(p))), 有些复杂尚不讨论)

```C
    tlb_invalidate(pgdir, va);

    /* Step 3: Do check, re-get page table entry to validate the insertion. */
    /* Step 3.1 Check if the page can be insert, if can’t return -E_NO_MEM */
    if(pgdir_walk(pgdir,va,1,&pgtable_entry)==-E_NO_MEM){
        return -E_NO_MEM;
    }

    /* Step 3.2 Insert page and increment the pp_ref */
    *pgtable_entry = (page2pa(pp) | PERM);
    pp->pp_ref++;

    return 0;
```

页表项无效或va已经映射好物理地址但不是想要的(已经移除映射关系),

- 更新TLB(, 用tlb_invalidate(pgdir, va))???????????
- 重新用pgdir_walk再找一遍, 给pgtable_entry赋va的对应的页表项的地址(如果页表项所在页表页未建立则建立)
- 页表项填入page2pa(pp)(赋标志位)
- pp结构体代表的页面引用次数++
- 返回0

