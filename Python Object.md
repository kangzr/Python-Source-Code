### PyObject

```c
// [object.h]

/* Define pointers to support a doubly-linked list of all live heap objects. */
#define _PyObject_HEAD_EXTRA            \
    struct _object *_ob_next;           \
    struct _object *_ob_prev;
/* PyObject_HEAD defines the initial segment of every PyObject. */
#define PyObject_HEAD                   \
    _PyObject_HEAD_EXTRA                \
    Py_ssize_t ob_refcnt;               \
    struct _typeobject *ob_type;
#define PyObject_VAR_HEAD               \
    PyObject_HEAD                       \
    Py_ssize_t ob_size; /* Number of items in variable part */
/* Nothing is actually declared to be a PyObject, but every pointer to
 * a Python object can be cast to a PyObject*.  This is inheritance built
 * by hand.  Similarly every pointer to a variable-size Python object can,
 * in addition, be cast to PyVarObject*.
 */
typedef struct _object {
    PyObject_HEAD
} PyObject;
typedef struct {
    PyObject_VAR_HEAD
} PyVarObject;
#define Py_REFCNT(ob)           (((PyObject*)(ob))->ob_refcnt)
#define Py_TYPE(ob)             (((PyObject*)(ob))->ob_type)
#define Py_SIZE(ob)             (((PyVarObject*)(ob))->ob_size)

```



### PyIntObject

```c
// [intobject.h]
typedef struct {
    PyObject_HEAD
    long ob_ival;
} PyIntObject;

```



### PyStringObject

```c
// [stringobject.h]
typedef struct {
    PyObject_VAR_HEAD
    long ob_shash;
    int ob_sstate;
    char ob_sval[1];

    /* Invariants:
     *     ob_sval contains space for 'ob_size+1' elements.
     *     ob_sval[ob_size] == 0.
     *     ob_shash is the hash of the string or -1 if not computed yet.
     *     ob_sstate != 0 if the string object is in stringobject.c's
     *       'interned' dictionary; in this case the two references
     *       from 'interned' to this object are *not counted* in ob_refcnt.
     */
} PyStringObject;	


```



**经典效率问题：**使用"+"进行字符串连接效率及其低下，其根源在于Python中的PyStringObject对象是一个不可变对象，意味着一次"+"就要创建一个新的PyStringObject. 如果连接N个PyStringObject对象，就必须进行N-1次的内存申请以及内存搬运的工作。

**官方推荐做法：** 利用join对存储在list or tuple中的一组PyStringObject对象进行连接，只需要分配一次内存，执行效率大大提高。

### PyListObject

```c
// [listobject.h]
typedef struct {
    PyObject_VAR_HEAD
    // ob_item为指向元素列表的指针
    /* Vector of pointers to list elements.  list[0] is ob_item[0], etc. */
    PyObject **ob_item;

    /* ob_item contains space(总共) for 'allocated' elements.  The number
     * currently in use is ob_size(已用).
     * Invariants:
     *     0 <= ob_size <= allocated
     *     len(list) == ob_size
     *     ob_item == NULL implies ob_size == allocated == 0
     * list.sort() temporarily sets allocated to -1 to detect mutations.
     *
     * Items must normally not be NULL, except during construction when * the list is not yet visible outside the function that builds it.
     */
    // o <= ob_size <= allocated
    Py_ssize_t allocated;
} PyListObject;
```



### PyDictObject

```c
// [dictobject.h]
// 1. Unused.  me_key == me_value == NULL
// 2. Active.  me_key != NULL and me_key != dummy and me_value != NULL
// 3. Dummy.  me_key == dummy and me_value == NULL
typedef struct{
    Py_ssize_t me_hash;
    PyObject *m_key;
    PyObject *me_value;
}PyDictEntry;

typedef struct _dictobject PyDictObject;
struct _dictobject {
    PyObject_HEAD
    Py_ssize_t ma_fill;  /* # Active + # Dummy */
    Py_ssize_t ma_used;  /* # Active */

    /* The table contains ma_mask + 1 slots, and that's a power of 2.
     * We store the mask instead of the size because the mask is more
     * frequently needed.
     */
    Py_ssize_t ma_mask;

    /* ma_table points to ma_smalltable for small tables, else to
     * additional malloc'ed memory.  ma_table is never NULL!  This rule
     * saves repeated runtime null-tests in the workhorse getitem and
     * setitem calls.
     */
    PyDictEntry *ma_table;
    PyDictEntry *(*ma_lookup)(PyDictObject *mp, PyObject *key, long hash);
    PyDictEntry ma_smalltable[PyDict_MINSIZE];
};
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

### PyTupleObject

```c
// [tupleobject.h]
typedef struct {
    PyObject_VAR_HEAD
    PyObject *ob_item[1];

    /* ob_item contains space for 'ob_size' elements.
     * Items must normally not be NULL, except during construction when
     * the tuple is not yet visible outside the function that builds it.
     */
} PyTupleObject;
```

