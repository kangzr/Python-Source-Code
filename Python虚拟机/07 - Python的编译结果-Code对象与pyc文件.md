### 7.1 Python程序执行过程

Python与Java, C#执行原理都可用**虚拟机+字节码**来描述。

Python: test.py --(1. **python27.dll 编译器 编译成成字节码**)---> test.pyc--(2. python27.dll **虚拟机按顺序逐条执行字节码**)-->a.out  (Python的**编译器和虚拟机**都藏身在python27.dll)

Python虚拟机与Java或.NET相比，是一种抽象层次更高的虚拟机。

### 7.2  Python编译器的编译结果--PyCodeObject对象

