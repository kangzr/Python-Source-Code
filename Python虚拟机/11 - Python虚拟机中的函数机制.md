#### 11.1 PyFunctionObject对象

```c
// funcobject.h
typdef struct {
    PyObject_HEAD
    PyObject *func_code;		// 对应函数编译后的PyCodeObject对象
    PyObject *func_globals;  	// 函数运行时的global名字空间
    PyObject *func_defaults;    // 默认参数(tuple or NULL)
    PyObject *func_closure;		// null or tuple of cell objects 用于实现从闭包
    PyObject *func_doc;			// 函数的文档PyStringObject
    PyObject *func_name;		// 函数名词 __name__属性
    PyObject *func_dict; 		// __dict__属性
    PyObject *func_weakreflist;
    PyObject *func_modules;		// __module__
}PyFuncObject;
```

PyCodeObject静态表示，且一个可能对应多个PyFunctionObject

PyFunctionObject为运行时动态产生，即在执行def语句时创建，也包含函数的静态信息，存储在`func_code`中，同时包含执行时必需的动态信息，即上下文信息。

#### 11.2 无参函数调用

#### 11.4 函数参数的实现

理论上，python中的函数只能有256个位置参数和256个键参数。