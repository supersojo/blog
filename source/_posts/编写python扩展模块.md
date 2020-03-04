---
title: 编写python扩展模块
date: 2019-05-01 21:18:10
tags:
- python
categories:
- python
---
一直以来更多的是使用c语言，看到python语言在云计算时代广泛运用，
想着如何用c编写模块供python调用。python的官方实现就是c语言实现的，
即cpython。python提供了运行时接口供我们编写c扩展模块。

linux发行版通常把头文件和库文件打包成xxxx-dev的包形式。我们需要
安装python-dev来编写python扩展。

```
 #include <python.h> /* Python API to python runtime system */
 static PyObject *SpamError;
 /*
 spam.system(string)
 */
 static PyObject * spam_system(PyObject *self, PyObject *args)
 {
         const char *command;
         int sts;
         if (!PyArg_ParseTuple(args,"s",&command))
                 return NULL;
         /*
          run the command in another process until it runs done     
         */
         sts = system(command);     
         if (sts < 0) {                                         
			PyErr_SetString(SpamError,"System command failed");     
         }                
         return PyLong_FromLong(sts);
 }
 static PyMethodDef SpamMethods[] = {
         {"system",spam_system,METH_VARARGS,"Execute a shell command."},
         {NULL,NULL,0,NULL}
 };
 PyMODINIT_FUNC
 initspam(void)
 {
         PyObject *m;
         m = Py_InitModule("spam",SpamMethods);
         if (!m)
                 return ;
         SpamError = PyErr_NewException("spam.error",NULL,NULL);
         Py_INCREF(SpamError);
         PyModule_AddObject(m,"error",SpamError);
 }
```

如果模块名为xxx则必须提供xxxinit的函数。在python解释器加载模块的时候会寻找
模块动态库文件中导出的xxxinit函数。
