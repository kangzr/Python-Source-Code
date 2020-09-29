#### 10.1 Python虚拟机中的if控制流

```c
[traceback.c]
int PyTraceBack_Here(PyFrameObject *frame){
	// 获取线程状态对象
	PyThreadState *tstate = frame->f_tstate;
	// 保存线程状态对象中现在维护的traceback对象
	PyTracebackObject *oldtb = (PyTracebackObject *)tstate->curexc_traceback;
	// 创建新的traceback对象
	PyTracebackObject *tb = newtracebackobject(oldtb, frame);
	//将新的traceback对象交给线程状态对象
	tstate->curexc_traceback = (PyObject *)tb;
	Py_XDECREF(oldtb);
	return 0;
}



[traceback.h]
typedef struct _traceback{
	PyObject_HEAD
	struct _traceback *tb_next;
	struct _frame *tb_frame;
	int tb_lasti;
	int tb_lineno;
}PyTracebackObject;
```



