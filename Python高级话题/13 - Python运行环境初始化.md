#### 13.1  线程环境初始化

`Py_InitializeEx`： 创建进程，创建线程，初始化Frame: `_PyFrame_Init`

##### 初始化线程环境

在`Py_InitializeEx`开始处，会先调用`PyInterpreterState_New`创建一个崭新的`PyInterpreterState`对象

```c
// pystate.c

static PyInterpreterState *interp_head = NULL;

PyInterpreterState* PyInterpreterState_New(void)
{
    PyInterpreterState *interp = malloc(sizeof(PyInterpreterState));
    if (interp != NULL) {
        ...
        interp_head = interp;  // 全局管理PyInterpreterState对象链表
    }
    return interp;
}
```

创建线程状态

```c
//pystate.c
PyThreadState* PyThreadState_New(PyInterpreterState *interp)
{
    PyThreadState *tstate = (PyThreadState *)malloc(sizeof(PyThreadState));
    // 设置获得线程中函数调用栈的操作
    if (_PyThreadState_GetFrame == NULL)
        _PyThreadState_GetFrame = threadstate_getframe;
    if (tstate != NULL) {
        // 在PyThreadState对象中关联PyInterpreterState对象
        tstate->interp = interp;
        ...
        tstate->thread_id = PyThread_get_thread_ident();  // python线程id
    }
}
```

设置全局变量`builtin_object`，

```c
// frameobject.c
static PyObject *builtin_object;

int _PyFrame_Init()
{
    builtin_object = PyString_InternFromString("__builtins__");
    return (builtin_object != NULL);
}
```

#### 13.2 系统module初始化

第一个创建的moudle就是`__builtin__`module

```c
// pythonrun.c
void Py_InitializeEx(int install_sigs)
{
    ...
    interp->modules = PyDict_New();  // 维护系统所有的module
    bimod = _PyBuiltin_Init();
    ...
}

// PyInterpreter对象中维护这所有的PyThreadState对象共享资源
// bltinmodule.c
PyObject*  _PyBuiltin_Init(void)
{
    PyObject *mod, *dict, *debug;
    // 创建并设置__builtin__ module
    mod = Py_InitModule4();
    // 将所有python内建对象加入__builtin__ module中
    dict = PyModule_GetDict(mod);
}
```





















