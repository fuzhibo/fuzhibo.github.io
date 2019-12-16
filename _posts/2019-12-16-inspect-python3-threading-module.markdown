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

```
&emsp;&emsp;这里可以看到