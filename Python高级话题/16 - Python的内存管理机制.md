#### 16.1 内存管理构架

4层结构：

第一层：操作系统的内存管理接口，为python提供统一的raw memory的管理接口（malloc）

第二层：为了保证可以执行，对原生内存管理接口进行包装(PyMem_Malloc)

第三层：提供了创建Python对象的接口(Pymalloc机制)，包括设置对象的类型对象参数，初始化对象的引用计数值等(PyObject_Malloc, PyObject_Free, PyObject_Realloc)

第四层：对于比较常用的对象（int，string，list等），实现了对象缓冲池机制。

#### 16.2 小块空间的内存池

内存池机制（Pymalloc）：用于管理小块内存的申请和释放。从下到上分为四层结构：block、pool、arena、内存池

##### 16.2.1 Block

所有block长度都是8字节对齐

```c
// obmalloc.c
#define ALIGNMENT	8

#define SMALL_REQUEST_THRESHOLD	256	 //上限，超过则交给第一层内存管理机制

#define NB_SMALL_SIZE_CLASSES (SMALL_REQUEST_THRESHOLD / ALIGNMENT)  //  sizes class 数量
```

size of allocated block : 8(0), 16(1), 24(3), 32, 40...其中()中为pool

比如当申请一块大小为28字节的内存时，`PyObject_Malloc`从内存池中分配一个32字节的block。

##### 16.2.2 Pool

一组block称为一个pool，一个pool管理一堆有固定大小的内存块。在python中，一个pool的大小通常为一个系统内存页，一般为4K.

```c
// obmalloc.c
#define SYSTEM_PAGE_SIZE	(4 * 1024)
```

每一个pool管理大小相同的block。

如何将4KB内存改造为一个管理32字节block的pool，并取出第一块block。

freeblock会把所有离散的block串联起来。

##### 16.2.3 arena

可容纳64个pool，一个arena大小为256KB.

```c
// obmalloc.c
#define ARENA_SIZE		(256 << 10)     // 256kb

typdef uchar block;

struct arena_object {
    
    struct arena_object* nextarena;
    struct arena_object* prevarena;
};

static struct arena_object* new_arena(void){
    
}
```

arena_object的集合arenas并不是链表，而是一个数组。

pool_header管理的内存与pool_header自身是一块连续的内存，而arena_object与器管理的内存则是分离的。

##### 16.2.4 内存池

pool的三种状态

- used状态：pool中至少有一个block已经被使用，并且至少有一个block还未被使用。这种状态的pool受控于Python内部维护的usedpool数组
- full状态：所有block都已经被使用，不在arena的freepools链表中
- empty状态：所有block都未被使用，处于这个状态的pool的集合通过其pool_header中的nextpool构成一个链表，表头就是arena_object中的freepools.























