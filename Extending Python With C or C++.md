### How to Extending Pytyhon With C

Step 1:  Write Your own c file

```c
// "Python.h" defines a set of functions, macros and variables that provide access to most aspects of the Python run-time system

#include <Python.h>
static PyObject *
spam_system(PyObject *self, PyObject *args)
{
    const char *command;
    int sts;

    if (!PyArg_ParseTuple(args, "s", &command))
        return NULL;
    sts = system(command);
    return Py_BuildValue("i", sts);
}

// The Module's Method Table and Intialization Function
// the method in SpamMethods can be access in python 
// METH_O/METH_VARARGS/METH_NOARGS
// struct PyMethodDef{char *ml_name; PyCFunction ml_meth; int ml_flags; char *ml_doc;}
// 其中ml_name暴露给python程序的函数名， ml_meth函数指针, ml_flags传参方式, ml_doc文档
static PyMethodDef SpamMethods[] = {
    ...
    {"system",  spam_system, METH_VARARGS,
     "Execute a shell command."},
    ...
    {NULL, NULL, 0, NULL}        /* Sentinel */
};

// When the Python Program imports module spam for the first time, initspam() is called.
// and module 'spam' will inserted in sys.modules
PyMODINIT_FUNC
initspam(void)
{
    (void) Py_InitModule("spam", SpamMethods);
}

```

Step 2: compile c file to so file

```
gcc -I/usr/include/python2.7 -lpython2.7 -g fPIC -c spam.c -o spam.o \
&& gcc -shared spam.o -o spam.so
```





**Py_BulidValue**: 将C变量构建为Python的PyObject* **PyArg_ParseTuple**反之