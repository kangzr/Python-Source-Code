本章主要通过一些简单的python表达式，了解运行时栈和local名字空间的流程，虚拟机的运行机理

#### 9.1 简单内建对象的创建

理解运行时栈和local名字空间的工作流程。

```python
i = 1

0 LOAD_CONST 0 (1)  // 将1入栈
3 STORE_NAME 0  (i)  // 在local名字空间建立i 和 1的关系，并将1从运行时栈中pop出来

d = {}
12 BUILD_MAP  0  // 建了新的PyDict-Object, 并入栈
15 STORE_NAME 2(d)  // 建立d 和 PyDict-Object的映射关系，并从运行时栈pop出来

l = []
18 BUILD_LIST 0
21 STORE_NAME 3(1)
24 LOAD_CONST 2(none)
27 RETURN_VALUE
```

#### 9.2 复杂内建对象的创建

#### 9.3 其它一般表达式

