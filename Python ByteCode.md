#### Python 字节码

STORE_FAST/LOAD_FAST: 存储/读取PyFrameObject的f_localsplus

STORE_NAME/LOAD_NAME: 存储/读取(以此搜索local，global，builtin)local命名空间

模块的栈帧对象中的`f_locals` 和`f_globals` 值一样，都是`sys.modules['__main__'].__dict__` , 因此python

中函数定义顺序无关，函数声明和实现其实是分离的，声明的字节码指令在模块的PyCodeObject中执行，而实现的字节码指令则是在函数自己的PyCodeObject中 

