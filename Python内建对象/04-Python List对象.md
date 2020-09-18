### Python中的List对象

#### 4.1 `PyListObject`对象

类似`vector<PyObject*>`, 属于可变对象，其内存管理策略同vector.

如果当前内存新分配内存 >= 新内存(不需要扩容) 且 新内存 >已有内存的一半(不需要缩容)。

新分配算法`(newsize) >> 3 + (newsize < 9 ? 3 : 6)`



```c
[listobject.h]
typedef struct{
	PyObject_VAR_HEAD  // ob_size 实际元素  同vector中size
    // ob_item 指向元素列表的指针，list[0] ==  ob_item[0]
    PyObject **ob_item;    // 内存块首地址
    int allocated;  // 可容纳元素总数/总内存   0<=ob_size<=allocated   len(list) == ob_size  同vector中capacity
}PyListObject;
```

#### 4.2 `PyListObject`对象的创建和维护

##### 4.2.1 创建对象

`size_t`，64位中`typedef unsigned long size_t`

缓存池`new_free_lists`中最大个数由`PyList_MAXFREELIST`定义

```c
[listobject.c]
PyObject* PyList_New(int size){  // 仅指定元素个数
	PyListObject *op;
    size_t nbytes;
    // 内存数量计算，溢出检查

    if ((size_t)size > PY_SIZE_MAX / sizeof(PyObject *))
        return PyErr_NoMemory();    
   	nbytes = size * sizeof(PyObject*);
    // 为PyListObject对象申请空间
    if(num_free_lists){  // 缓冲池是否可用
        num_free_lists--;
        op = free_lists[num_free_lists];
        _Py_NewReference((PyObject*) op);
    }else{
        //不可用
        op = PyObject_GC_NEW(PyListObject, &PyList_Type);
        if (op == NULL)
            return NULL;
    }
    // 为PyListObject对象中维护的元素列表申请空间
    if(size <= 0)
        op->ob_item = NULL;
    else{
        op->ob_item = (PyObject **) PyMem_MALLOC(nbytes);
        memset(op->ob_item, 0, nbytes);
    }
    op->ob_size = size;
    op->allocated = size;
    return (PyObject*)op;
}
```

##### 4.2.2 设置元素

list[3] = 100 会调用`PyList_SetItem`

```c
[listobject.c]
int PyList_SetItem(register PyOject *op, register int i, register PyObject *newItem){
    register PyObject* olditem;
    register PyObject **p;
    // 类型检查
    // 索引检查
    if(i < 0 || i >= PySize(op)){
        Py_XDECREF(newitem);
        PyErr_SetString(PyExc_IndexError, "out of range");
        return -1;
    }
    // 设置元素
    p = ((PyListObject*) op)->ob_item + i;
    olditem = *p;
    *p = newItem;
    Py_XDECREF(olditem);
    return 0;
}
```

##### 4.2.3 插入元素

list.insert(1) 会调用`PyList_Insert`接口
list.append

```c
[listobject.c]
static list_resize(PyListObject *self, Py_ssize_t newsize){
	PyObject **items;
    size_t nwe_allocated;
    Py_ssize_t allocated = self->allocated;
    // 如果allocated比newsize大，且newsize大于allocated的一半，就不需要调整大小.
    if (allocated >= newsize && newsize >= (allocated >> 1)){
        Py_SIZE(self) = newsize;
        return 0;
    }
    //the growth pattern is 0, 4, 8 , 16, 25, 35, 46, 58, 72..
    new_allocated = (newsize >> 3) + (newsizes < 9 ? 3 : 6);
    // check
    if (new_allocated > PY_SIZE_MAX - newsize){
        PyErr_NoMemory();
        return -1;
    }else {
        new_allocated += newsize;
    }
    
    if (newsize == 0)
        new_allocated = 0;
    items = self->ob_item;
    if (new_allocated <= (PY_SIZE_MAX / sizeof(PyObject *)))
        PyMem_RESIZE(items, PyObject*, new_allocated);
    else
        items = NULL;
    if(items == NULL){
        PyErr_NoMemory();
        return -1;
    }
    self->ob_item = items;
    Py_SIZE(self) = newsize;
    self->allocated = new_allocated;
    return 0;
}

static int
ins1(PyListObject *self, Py_ssize_t where, PyObject *v){
    Py_ssize_t i, n = Py_SIZE(self);
    PyObject **items;
    // 一堆检查
    // 调整list大小
    if (list_resize(self, n+1) == -1)
        return -1;
    // 处理小于0的情况
    if(where < 0) {
        where += n;
        if (where < 0)
            where = 0;
    }
    if (where > n)
        where = n;
    items = self->ob_item;
    for (i = n; --i >= where; )
        items[i+1] = items[i];
    Py_INCREF(v);
    items[where] = v;
    return 0;
}

int PyList_Insert(PyObject *op, Py_ssize_t where, PyObject *newItem){
	// 类型检查
    return ins1((PyListObject) *op, where, newItem);
}
```

##### 4.2.4 删除元素

list.remove

```c
[listobject.c]
static PyObject* listremove(PyListObject *self, PyObject *v){
    ...
    list_ass_slice
    ...
}
```

#### 4.3 `PyListObject`对象缓冲池

`PyListObject`在**被销毁的时候**被加入**free_lists**缓冲池

```c
[listobject.c]
static PyListObject *free_listp[PyList_MAXFREELIST];

static void list_dealloc(PyListObject *op){
    int i;
    // 销毁PyListObject对象维护的元素列表
    if(op->ob_item != NULL){
        i = op->ob_size;
        while(--i >= 0)
            Py_XDECREF(op->ob_item[i]);
        PyMem_FREE(op->ob_item);
    }
    // 释放PyListObject自身，加入free_lists
    if(num_free_lists < MAXFREELISTS && PyList_CheckExact(op))
        free_lists[num_free_lists++] = op;
    else
        op->ob_type->tp_free((PyObject*)op);
}
```







































