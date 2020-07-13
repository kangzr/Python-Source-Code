### Python中的Dict对象

C++ STL map基于RB-tree (**O(logN)**)； Python PyDictObejct 对搜索效率要求极其苛刻，因此采用散列表(hash table) **O(1)**.

#### 5.1 散列表概述

散列冲突解决：SGI STL中hash table采用开链法，Python中采用开放定址法

开放定址法：散列冲突时，采用二次探测函数f，计算下一个候选位置addr，addr可用则插入，否则，再次探测f（形成**冲突探测链**），直到找到一个可用位置。

删除某条探测链上元素时，**不能真正删除**，而是进行一种“伪删除”操作，防止探测链断裂。

#### 5.2 `PyDictObject`

##### 5.2.1 关联容器的entry

```c
[dictobject.h]
typdef struct{
    Py_ssize_t me_hash;  // 存me_key的散列值，避免每次查询重新计算一遍散列值 
    PyObject *me_key;  
    PyObject *me_valu;
}PyDictEntry;
```

entry可在3中状态间转换：Unused, Active, Dummy

- Unused: me_key和me_value都为NULL，初始太
- Active: 存储一个(key, value)后
- Dummy: (key, value)被删除之后，伪删除(防止冲突探测链中断)

##### 5.2.2 关联容器的实现

```c
[dictobject.h]
#define PyDict_MINSIZE 8
typedef struct _dictobject PyDictObject;
struct _dictobject{
    PyObject_HEAD
    Py_ssize_t ma_fill;  //元素个数 Active+Dummy
    Py_ssize_t ma_used;  //元素个数 Active
    Py_ssize_t ma_mask;  // 所拥有的entry数量
    PyDictEntry *ma_table;  // entry少于8个，ma_table域将指向ma_smalltable, 大于则指向额外申请的内存。总是有效
    PyDictEntry *(*ma_lookup) (PyDictObject *mp, PyObject *key, long hash);
    PyDictEntry ma_smalltable(PyDict_MINSIZE);  // 至少多少个entry被创建
};
```

#### 5.3 PyDictObject的创建和维护

##### 5.3.1 PyDictObject对象创建

```c
[dictobject.c]
PyObject* PyDict_New(void){
    register dictobject *mp;
    // 自动创建dummy对象
    if(dummy == NULL)
        dummy = PyString_FromString("<dummy key>");
    if (num_free_dicts){
        ... // 使用缓冲区
    }
    else{
        // 创建PyDictObject对象
        mp = PyObject_GC_NEW(dictobject, &PyDict_Type);
        BMPTY_TO_MINSIZE(mp);
    }
    mp->ma_lookup = lookdict_string;  
    return (PyObject*)mp;
}
```

ma_lookup指定了PyDictObject在entry集合中搜索某特定entry时需要进行的动作，包含散列函数一级二次探测函数实现，即PyDictObject的搜索策略。

##### 5.3.2 PyDictObject中的元素搜索

**lookdict**合**lookdict_string**两种搜索策略使用算法相同，只不过**lookdict_string**是针对`PystringObject`对象的特殊形式（也即，lookdict的特殊形式）。以`PyStringObject`对象作为`PyDictObject`对象中entry的键再Python中如此广泛和重要，所以**lookdict_string**也成为了`PyDict-Object`创建时所默认采用的搜索策略。

```c
[dictobject.c]
static dictentry* lookdict(dictobject *mp, PyObject *key, register long hash){

}
```



```c
[dictobject.c]
static dictentry* lookdict_string(dictobject *mp, PyObject *key, register long hash){
    ...
	if(!PyString_CheckExact(key)){
		mp->ma_lookup = lookdict;
		return lookdict(mp, key, hash);
	}
    // 搜索第一阶段：检查冲突链上第一个entry
    // 1. 散列，定位冲突检查链的第一个entry
    i = hash & mask;
    ep = &ep0[i];
    // 2. entry处于Unused状态，entry中的key与待搜索的key匹配
    if(ep->me_key == NULL || ep->me_key == key)
        return ep;
    // 3. 第一个entry处于dummy状态，设置freeslot
    if(op->me_key == dummy)
        freeslot = ep;
    else{
        // 4. 检查Active态entry
        if (ep->me_hash == hash && _PyString_Eq(ep->me_key, key))
            return ep;
        freeslot = NULL;
    }
    // 搜索第二阶段：遍历冲突链，检查每一个entry
    for (perturb = hash; ; perturb >>= PERTURB_SHIFT){
        
    }
}
```



**lookdict_string**可以认为是**lookdict**的一个优化版本：

- lookdict中有很多捕捉错误并处理错误的代码，因为其面对的是`PyObject*`, 而lookdict_string中，完全没有这些处理错误代码
- lookdict中使用的是非常通用的`PyObject_RichCompareBool`, 而lookdict_string使用的是`_PyString_Eq`, 简单很多。

##### 5.3.3 插入与删除

#### 5.4 PyDictObject对象缓冲池

free_dicts, 与`PyListObject`中使用的缓冲池机制一样，刚开始什么都没有，直到第一个`PyDictObject`被销毁时，这个缓冲池才开始接纳被缓冲的`PyDictObject`对象

```c
[dictobject.c]
static void dict_dealloc(register dictobject *mp){

}
```

如果`PyDictObject`对象中`ma_table`维护的是从系统堆申请的内存空间，那么Python将释放这块内存空间，归还给系统堆，如果没有从系统堆申请，而是指向`PyDictObject`固有的`ma_smalltable`， 那么只需要调整`ma_smalltable`中的对象引用计数就行。



























