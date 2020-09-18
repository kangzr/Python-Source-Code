### 2.1 初识`PyIntObject`对象

定长对象与变长对象。可变对象(mutable)和不可变对象(immutable)

为**immutable对象**，的 一旦创建一个`PyIntObject`，就不能改变该对象的值。

**如何针对整数对象设计一个高效机制？**

整数对象池！！！一个优雅而巧妙的整数对象缓冲池机制

在此基础上，运行时整数对象并非一个个独立对象，而是通过一定结构联结在一起的**整数对象系统**。

这种**面向特定对象的缓冲池机制**也是Python语言实现时的核心设计策略之一。几乎所有的内建对象，都有自己特有的**对象池机制**。

```c
[intobject.h]
typedef struct{
	PyObject_HEAD
	long ob_ival;  // C中原生类型long的简单包装
}PyInObject;
```

对象相关元信息实际都是保存在与对象对应的**类型对象**中。`PyInObject`---`PyInt_Type`

```c
[intobject.c]
PyTypeObject PyInt_Type = {
	PyObject_HEAD_INIT(&PyType_Type)
    0,
    "int",
    sizeof(PyInObject),
    ...
    (printfunc) int_print,   // tp_print
    ...
    (cmpfunc)int_compare,
};


typedef int (*cmpfunc)(PyObject*, PyObject*);  // 函数指针

static int int_compare(PyIntObject *v, PyIntObject *w){
    register long i = v->ob_ival;
    register long j = w->ob_ival;
    return (i > j) ? -1 : (i < j) ? 1: 0;
}


/*
  register: hints to compiler that a given variable can be put in a register. It's compiler's choice to put it in a register or not. Generally, compilers themselves do optimizations and put the variables in register.(Registers are faster than memory to access)
  
  1) If you use & operator with a register variable then compiler may give an error or warning.
 	 because when we say a variable is a register, it may be stored in a register instead of memory and accessing address of a register is invalid.
   2) register keyword can be used with pointer variables. Obviously, a register can have address of a memory location. 
   3) Register is a storage class, and C doesn’t allow multiple storage class specifiers for a variable. So, register can not be used with static
   4） Register can only be used within a block (local), it can not be used in the global scope (outside main).
   
*/
```

### 2.2 `PyIntObject对象的创建和维护`

#### 2.2.1 对象创建的3中途径

```c
Pyobject *PyInt_FromLong(long ival)
PyObject *PyInt_FromString(char *s, char **pend, int base)
#ifdef Py_USING_UNICODE
PyObject *PyInt_FromUnicode(PY_UNICODE *s, int length, int base)
#endif
```

其中后两者都会转换成浮点数，再调用`PyInt_FromFloat`.

#### 2.2.2 小整数对象

Python将小整数对应的`PyIntObject`缓存再内存中，并将其指针存放在`small_ints`中, 默认范围[-5, 257)

```c
[intobject.c]
#ifndef NSMALLPOSINTS
	#define NSMALLPOSINTS 257
#endif

#ifndef NSMALLNEGINTS
	#define NSMALLNEGINTS 5
#endif

#if NSMALLNEGINTS + NSMALLPOSINTS > 0
	static PyIntObject *small_ints[NSMALLNEGINTS + NSMALLPOSINTS];
#endif
```

#### 2.2.3 大整数对象

Python运行环境为大整数对象提供一块内存空间，由大整数轮流使用。有一个`PyIntBlock`,实现一个单向列表。

Python用一个单向链表来管理全部block的objects中所有空闲内存，**自由内存链表**的表头就是**free_list**

```c
[intobject.c]
#define BLOCK_SIZE 1000
#define BHEAD_SIZE 8
#define N_INTOBJECTS ((BLOCK_SIZE - BHEAD_SIZE) / sizeof(PyIntObject)

struct _intblock{
	struct _intblock *next;
	PyIntObject objects[N_INTOBJECTS];
};

typedef struct _intblock PyIntBlock;
typedef PyIntBlock *block_list = NULL;  // 维护整个整数对象的通用对象池
typedef PyIntObject *free_list = NULL;

/*
 * 当freelist为NULL时，需要申请一个_intblock,然后接入block_list中，
 * 因此block_list为一个单列表，每个节点为一个_intblock, 
 * 每一个_intblock可以存储N_INTOBJECTS个PyIntObject
 */
static PyIntObject *
fill_free_list(void){
    PyInObject *p, *q;
    p = (PyIntObject *) PyMem_MALLOC(sizeof(PyIntBlock));
    if (p == NULL)
        return (PyIntObject *) PyErr_NoMemory();
    ((PyIntBlock *)p)->next = block_list;
    block_list = (PyIntBlock*) p;
    p = &((PyIntBlock *)p)->objects[0];
    q = p + N_INTOBJECTS;
    while (--q)
        Py_TYPE(q) = (struct _typeobject *)(q-1);  //通过ob_type连接成单向链表
    Py_TYPE(q) = NULL;
    return p + N_INTOBJECTS - 1;
}
```

