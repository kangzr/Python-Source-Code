##### Python虚拟机中的执行环境

##### 8.1.1 Python源码中的PyFrameObject

Python真正执行时面对的不是PyCodeObject，而是PyFrameObject，它就是执行环境。多个PyFrameObject对象会链接起来，形成一条执行环境链表。

一个PyFrameObject维护一个PyCodeObject(一个Code block)。出栈时，由f_back字段取得上一个frame的信息。

```c
// frameobject.h
typedef struct _frame {
    PyObject_VAR_HEAD
     struc _frame *f_back;  // 执行环境链上的前一个frame
    PyCodeObject *f_code;  	// PyCodeObject 对象
    PyObject *f_builtins;	// builtin 名字空间
    PyObject *f_locals;		// local名字空间
    PyObject **f_valuestack;	// 运行时栈的栈底位置
    PyObject **f_stacktop;		// 运行时栈的栈顶位置
    ...
    int f_lasti;				// 上一条字节码指令在f_code中的偏移位置
    int f_lineno;				// 当前字节码对应的源码代码行
    ...
    // 动态内存，维护(局部变量+cell对象集合+free对象集合+运行时栈) 所需空间
    PyObject *f_localsplus[1];
}PyFramObject;
```

##### 8.1.2 PyFrameObject中的动态内存空间

```c
// frameobject.c
PyFrameObject *
PyFrame_New(PyThreadState *tstate, PyCodeObject *code, PyObject *globals, PyObject *locals)
{
	...
    // 四部分构成了PyFrameObject维护的动态内存区，其大小由extras确定
    extras = code->co_stacksize + code->co_nlocals + ncells + nfrees;
    f->f_valuestack = f->f_localsplus + extras;  // 栈底
    f->f_stacktop = f->f_valuestack; // 栈顶
}
```

额外申请的内存空间一部分给PyCode-Object对象中存储的哪些局部变量的、co_freevars, co_cellvars,另一部分给运行时栈使用。

f_localsplus = extras + stack

##### 8.1.3 在Python中访问PyFrameObject对象

frame object是对c一级的PyFrameObject的包装，通过sys module中的_getframe可以获取frame object

```python
import sys
sys._getframe()
```

#### 8.2 名字、作用域和名字空间

PyFrameObject有3个独立的命名空间：local名字空间，global名字空间，builtin名字空间。

##### 8.2.1 Python程序的基础结构  ---- module

一个文件对应一个module；

加载module的两种方式：1. import进行动态加载 2. python main.py

##### 8.2.2 约束与名字空间

一个对象的namespace中的所有名字都称为该对象的属性。

##### 8.2.3 作用域与命名空间 LGB规则 LEGB规则

LGB：local --> global --> builtin

LEGB：local --> enclosing(直接外围作用域) --> global -->builtin

```python
// 闭包: 实现最内嵌作用域规则
a = 1
def f():
    a = 2
    def g():
        print a
    return g
```

根据LEGB规则，f中的a可以在f作用域中找到，所以不会往外去找，但是f作用域中对a的使用在赋值之前，因此报错。

```python
a = 1

def g():
    print a  // 1

def f():
    print a // error:  local variable 'a' referenced before assignment
    a = 2  // 会隐藏global命名空间中的a

g()
f()
```

从字节码层面分析上述例子

```
a = 1
def g():
	print a
	#0 LOAD_GLOBAL    0 // 在global命名空间查找
	#3 PRINT_ITEM
	#4 PRINT_NEWLINE

def f():
	print a
	#0 LOAD_FAST    0  // local命名空间查找
	#3 PRINT_ITEM
	#4 PRINT_NEWLINE 
```

通过global，可以强制命令python对某个名字的引用直接使用global命名空间。

#### 8.3 Python虚拟机的运行框架

运行环境：全局概念；

执行环境：一个栈帧，对应一个Code Block对应的概念;

**PyEval_EvalFramEx**  ==  Python的虚拟机具体实现。

Python虚拟机**从头到尾**遍历co_code(PyStringObject对象)来执行字节码指令序列。依托三个变量完成: `first_instr` `next_instr` `f_lasti`, 其框架是一个for循环加上一个巨大的switch/case结构。

```c
// ceval.c  PyEval_EvalFramEx
PyObject* PyEval_EvalFrameEx(PyFrameObject *f, int thowflag)
{
    ...
    // 获取当前活动线程对应的线程状态对象
    PyThreadState *tstate = PyThreadState_GET();
    ...
    // 设置线程状态对象中的frame
    tstate->frame = f;
    co = f->f_code;
    names = co->co_names;
    consts = co->co_consts;
    why = WHY_NOT;
    ...
    // 虚拟机主循环
    for(;;){
    ...
    fast_next_opcode:
        f->f_lasti = INSTR_OFFSET();
        // 获取字节码
        opcode = NEXTOP();
        oparg = 0;
        // 如果指令需要参数，获得指令参数
        if (HAS_ARG(op_code))
            oparg = NEXTARG();
dispatch_opcode:
    // 指令分派
	switch (opcode){
	case:
    ...
	}
}
}


// Python结束字节码执行状态
enum why_code {
    WHY_NOT = 0x001, // no error
    ...
}
```

#### 8.4 Python运行时环境初探

**PyInterpreterObject：对进程的抽象，**

**PyThreadState： 线程状态信息的抽象**

**Python中通过一个全局解释器GIL(Global Interpreter Lock)来是实现线程同步的，**

Python虚拟机可看成是对CPU的抽象

```c
//pystate.h
typdef struct _is {
    struct _ts *tstate_head;  // 模拟进程环境中的线程集合
}PyInterpreterState;

typedef struct _ts {
    struct _frame *frame;  // 模拟线程中的函数调用堆栈
} PyThreadState;
```

Python虚拟机开始执行时，会将当前线程状态对象中的frame设置为当前的执行环境(frame);



在创建新的PyFrameObject对象时，则从当前线程的状态对象中取出旧的frame，建立PyFrameObject链表

```c
// frameobject.c
PyFrameObject *
PyFrame_New(PyThreadState *tstate, PyCodeObject *code, PyObject *globals, PyObject *locals)
{
    // 从PyThreadState中获得当前线程的当前执行环境
    PyFrameObject *back = tstate->frame;
    PyFrameObject *f;
    ...
    // 创建新的执行环境
    f = PyObject_GC_Resize(PyFrameObject, f, extras);
    // 链接当前执行环境
    f->f_back = back;
    f->f_tstate = tstate;
}
```

























