---
layout: post
title:  "对于 Python 3 中的 threading 模块阅读笔记"
date:   2019-12-15 12:18:29 +0800
categories: python,colorama
---
&emsp;&emsp;这个想法起初只是为了找到 python 3 下的 **threading.Thread** 中类似于线程关闭的方法，可惜似乎没有直接的实现。在 python 2 下 Thread 实例的退出可以调用 **Thread._Thread__stop()** 方法来完成，但是这个内置方法在 python 3 下已经没有，通过代码跟踪发现 `threading.py` 并不是实现 python 线程的基础文件，真正的启动操作应该是调用 cpython 暴露出来的 **start_new_thread** 方法来完成启动，在启动的同时传入了 **self._bootstrap()** 做一些初始化操作：  
```python
def start(self):
	"""Start the thread's activity.

	It must be called at most once per thread object. It arranges for the
	object's run() method to be invoked in a separate thread of control.

	This method will raise a RuntimeError if called more than once on the
	same thread object.

	"""
	if not self._initialized:
	    raise RuntimeError("thread.__init__() not called")

	if self._started.is_set():
	    raise RuntimeError("threads can only be started once")
	with _active_limbo_lock:
	    _limbo[self] = self
	try:
	    _start_new_thread(self._bootstrap, ())
	except Exception:
	    with _active_limbo_lock:
		del _limbo[self]
	    raise
	self._started.wait()
```
&emsp;&emsp;**_start_new_thread** 的真正实现应该在 cpython 源码下的 `Modules/_threadmodule.c` 中：
```c
static PyObject *
thread_PyThread_start_new_thread(PyObject *self, PyObject *fargs)
{
    PyObject *func, *args, *keyw = NULL;
    struct bootstate *boot;
    unsigned long ident;

    if (!PyArg_UnpackTuple(fargs, "start_new_thread", 2, 3,
                           &func, &args, &keyw))
        return NULL;
    if (!PyCallable_Check(func)) {
        PyErr_SetString(PyExc_TypeError,
                        "first arg must be callable");
        return NULL;
    }
    if (!PyTuple_Check(args)) {
        PyErr_SetString(PyExc_TypeError,
                        "2nd arg must be a tuple");
        return NULL;
    }
    if (keyw != NULL && !PyDict_Check(keyw)) {
        PyErr_SetString(PyExc_TypeError,
                        "optional 3rd arg must be a dictionary");
        return NULL;
    }
    boot = PyMem_NEW(struct bootstate, 1);
    if (boot == NULL)
        return PyErr_NoMemory();
    boot->interp = _PyInterpreterState_Get();
    boot->func = func;
    boot->args = args;
    boot->keyw = keyw;
    boot->tstate = _PyThreadState_Prealloc(boot->interp);
    if (boot->tstate == NULL) {
        PyMem_DEL(boot);
        return PyErr_NoMemory();
    }
    Py_INCREF(func);
    Py_INCREF(args);
    Py_XINCREF(keyw);
    PyEval_InitThreads(); /* Start the interpreter's thread-awareness */
    ident = PyThread_start_new_thread(t_bootstrap, (void*) boot);
    if (ident == PYTHREAD_INVALID_THREAD_ID) {
        PyErr_SetString(ThreadError, "can't start new thread");
        Py_DECREF(func);
        Py_DECREF(args);
        Py_XDECREF(keyw);
        PyThreadState_Clear(boot->tstate);
        PyMem_DEL(boot);
        return NULL;
    }
    return PyLong_FromUnsignedLong(ident);
}
```
&emsp;&emsp;这里可以看到，这个方法使用 c 语言实现的一个标准的 python 接口方法，最终调用了 **PyThread_start_new_thread** 方法，而这个方法，在 `Python/thread_pthread.h` 中实现：  
```c
unsigned long
PyThread_start_new_thread(void (*func)(void *), void *arg)
{
    pthread_t th;
    int status;
#if defined(THREAD_STACK_SIZE) || defined(PTHREAD_SYSTEM_SCHED_SUPPORTED)
    pthread_attr_t attrs;
#endif
#if defined(THREAD_STACK_SIZE)
    size_t      tss;
#endif

    dprintf(("PyThread_start_new_thread called\n"));
    if (!initialized)
        PyThread_init_thread();

#if defined(THREAD_STACK_SIZE) || defined(PTHREAD_SYSTEM_SCHED_SUPPORTED)
    if (pthread_attr_init(&attrs) != 0)
        return PYTHREAD_INVALID_THREAD_ID;
#endif
#if defined(THREAD_STACK_SIZE)
    PyThreadState *tstate = _PyThreadState_GET();
    size_t stacksize = tstate ? tstate->interp->pythread_stacksize : 0;
    tss = (stacksize != 0) ? stacksize : THREAD_STACK_SIZE;
    if (tss != 0) {
        if (pthread_attr_setstacksize(&attrs, tss) != 0) {
            pthread_attr_destroy(&attrs);
            return PYTHREAD_INVALID_THREAD_ID;
        }
    }
#endif
#if defined(PTHREAD_SYSTEM_SCHED_SUPPORTED)
    pthread_attr_setscope(&attrs, PTHREAD_SCOPE_SYSTEM);
#endif

    pythread_callback *callback = PyMem_RawMalloc(sizeof(pythread_callback));

    if (callback == NULL) {
      return PYTHREAD_INVALID_THREAD_ID;
    }

    callback->func = func;
    callback->arg = arg;

    status = pthread_create(&th,
#if defined(THREAD_STACK_SIZE) || defined(PTHREAD_SYSTEM_SCHED_SUPPORTED)
                             &attrs,
#else
                             (pthread_attr_t*)NULL,
#endif
                             pythread_wrapper, callback);

#if defined(THREAD_STACK_SIZE) || defined(PTHREAD_SYSTEM_SCHED_SUPPORTED)
    pthread_attr_destroy(&attrs);
#endif

    if (status != 0) {
        PyMem_RawFree(callback);
        return PYTHREAD_INVALID_THREAD_ID;
    }

    pthread_detach(th);

#if SIZEOF_PTHREAD_T <= SIZEOF_LONG
    return (unsigned long) th;
#else
    return (unsigned long) *(unsigned long *) &th;
#endif
}
```
&emsp;&emsp;由此可知，python 的线程创建最终也落到 **pthread_create** 方法上，那么如果要关闭这个线程，是不是就直接使用 **pthread_kill** 就可以了？