#### 2.2.4 添加和删除

`PyInt_FromLong` 步骤

```c
PyObject *
PyInt_FromLong(long ival)
{
    register PyIntObject *v;
#if NSMALLNEGINTS + NSMALLPOSINTS > 0
    // 尝试用小整数
    if (-NSMALLNEGINTS <= ival && ival < NSMALLPOSINTS) {
        v = small_ints[ival + NSMALLNEGINTS];
        Py_INTREF(v);  // ob_refcnt++
        // 省去COUNT_ALLOCS
        return (PyObject *)v;
    }
    // 通用整数池申请新的内存空间
    if (free_list == NULL) {
        if ((free_list = fill_free_list()) == NULL)
            return NULL;
    }
    v = free_list; // 取free_list中objects第一个PyIntObject
    free_list = (PyIntObject *)Py_TYPE(v); // free_list 指向下一个元素
    (void)PyObject_INIT(v, &PyInt_Type); // 将v 的类型ob_type设置为PyInt_Type。。。之前是当作指针来用
    v->ob_ival = ival;  // 设置v的值
    return (PyObject *)v;
}
```



- 如果小整数对象池机制被激活，则尝试使用小整数对象池(`small_ints`)
- 如果不能使用小整数对象池，则使用通用整数对象池(`fill_free_list`)

##### 2.2.4.1 使用小整数对象池

##### 2.2.4.2 创建通用整数对象池

```c
static PyIntObject* fill_free_list(void){
	...
}
```



`fill_free_list`：创建新的block，创建新的空闲内存，Python对fill_free_list调用不光发生在`PyInt_FromLong`首次调用，在Python运行期间，只要所有block的空闲内存被都被用完了，就会导致`free_list`为NULL。

- 申请一个新的PyIntBlock结构，连接至block_list
- 并将其中objects数组中对象通过指针(ob_type)以此连接起来，将数组转成一个单向链表。

因此，从free_list开始，沿着ob_type指针，就可以遍历`PyIntBlock`中的所有空闲内存。

##### 2.2.4.3 使用通用整数对象池

 若P1满，P2未满。free_list指向P2, 现在P1删除一个元素，P1便有空闲内存，因此下次创建新元素时，应当使用P1，而不是P2，如果实现？

不同`PyIntBlock`对象的objects中的空闲内存块是被链接在一起形成一个单向链表的，表头指针为`free_list`; 防止内存泄漏

**如何连接不同PyInBlock的空闲内存？**

也是通过int_dealloc实现。Block1满，Block2未满，Block1中删除元素，然后free_list将指向Block1中空闲位置。

```c
[intobject.c]
// 引用计数减为0时
static void int_dealloc(PyIntObject *v){
	if(PyInt_CheckExact(v)){  // 如果v为整数，则将其加入free_list中，而不是归还内存 （理论上时可以利用这个bug将系统内存吃光的。
		v->ob_type = (struct _typeobject*)free_list;
		free_list = v;
	}
	else
		v->ob_type->tp_free((PyObject*)v);
}
```

#### 2.2.5 小整数对象池的初始化

small_ints初始化。

```c
[intobject.c]
int _PyInt_Init(void){
	PyIntObject *v;
	int ival;
#if NSMALLNEGINTS + NSMALLPOSINTS > 0
	for (ival = -NSMALLNEGINTS; ival < NSMALLPOSINTS; ival++){
		if(!free_list && (free_list = fill_free_list()) == NULL)
			return 0;
		v = free_list;
		free_list = (PyInObject *)v->ob_type;
		PyObject_INIT(v, &PyInt_Type);
		v->ob_ival = ival;
		small_ints[ival + NSMALLNEGINTS] = v;
	}
#endif
	return 1;
}
```

### 2.3 Hack PyIntObject





































