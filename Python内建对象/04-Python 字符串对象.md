### Python中的字符串对象

字符串属于可变对象，即其大小只有在创建的时候才能确定，但是一旦创建后，是不可以改变的。

因此Python可分为可变对象和不可变对象（标准：创建前是否可知?）,而可变对象又可分为不变和不可变：如string/tuple为不可变，list为可变

#### 3.1 `PyStringObject `与 `PyString_Type`

`PyStringObject` 是一个拥有可变对象长度的对象，其一旦创建，**该对象内维护的字符串就不能在改变**，这一特性使得其可作为**dict的key**值，

同时也使得一些**字符串操作效率大大降低**(比如连接)。

```c
[stringobject.h]
typdef struct{
	PyObject_VAR_HEAD  //其中ob_size保存对象大小，比如"Python": ob_size为6
	long ob_shash;  // 存hash值，避免重复计算，初始值为-1
	int ob_sstate;  // 标记是否被intern机制处理过(该机制让python效率提升20%)
	char ob_sval[1];  // ob_sval作为一个指针指向一段内存，保存着这个字符串对象所维护的实际字符串, 必须满足ob_sval[ob_size] = '\0';
}PyStringObject;


[stringobject.c]
static long string_hash(PyStringObject *a){
    register int len;
    register unsigned char *p;
    register long x;
    if(a->ob_shash != -1) 
        return a->ob_shash;
    len = a->ob_size;
    p = (unsigned char *) a->ob_sval;
    x = *p << 7;
    while (--len >= 0)
        x = (1000003 * x) ^ *p++;
    x ^= a->ob_size;
    if (x == -1)
        x = -1;
    a->ob_shash = x;
    return x;
}
```

#### 3.2 创建 `PyStringObject`对象

```c
[stringobject.c]
PyObject* PyString_FrmString(const char *str){
	register size_t sizes;
    register PyStringObject *op;
    
    // 判断字符串长度
    size = strlen(str);
    if (size > PY_SSIZE_T_MAX)  // 平台最大值
        return NULL;
    
    // 处理null string, nullstring专门负责处理NULL，intern机制共享。
    if (size == 0 && (op = nullstring) != NULL)
        return (PyObject*)op;
    
    // 处理字符
    if (size == 1 && (op = characters[*str & UCHAR_MAX]) != NULL)
        return (PyObject*)op;
    
    // 创建新的PyStringObject对象，并初始化
    op = (PyStringObject *)PyObject_MALLOC(sizeof(PyStringObject) + size);
    PyObject_INIT_VAR(op, &PyString_Type, size);
    op->ob_shash = -1;
    op->ob_sstate = SSTATE_NOT_INTERNED;
    memcpy(op->ob_val, str, size+1);
    ...
    return (PyObject*) op;
}
```

#### 3.3 字符串对象的intern机制

在创建字符串时，Python会首先在interned(已经interned的map中查找)，如果存在，直接返回，否则创建一个新的。

判断两个`PyStringObject`对象是否相同，如果被intern了，秩序检查对应的`PyObject*`是否相同即可。

#### 3.4 字符缓冲池

`PyStringObject` 对象池设计了对象池**characters**:

`static PyStringObject *characters[UCHAR_MAX+1]`

- 创建`PyStringObject`对象<string P>;
- 对对象<string p> 进行intern操作
- 将对象<string P> 缓存至字符缓冲池中

#### 3.5 `PyStringObject` 效率相关问题

“+” 操作符对字符串进行连接时，会调用string_concat函数，N个`PyStringObject`, 就会又N次内存申请

"".join(list)，会调用string_join,  N个PyStringObject也只申请一次内存，会计算list中所有字符串的总长度，然后申请一次内存。























































