字符串属于可变对象，即其大小只有在创建的时候才能确定，但是**一旦创建后，是不可以改变的**。

因此Python可分为变长对象(string ,list,tuple)和定长对象(int)（标准：创建前是否可知?）,而**变长对象**又可分为可变对象(list)和不可变对象(string/tuple)

#### 3.1 `PyStringObject `与 `PyString_Type`

`PyStringObject` 是一个拥有可变对象长度的对象(不同字符串长度可不一致)；

其一旦创建，**该对象内维护的字符串就不能在改变**，这一特性使得其可作为**dict的key**值，同时也使得一些**字符串操作效率大大降低**(比如连接)。

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

查看对象的大小可以通过`__sizeof__`接口，int，string等都可以。

```c
[stringobject.c]
//注意这里的str并不是我们字符串，而是其变量。比如a="python", str就是a
PyObject* PyString_FromString(const char *str){
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
    
    // 创建新的PyStringObject对象，并初始化，因此就算是用interned机制也是会先创建再销毁
    op = (PyStringObject *)PyObject_MALLOC(sizeof(PyStringObject) + size);
    PyObject_INIT_VAR(op, &PyString_Type, size);
    op->ob_shash = -1;
    op->ob_sstate = SSTATE_NOT_INTERNED;
    memcpy(op->ob_val, str, size+1);
    /*intern share short strings */
    if (size == 0){ // 空字符串通过intern机制共享
        PyObject *t = (PyObject *)op;
        PyString_InternInPlace(&t);
        op = (PyStringObject *)t;
        nullstring = op;
        Py_INCREF(op);
    } else if (size == 1){  // 一个字符的字符串通过intern机制进行共享
        PyObject *t = (PyObject *)op;
        PyString_InternPlace(&t);
        op = (PyStringObject *)t;
        characters[*str & UCHAR_MAX] = op;
        Py_INCREF(op);
    }
    return (PyObject*) op;
}
```

`PyString_FromStringAndSize`不要求传入的`str`以`NUl('\0')`结尾。

#### 3.3 字符串对象的intern机制

在创建字符串时，Python会首先在interned(已经interned的map中查找)，如果存在，直接返回改对象引用（但是也会先创建，再销毁），否则创建一个新的。

判断两个`PyStringObject`对象是否相同，如果被intern了，秩序检查对应的`PyObject*`是否相同即可。

```c
[stringobject.c]
void PyString_InterInPlace(PyObject **p)
{
    register PyStringObject *s = (PyStringObject *)(*p);
    PyObject *t;
    if (!PyString_CheckExact(s))
        return;
    if (PyString_CHECK_INTERNED(s))
        return;
    if (interned == NULL)
        interned = PyDict_New();  //interned为一个Dict
    // 检查PyStringObject对象s是否存在对应的intern后的PyStringObject对象
    t = PyDict_GetItem(interned, (PyObject*)s);
    if (t) {
        /*interned中存在与s字符串相同的对象t，则增加t的引用计数，并把*p指向t,*/
        Py_INCREF(t);
        /**p指向t，*p引用计数减一*/
        Py_SETREF(*p);
        return;
    }
    // 这里会对s有两次+1操作，有两次对a的引用。因此后续需要对s的引用减2
    if (PyDict_SetItem(interned, (PyObject* )s, (PyObject *)s) < 0){
        PyErr_Clear();
        return;
    }
    Py_REFCNT(s) -= 2;
    PyString_CHECK_INTERNED(s) = SSTATE_INTERNED_MORTAL;
}

// A的引用计数减为0后，系统会销毁a，同时会在interned中删除指向a的指针
static void string_dealloc(PyObject *op)
{
    switch (PyString_CHECK_INTERNED(op)) {
        case SSTATE_NOT_INTERNED:
            break;
        case SSTATE_INTERNED_MORAL:
            op->ob_refcnt = 3;
            if (PyDict_DelItem(interned, op) != 0)
                Py_FatalError("deleteion of interned string failed");
            break;
        case SSTATE_INTERNED_IMMORTAL:  //永不销毁 
            Py_FatalError("Inconsistent interned string state.");
    }
    op->ob_type->tp_free(op);
}
```

当对a进行intern机制时，会先检查interned这个dict中是否存在对象b，满足其原生字符串与a相同。

如果存在b，a的Pyobject指针指向b，且a的引用计数减一，因此a是一个临时创建的对象。

如果不存在，则将a记录到interned中(PyDict_SetItem).



**Q1: 为啥加入非数字或大小写字符则不会intern？为啥PyString_InterInPlace中指定size为0，1才会intern，而我们输入python，为啥会被intern？**

