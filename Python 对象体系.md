### PyObject

Everything is Object

```
[object.h]
//python 对象机制的核心基石
typedef struct _object{
	PyObject_HEAD
}PyObject;
```



### PyIntObject



### PyStringObject



**经典效率问题：**使用"+"进行字符串连接效率及其低下，其根源在于Python中的PyStringObject对象是一个不可变对象，意味着一次"+"就要创建一个新的PyStringObject. 如果连接N个PyStringObject对象，就必须进行N-1次的内存申请以及内存搬运的工作。

**官方推荐做法：** 利用join对存储在list or tuple中的一组PyStringObject对象进行连接，只需要分配一次内存，执行效率大大提高。

### PyListObject

### PyDictObject

```c
typedef struct{
    Py_ssize_t me_hash;
    PyObject *m_key;
    PyObject *me_value;
}PyDictEntry;
```



**entry的三种状态**

**Unused**: me_key==NULL;me_vale==NULL

**Active**:me_key!=NULL, dummy; me_value!=NULL

**Dummy**:me_key==dummy;me_value=NULL

```mermaid
graph LR
A[Unused]--insert-->B
B[Active]--delete-->C
C[Dummy]--insert-->B
```

**PyDictObject两种搜索策略:** 

- lookdict : 通用

- lookdict_string：针对key为string

