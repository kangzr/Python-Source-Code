#### 7.1 Python程序执行过程

Python与Java, C#执行原理都可用**虚拟机+字节码**来描述。

Python: test.py --(1. **python27.dll 编译器 编译成成字节码**)---> test.pyc--(2. python27.dll **虚拟机按顺序逐条执行字节码**)-->a.out  (Python的**编译器和虚拟机**都藏身在python27.dll)

```mermaid
graph LR
A[test.py] --编译器编译dll--> B[test.pyc字节码]--虚拟机执行dll-->C[执行结果]
```



Python虚拟机与Java或.NET相比，是一种抽象层次更高的虚拟机。

#### 7.2  Python编译器的编译结果--PyCodeObject对象

##### 7.2.1 PyCodeObject对象与pyc文件

pyc文件中是一个`PyCodeObject`对象，`PyCodeObject`才是真正的编译结果，pyc只是这个对象在磁盘上的表现形式。

即pyc(二进制文件)和PyCodeObject是python对编译结果的两种不同存在方式。

程序运行期间，编译结果存在于内存的PyCodeObject对象中，运行结束后，编译结果又被保存到了pyc文件中。

**下次运行相同的程序时，Python会根据pyc文件中记录的编译结果直接建立内存中的PyCodeObject对象，而不再对源文件进行编译。**

##### 7.2.2 Python源码中的PyCodeObject

```c
[code.h]
typedef struct{
	PyObject_HEAD
	int co_argcount; // arguments(位置参数的个数), except *args,
	int co_nlocals;  // local variables，局部变量个数
	int co_stacksize;  // entries needed for evaluation stack 执行该段code block需要的栈空间
	int co_flags;  // 
	PyObject *co_code; //instruction opcodes Code block编译所得的字节码指令序列，以PyStringObject形式存在
	PyObject *co_consts; // list (constants used)PyTupleObject对象，保存Code Block中的所有变量
	PyObject *co_names;  // list of strings(names used)PyTupleObject对象，保存CodeBlock中的所有符号
	PyObject *co_varnames;  //tuple of strings(local variable names) Code block中局部变量名集合
	PyObject *co_freevars; //tuple of strings(free variable names), 实现闭包需要用到
	PyObject *co_cellvars;  //tuple of strings(cell variable names)内嵌函数所引用局部变量集合
	PyObject *co_filename;  // Code Block所对应的py文件的完整路径
	PyObject *co_name;  // Code Block名字，通常函数名或类名
    int co_firstlineno;  // first source line number
	PyObject *co_inotab;  //string encoding addr <-> lineno mapping
}
```

一个Code Block，创建一个PyCodeObject（对应namespace，或者作用域）。以下会产生3个Code Block.整个文件也会对应一个

```python
# demo.py
# Code Block <-->  NameSpace
class A(object):
	pass

def fun():
	pass
a = A()
fun()
```

##### 7.2.3 pyc文件

并非所有的python文件运行都会生成pyc文件，import机制会触发pyc文件的生成，import机制：会现在path中搜索相应pyc或者dll文件，如果没有文件，只发现了相应的py文件，就会把py文件编译成PyCodeObject的中间结果，然后将中间结果写入pyc文件，再对。

```flow
st=>start: import abc
op1=>operation: 编译abc.py
cond=>condition: 寻找pyc或者dll文件?
op=>operation: ERROR
e=>end: import(将pyc中PyCodeObject重新从内存中复制出来)

st->cond
cond(yes)->e
cond(no)->op1
op1->e

```



##### 7.2.4 在Python中访问PyCodeObject对象

Python中有与C一级的PyCodeObject对象对应的对象(code对象)。通过code对象，可访问PyCodeObject对象中的各个域。

```python
>>> source = open("demo.py").read()
>>> co = compile(source, 'demo.py', 'exec')
>>> type(co)
<type 'code'>
>>> dir(co)
>>> print co.co_filename
```

#### 7.3 Pyc文件的生成

##### 7.3.1 创建pyc文件的具体过程

一个pyc文件包含三部分独立信息：python的magic number, pyc文件创建的时间信息，pycodeobject对象

```c
[import.c]

static void
write_compiled_module(PyCodeObject *co, char *cpathname, struct stat *srcstat, time_t mtime){
	...
    // 排他性打开文件
    fp = open_exclusive(cpathname, mode);
    // 写入Python的magic number
    PyMarshal_WriteLongToFile(pyc_magic, fp, Py_MARSHAL_VERSION);
    // 写入时间信息
    PyMarshal_WriteLongToFile(0L, fp, Py_MARSHAL_VERSION);
    // 写入PyCodeObject对象
    // co就是编译出来的PyCodeObject对象
    PyMarshal_WriteObjectToFile((PyObject *)co, fp, Py_MARSHAL_VERSION);
    ...
}
```

**magic number**: 在加载pyc文件时首先检查magic number，如果发现python自身的magic number与待加载的pyc文件中记录的magic number不同，则拒绝加载

**pyc文件创建时间**：加载过程中，如果pyc文件的时间早于py文件的时间，就会重新编译py文件生成pyc。

python必须要将对象的类型信息也写入pyc；所有的对象都会归结为两种形式：1. 对数值的写入；2. 对字符串的写入；

```c
[import.c]
#define TYPE_INT 'i'
#define TYPE_LIST '['

// w_object in marshal.c
static void
w_object(PyObject* v, WFILE *p)
{
    ...
    else if(PyInt_CheckExact(v)) {
        w_byte(TYPE_INT, p);
        w_long(x, p);
    }
    else if(PyList_CheckExact(v)) {
        w_byte(TYPE_LIST, p);
        n = PyList_GET_SIZE(v);
        W_SIZE(n, p);
        for (i = 0; i < n; i++)
            w_object(PyList_GET_ITEM(v, i), p);
    }
	w_byte(TYPE_LIST, p);
}

```

##### 7.3.2 向pyc文件写入字符串

```c
// marshal.c
typedef struct {
    FILE *fp;
    PyObject *strings // dict on marshal, list on unmarshal
}WFILE;
```

向pyc写入时，strings为一个dict对象，从pyc读出，strings时一个list对象

```c
// marshal.c  写入
void PyMarshal_WriteObjectToFile(PyObject *x, FILE *fp, int version)
{
    WFILE wf;
    wf.strings = (version > 0) ? PyDict_New() : NULL;
    w_object(x, &wf);
}
```

```c
// marshal.c 读出
PyObject *PyMarshal_ReadObjectFromFile(FILE *fp)
{
	RFIEL rf;
    rf.strings = PyList_new(0);
    result = r_object(&rf);
    return result;
}
```

##### 7.3.3 一个PyCodeObject, 多个PyCodeObject

PyCodeObject有嵌套的关系(根据作用域关系进行嵌套，因此文件中的func对应的object是包含在文件对应的object中的)，在写入时，也会递归写入；

这种嵌套的关系意味着，pyc文件中的二进制数据实际是一种有结构的数据，这种结构化性质预示着我们能够以XML的形式来讲pyc文件尽心给可视化。

#### 7.4 Python的字节码

```c
// opcode.h
#define STOP_CODE 	0
#define POPTOP 		1
#define ROT_TOW		2
...
#define CALL_FUNCTION_KW 	141

```

#### 7.5 解析pyc文件

```python
>>> s = open("test.py").read()
>>> co = compile(s, 'test.py', 'exec')
>>> import dis
>>> dis.dis(co)
```























