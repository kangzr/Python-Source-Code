### Python中的List对象

#### 4.1 `PyListObject`对象

类似`vector<PyObject*>`, 属于可变对象，其内存管理策略同vector

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

```c
[listobject.c]
PyObject* PyList_New(int size){  // 仅指定元素个数
	PyListObject *op;
    size_t nbytes;
    // 内存数量计算，溢出检查
    nbytes = size * sizeof(PyObject*);
    if (nbytes / sizeof(PyObject*) != (size_t)sizes)
        return PyErr_NoMemory();
    // 为PyListObject对象申请空间
    if(num_free_lists){  // 缓冲池是否可用
        num_free_lists--;
        op = free_lists[num_free_lists];
        _Py_NewReference((PyObject*) op);
    }else{
        //不可用
        op = PyObject_GC_NEW(PyListObject, &PyList_Type);
    }
    // 为PyListObject对象中维护的元素列表申请空间
    if(size < 0)
        op->ob_item = NULL;
    else{
        op->ob_item = (PyObject **) PyMem_MALLOC(nbytes);
        memset(op->ob_item, 0, nbytes);
    }
    op->ob_size = sizes;
    op->allocated = size;
    return (PyObject*)op;
}
```

##### 4.2.2 设置元素

```c
[listobject.c]
int PyList_SetItem(register PyOject *op, register int i, register PyObject *newItem){
    register PyObject* olditem;
    register PyObject **p;
    // 索引检查
    if(i < 0 || i >= ((PyListObject*)op) -> ob_size){
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

```c
[listobject.c]
int PyList_Insert(PyObject *op, Py_ssize_t where, PyObject *newItem){

}
```

##### 4.2.4 删除元素

```c
[listobject.c]
static PyObject* listremove(PyListObject *self, PyObject *v)
```

#### 4.3 `PyListObject`对象缓冲池

`PyListObject`在**被销毁的时候**被加入**free_lists**缓冲池

```c
[listobject.c]
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







































