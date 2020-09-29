#### 12.1 Python中的对象模型

- is-kind-of: 基类与子类 (object :  A)
- is-instance-of: 实例与类(a: A)

type： 为metaclass, 可以成为其它class对象的type。在Python内部对应PyType_Type

object: 任何class对象都必须直接或间接继承自object, 可视为万物之母。在Python内部对应PyBaseObject_Type



class A 即是class对象（可以通过实例化的动作创建新的instance），又是instance对象(通过metaclass对象实例化得到)；

#### 12.2 从type对象到class对象

可调用性：对象对应的类实现了`__call__`，那么这个对象就是可调用的对象。

在python中，所谓调用就是执行对象的type对应的class对象的`tp_call`操作在。通过PyObject_Call操作

一个对象能否被调用在运行时才能确定(PyObject_CallFunctionObjArgs)。