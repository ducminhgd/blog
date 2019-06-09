# Python integer and memory

## Memory allocation

```python
>>> id(5)  
11381600
>>> x = 5  
>>> id(x)  
11381600
>>> y = 5  
>>> id(y)  
11381600
```

As you can see that `x`, `y` and `5` has the same memory address as `5`. First, I was so currious about this and try to test other number

```python
>>> y = 7
>>> id(7)
11381664
>>> id(y)
11381664
```

The result is the same, then I had an idea

> In python, everything is objects, is it possible that 5 and 7 are objects allocated in memory and `x` and `y` _point_ to them?

Wait! There is something more, something....

```python
>>> 11381664 - 11381600
64
>>> id(7) - id(5)
64
>>> 64 / 2
32.0
```

So, `7` and `5` are allocatted near to each others in memory space. Try a little more.

```python
>>> import ctypes
>>> ctypes.cast(11381600, ctypes.py_object).value
5
>>> ctypes.cast(11381600 + 32, ctypes.py_object).value
6
>>> ctypes.cast(11381600 + 64, ctypes.py_object).value
7
```

Oh... they are allocated like an array of memory space, a sequent allocation

## PyObject

Taking a look at [PyObject Integer Object](https://docs.python.org/3/c-api/long.html#c.PyLong_FromLong)

> The current implementation keeps an array of integer objects for all integers between -5 and 256, when you create an int in that range you actually just get back a reference to the existing object. So it should be possible to change the value of 1. I suspect the behaviour of Python in this case is undefined. :-)

Let make a test

```python
>>> id(260)
140219535811728
>>> x = 260
>>> id(x)
140219536828848
```

But, with Python Interpreter, two assignations occur in the same line both variables will refer to the same object to avoid wasting space.

```python
>>> x = 360; y = 360
>>> id(x)
140219535811696
>>> id(y)
140219535811696
```