```c
>>> a="python"
>>> b="python"
>>> a is b
True
>>> a="pytho!"
>>> b="pytho!"
False
```

A：因为代码执行时从**PyCode_New**开始，其参数中consts 包含了编译器定义的Literals：boolean，浮点数，整数以及字符串。其中字符串会经过intern_string_constants中的all_name_chars过滤后的字符串才会。

```c
[codeobject.c]
PyCodeObject *
PyCode_New(..., PyObject *consts, ...)
{
    intern_string_constants(consts);
}

#define NAME_CHARS \
	"0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ_abcdefghijklmnopqrstuvwxyz"

/* all_name_chars(s): true iff all chars in s are valid NAME_CHARS */
static int
all_name_chars(PyObject *o)
{
    static char ok_name_char[256];
    static const unsigned char *name_chars = (unsigned char *) NAME_CHARS;
    const unsigned char *s, *e;
    
    if (ok_name_char[*name_chars] == 0) {
        const unsigned char *p;
        for (p == name_chars; *p; p++)
            ok_name_char[*p] = 1;
    }
    s = (unsigned char *)PyString_AS_STRING(o);
    e = s + PyString_GET_SIZE(o);
    while (s != e) {
        if (ok_name_char[*s++] == 0)
            return 0;
    }
    return 1;
}
```

strings are interned at compile time

the tuple `consts` contains the literals defined at compile time: booleans, floating-point numbers, integers, and strings declared in your program. 

Q2: 

```c
>>> s1 = 'foo'
>>> s2 = ''.join(['f', 'o', 'o'])
>>> s1 is s2
False
>>> a=intern("pytho!")
>>> b=intern("pytho!")
True
```

A:

s1是在编译期，s2是在运行期。intern机制是发生再编译期，因此s1会被intern，s2不会。强行intern的话是可以的。

#### 3.4 字符缓冲池

`PyStringObject` 对象池设计了对象(缓冲)池**characters**，在Python初始化被创建，初始其中所有的PyStringObject都为NULL。

`static PyStringObject *characters[UCHAR_MAX+1]`

- 创建`PyStringObject`对象<string P>;
- 对对象<string p> 进行intern操作
- 将对象<string P> 缓存至字符缓冲池中

#### 3.5 `PyStringObject` 效率相关问题

**由于PyStringObject对象是一个不可变对象，意味着字符串连接必须创建一个新的。**

“+” 操作符对字符串进行连接时，会调用string_concat函数，N个`PyStringObject`, 就会又N次内存申请

```c
static PyObject *
string_concat(register PyStringObject *a, register PyObject *bb)
{
    register PyStringObject *op;
    ...
#define b ((PyStringObject *)bb)
    size = Py_SIZE(a) + Py_SIZE(b);
    op = (PyStringObject *)PyObject_MALLOC(PyStringObject_SIZE + size);
    if (op == NULL)
        return PyErr_NoMemory();
    (void)PyObject_INIT_VAR(op, &PyString_Type, size);
    op->ob_shash = -1;
    op->ob_sstate = SSTATE_NOT_INTERNED;
    //把a b字符copy到新创建的PyStringObject中
    Py_MEMCPY(op->ob_sval, a->ob_sval, Py_SIZE(a));
    Py_MEMCPY(op->ob_sval + Py_SIZE(a), b->ob_sval, Py_SIZE(b));
    op->ob_sval[size] = '\0';
    return (PyObject *) op;
}
```

"".join(list)，会调用string_join,  N个PyStringObject也只申请一次内存，会计算list中所有字符串的总长度，然后申请一次内存。

```c
[stringobject.c]
static PyObject* string_join(PyStringObject *self, PyObject *orig)
{
    /*"abc".join(list), self is "abs"*/
    char *sep = PyString_AS_STRING(self);
    const int seplen = PyStrint_GET_SIZE(self);
    ...
    /*创建长度为sz的PyStringObject对象*/
    res = PyString_FromStringAndSize((char*)NULL, (int)sz);
    /*将list中的字符串copy到新创建的PyStringObject中*/
    p = PyString_AS_STRING(res);
    for (i = 0; i < seqlen; ++i)
    {
        size_t n;
        item = PySequence_Fast_ITEM(seq, i);
        n = PyString_GET_SIZE(item);
        memcpy(p, PyString_AS_STRING(item), n);
        p += n;
        if (i < seqlen - 1)
        {
            memcpy(p , sep, seplen);
            p += seplen;
        }
    }
    return res;
}
```

#### 3.6 Hack PyStringObject

在string_length中打印characters缓存池中的信息，在观察intern机制





















































