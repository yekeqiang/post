# Python -- 让你的代码运行的更快的技巧

标签（空格分隔）： Python 优化 性能

---


> 注：原文地址 [Python: faster way][1]


> 注：个人学习记录用。建议大家看原文，原文对比更加清晰，一目了然。

> 注：各位要使用这些技巧的话，请在自己的服务器上测试一遍，并且加大测试的数值，目前的用例都是 10 W 次，我们可以测试 100 W , 1000 W 次。。。。
> 注：如果测试的性能相差不多，请以可读性为主。

## Plataform

运行测试的平台环境：

```
>>> import sys
>>> import platform
>>> platform.system()
'Linux'
>>> platform.release()
'3.11.0-19-generic'
>>> sys.version
'2.7.5+ (default, Feb 27 2014, 19:37:08) 
[GCC 4.8.1]'
>>> from timeit import timeit
>>> from dis import dis
>>> 
```

以下的代码主要是使用了 timeit 函数测试处理时间，以及使用 dis 函数显示详细的处理步骤（汇编的方式），能让你清楚的知道，慢在哪里？为什么慢？

## 测试用例 1

更快的方式：
```
def a():
    d = {}
    return d

>>> 
>>> timeit(a, number=1000000)
... 0.0905051231384
>>> 
>>> dis(a)
  5           0 BUILD_MAP                0
              3 STORE_FAST               0 (d)

  6           6 LOAD_FAST                0 (d)
              9 RETURN_VALUE        

>>> 
```

更慢的方式：

```
def a():
    d = dict()
    return d

>>> 
>>> timeit(a, number=1000000)
... 0.206549167633
>>> 
>>> dis(a)
  5           0 LOAD_GLOBAL              0 (dict)
              3 CALL_FUNCTION            0
              6 STORE_FAST               0 (d)

  6           9 LOAD_FAST                0 (d)
             12 RETURN_VALUE        

>>> 
      
```

## 测试用例 2

更快的方式：

```
def a():
    l = [0, 8, 6, 4, 2, 1, 3, 5, 7, 9]
    l.sort()
    return l

>>> 
>>> timeit(a, number=1000000)
... 0.53688287735
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (0)
              3 LOAD_CONST               2 (8)
              6 LOAD_CONST               3 (6)
              9 LOAD_CONST               4 (4)
             12 LOAD_CONST               5 (2)
             15 LOAD_CONST               6 (1)
             18 LOAD_CONST               7 (3)
             21 LOAD_CONST               8 (5)
             24 LOAD_CONST               9 (7)
             27 LOAD_CONST              10 (9)
             30 BUILD_LIST              10
             33 STORE_FAST               0 (l)

  6          36 LOAD_FAST                0 (l)
             39 LOAD_ATTR                0 (sort)
             42 CALL_FUNCTION            0
             45 POP_TOP             

  7          46 LOAD_FAST                0 (l)
             49 RETURN_VALUE        

>>> 
```

更慢的方式：

```
def a():
    l = [0, 8, 6, 4, 2, 1, 3, 5, 7, 9]
    return sorted(l)

>>> 
>>> timeit(a, number=1000000)
... 0.781757831573
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (0)
              3 LOAD_CONST               2 (8)
              6 LOAD_CONST               3 (6)
              9 LOAD_CONST               4 (4)
             12 LOAD_CONST               5 (2)
             15 LOAD_CONST               6 (1)
             18 LOAD_CONST               7 (3)
             21 LOAD_CONST               8 (5)
             24 LOAD_CONST               9 (7)
             27 LOAD_CONST              10 (9)
             30 BUILD_LIST              10
             33 STORE_FAST               0 (l)

  6          36 LOAD_GLOBAL              0 (sorted)
             39 LOAD_FAST                0 (l)
             42 CALL_FUNCTION            1
             45 RETURN_VALUE        

>>> 
        
```

## 测试用例 3

更快的方式：

```
def a():
    a, b, c, d, e, f, g, h, i, j = 0, 1, 2, 3, 4, 5, 6, 7, 8, 9
    return j, i, h, g, f, e, d, c, b, a

>>> 
>>> timeit(a, number=1000000)
... 0.216089963913
>>> 
>>> dis(a)
  5           0 LOAD_CONST              11 ((0, 1, 2, 3, 4, 5, 6, 7, 8, 9))
              3 UNPACK_SEQUENCE         10
              6 STORE_FAST               0 (a)
              9 STORE_FAST               1 (b)
             12 STORE_FAST               2 (c)
             15 STORE_FAST               3 (d)
             18 STORE_FAST               4 (e)
             21 STORE_FAST               5 (f)
             24 STORE_FAST               6 (g)
             27 STORE_FAST               7 (h)
             30 STORE_FAST               8 (i)
             33 STORE_FAST               9 (j)

  6          36 LOAD_FAST                9 (j)
             39 LOAD_FAST                8 (i)
             42 LOAD_FAST                7 (h)
             45 LOAD_FAST                6 (g)
             48 LOAD_FAST                5 (f)
             51 LOAD_FAST                4 (e)
             54 LOAD_FAST                3 (d)
             57 LOAD_FAST                2 (c)
             60 LOAD_FAST                1 (b)
             63 LOAD_FAST                0 (a)
             66 BUILD_TUPLE             10
             69 RETURN_VALUE        

>>> 
        
```

更慢的方式：

```
def a():
    a = 0
    b = 1
    c = 2
    d = 3
    e = 4
    f = 5
    g = 6
    h = 7
    i = 8
    j = 9
    return j, i, h, g, f, e, d, c, b, a

>>> 
>>> timeit(a, number=1000000)
... 0.249029874802
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (0)
              3 STORE_FAST               0 (a)

  6           6 LOAD_CONST               2 (1)
              9 STORE_FAST               1 (b)

  7          12 LOAD_CONST               3 (2)
             15 STORE_FAST               2 (c)

  8          18 LOAD_CONST               4 (3)
             21 STORE_FAST               3 (d)

  9          24 LOAD_CONST               5 (4)
             27 STORE_FAST               4 (e)

 10          30 LOAD_CONST               6 (5)
             33 STORE_FAST               5 (f)

 11          36 LOAD_CONST               7 (6)
             39 STORE_FAST               6 (g)

 12          42 LOAD_CONST               8 (7)
             45 STORE_FAST               7 (h)

 13          48 LOAD_CONST               9 (8)
             51 STORE_FAST               8 (i)

 14          54 LOAD_CONST              10 (9)
             57 STORE_FAST               9 (j)

 15          60 LOAD_FAST                9 (j)
             63 LOAD_FAST                8 (i)
             66 LOAD_FAST                7 (h)
             69 LOAD_FAST                6 (g)
             72 LOAD_FAST                5 (f)
             75 LOAD_FAST                4 (e)
             78 LOAD_FAST                3 (d)
             81 LOAD_FAST                2 (c)
             84 LOAD_FAST                1 (b)
             87 LOAD_FAST                0 (a)
             90 BUILD_TUPLE             10
             93 RETURN_VALUE        

>>> 
        
```

## 测试用例 4 

更快的方式：

```
def a():
    a, b, c, d, e, f = 2, 5, 52, 25, 225, 552
    if a < b and b < c and c < d and d < e and e < f:
        return True
    return False

>>> 
>>> timeit(a, number=1000000)
... 0.176034927368
>>> 
>>> dis(a)
  5           0 LOAD_CONST               7 ((2, 5, 52, 25, 225, 552))
              3 UNPACK_SEQUENCE          6
              6 STORE_FAST               0 (a)
              9 STORE_FAST               1 (b)
             12 STORE_FAST               2 (c)
             15 STORE_FAST               3 (d)
             18 STORE_FAST               4 (e)
             21 STORE_FAST               5 (f)

  6          24 LOAD_FAST                0 (a)
             27 LOAD_FAST                1 (b)
             30 COMPARE_OP               0 (<)
             33 POP_JUMP_IF_FALSE       88
             36 LOAD_FAST                1 (b)
             39 LOAD_FAST                2 (c)
             42 COMPARE_OP               0 (<)
             45 POP_JUMP_IF_FALSE       88
             48 LOAD_FAST                2 (c)
             51 LOAD_FAST                3 (d)
             54 COMPARE_OP               0 (<)
             57 POP_JUMP_IF_FALSE       88
             60 LOAD_FAST                3 (d)
             63 LOAD_FAST                4 (e)
             66 COMPARE_OP               0 (<)
             69 POP_JUMP_IF_FALSE       88
             72 LOAD_FAST                4 (e)
             75 LOAD_FAST                5 (f)
             78 COMPARE_OP               0 (<)
             81 POP_JUMP_IF_FALSE       88

  7          84 LOAD_GLOBAL              0 (True)
             87 RETURN_VALUE        

  8     >>   88 LOAD_GLOBAL              1 (False)
             91 RETURN_VALUE        

>>> 
```

更慢的方式：

```
def a():
    a, b, c, d, e, f = 2, 5, 52, 25, 225, 552
    if a < b < c < d < e < f:
        return True
    return False

>>> 
>>> timeit(a, number=1000000)
... 0.208202123642
>>> 
>>> dis(a)
  5           0 LOAD_CONST               7 ((2, 5, 52, 25, 225, 552))
              3 UNPACK_SEQUENCE          6
              6 STORE_FAST               0 (a)
              9 STORE_FAST               1 (b)
             12 STORE_FAST               2 (c)
             15 STORE_FAST               3 (d)
             18 STORE_FAST               4 (e)
             21 STORE_FAST               5 (f)

  6          24 LOAD_FAST                0 (a)
             27 LOAD_FAST                1 (b)
             30 DUP_TOP             
             31 ROT_THREE           
             32 COMPARE_OP               0 (<)
             35 JUMP_IF_FALSE_OR_POP    80
             38 LOAD_FAST                2 (c)
             41 DUP_TOP             
             42 ROT_THREE           
             43 COMPARE_OP               0 (<)
             46 JUMP_IF_FALSE_OR_POP    80
             49 LOAD_FAST                3 (d)
             52 DUP_TOP             
             53 ROT_THREE           
             54 COMPARE_OP               0 (<)
             57 JUMP_IF_FALSE_OR_POP    80
             60 LOAD_FAST                4 (e)
             63 DUP_TOP             
             64 ROT_THREE           
             65 COMPARE_OP               0 (<)
             68 JUMP_IF_FALSE_OR_POP    80
             71 LOAD_FAST                5 (f)
             74 COMPARE_OP               0 (<)
             77 JUMP_FORWARD             2 (to 82)
        >>   80 ROT_TWO             
             81 POP_TOP             
        >>   82 POP_JUMP_IF_FALSE       89

  7          85 LOAD_GLOBAL              0 (True)
             88 RETURN_VALUE        

  8     >>   89 LOAD_GLOBAL              1 (False)
             92 RETURN_VALUE        

>>> 
        
```

## 测试用例 5 

更快的方式：

```
def a():
    a = True
    if a:
        return True
    return False

>>> 
>>> timeit(a, number=1000000)
... 0.109908103943
>>> 
>>> dis(a)
  5           0 LOAD_GLOBAL              0 (True)
              3 STORE_FAST               0 (a)

  6           6 LOAD_FAST                0 (a)
              9 POP_JUMP_IF_FALSE       16

  7          12 LOAD_GLOBAL              0 (True)
             15 RETURN_VALUE        

  8     >>   16 LOAD_GLOBAL              1 (False)
             19 RETURN_VALUE        

>>> 
```

更慢方式：

```
def a():
    a = True
    if a is True:
        return True
    return False

>>> 
>>> timeit(a, number=1000000)
... 0.141321897507
>>> 
>>> dis(a)
  5           0 LOAD_GLOBAL              0 (True)
              3 STORE_FAST               0 (a)

  6           6 LOAD_FAST                0 (a)
              9 LOAD_GLOBAL              0 (True)
             12 COMPARE_OP               8 (is)
             15 POP_JUMP_IF_FALSE       22

  7          18 LOAD_GLOBAL              0 (True)
             21 RETURN_VALUE        

  8     >>   22 LOAD_GLOBAL              1 (False)
             25 RETURN_VALUE        

>>> 
        
```

最慢的方式：

```
def a():
    a = True
    if a == True:
        return True
    return False

>>> 
>>> timeit(a, number=1000000)
... 0.149966001511
>>> 
>>> dis(a)
  5           0 LOAD_GLOBAL              0 (True)
              3 STORE_FAST               0 (a)

  6           6 LOAD_FAST                0 (a)
              9 LOAD_GLOBAL              0 (True)
             12 COMPARE_OP               2 (==)
             15 POP_JUMP_IF_FALSE       22

  7          18 LOAD_GLOBAL              0 (True)
             21 RETURN_VALUE        

  8     >>   22 LOAD_GLOBAL              1 (False)
             25 RETURN_VALUE        

>>> 

```

## 测试用例 6

更快的方式：

```
def a():
    a = 1
    if not a is 2:
        return True
    return False

>>> 
>>> timeit(a, number=1000000)
... 0.0980911254883
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (1)
              3 STORE_FAST               0 (a)

  6           6 LOAD_FAST                0 (a)
              9 LOAD_CONST               2 (2)
             12 COMPARE_OP               9 (is not)
             15 POP_JUMP_IF_FALSE       22

  7          18 LOAD_GLOBAL              0 (True)
             21 RETURN_VALUE        

  8     >>   22 LOAD_GLOBAL              1 (False)
             25 RETURN_VALUE        

>>> 

```

更慢的方式：

```
def a():
    a = 1
    if a is not 2:
        return True
    return False

>>> 
>>> timeit(a, number=1000000)
... 0.10059595108
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (1)
              3 STORE_FAST               0 (a)

  6           6 LOAD_FAST                0 (a)
              9 LOAD_CONST               2 (2)
             12 COMPARE_OP               9 (is not)
             15 POP_JUMP_IF_FALSE       22

  7          18 LOAD_GLOBAL              0 (True)
             21 RETURN_VALUE        

  8     >>   22 LOAD_GLOBAL              1 (False)
             25 RETURN_VALUE        

>>> 

```

最慢的方式：

```
def a():
    a = 1
    if a != 2:
        return True
    return False

>>> 
>>> timeit(a, number=1000000)
... 0.102056026459
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (1)
              3 STORE_FAST               0 (a)

  6           6 LOAD_FAST                0 (a)
              9 LOAD_CONST               2 (2)
             12 COMPARE_OP               3 (!=)
             15 POP_JUMP_IF_FALSE       22

  7          18 LOAD_GLOBAL              0 (True)
             21 RETURN_VALUE        

  8     >>   22 LOAD_GLOBAL              1 (False)
             25 RETURN_VALUE        

>>> 
        
```

## 测试用例 7

更快的方式：

```
def a():
    a = []
    if a:
        return False
    return True

>>> 
>>> timeit(a, number=1000000)
... 0.111177921295
>>> 
>>> dis(a)
  5           0 BUILD_LIST               0
              3 STORE_FAST               0 (a)

  6           6 LOAD_FAST                0 (a)
              9 POP_JUMP_IF_FALSE       16

  7          12 LOAD_GLOBAL              0 (False)
             15 RETURN_VALUE        

  8     >>   16 LOAD_GLOBAL              1 (True)
             19 RETURN_VALUE        

>>> 
```

更慢的方式：

```
def a():
    a = []
    if not a:
        return True
    return False

>>> 
>>> timeit(a, number=1000000)
... 0.113872051239
>>> 
>>> dis(a)
  5           0 BUILD_LIST               0
              3 STORE_FAST               0 (a)

  6           6 LOAD_FAST                0 (a)
              9 POP_JUMP_IF_TRUE        16

  7          12 LOAD_GLOBAL              0 (True)
             15 RETURN_VALUE        

  8     >>   16 LOAD_GLOBAL              1 (False)
             19 RETURN_VALUE        

>>> 
```

最慢的方式：

```
def a():
    a = []
    if len(a) <= 0:
        return True
    return False

>>> 
>>> timeit(a, number=1000000)
... 0.13893699646
>>> 
>>> dis(a)
  5           0 BUILD_LIST               0
              3 STORE_FAST               0 (a)

  6           6 LOAD_GLOBAL              0 (len)
              9 LOAD_FAST                0 (a)
             12 CALL_FUNCTION            1
             15 LOAD_CONST               1 (0)
             18 COMPARE_OP               1 (<=)
             21 POP_JUMP_IF_FALSE       28

  7          24 LOAD_GLOBAL              1 (True)
             27 RETURN_VALUE        

  8     >>   28 LOAD_GLOBAL              2 (False)
             31 RETURN_VALUE        

>>> 
```

超级慢的方式：

```
def a():
    a = []
    if a == []:
        return True
    return False

>>> 
>>> timeit(a, number=1000000)
... 0.16094994545
>>> 
>>> dis(a)
  5           0 BUILD_LIST               0
              3 STORE_FAST               0 (a)

  6           6 LOAD_FAST                0 (a)
              9 BUILD_LIST               0
             12 COMPARE_OP               2 (==)
             15 POP_JUMP_IF_FALSE       22

  7          18 LOAD_GLOBAL              0 (True)
             21 RETURN_VALUE        

  8     >>   22 LOAD_GLOBAL              1 (False)
             25 RETURN_VALUE        

>>> 
        
```


## 测试用例 8

更快的方式：

```
def a():
    a = object()
    if not a:
        return False
    return True

>>> 
>>> timeit(a, number=1000000)
... 0.150635004044
>>> 
>>> dis(a)
  5           0 LOAD_GLOBAL              0 (object)
              3 CALL_FUNCTION            0
              6 STORE_FAST               0 (a)

  6           9 LOAD_FAST                0 (a)
             12 POP_JUMP_IF_TRUE        19

  7          15 LOAD_GLOBAL              1 (False)
             18 RETURN_VALUE        

  8     >>   19 LOAD_GLOBAL              2 (True)
             22 RETURN_VALUE        

>>> 
```

更慢的方式：

```
def a():
    a = object()
    if a is None:
        return False
    return True

>>> 
>>> timeit(a, number=1000000)
... 0.15754199028
>>> 
>>> dis(a)
  5           0 LOAD_GLOBAL              0 (object)
              3 CALL_FUNCTION            0
              6 STORE_FAST               0 (a)

  6           9 LOAD_FAST                0 (a)
             12 LOAD_CONST               0 (None)
             15 COMPARE_OP               8 (is)
             18 POP_JUMP_IF_FALSE       25

  7          21 LOAD_GLOBAL              2 (False)
             24 RETURN_VALUE        

  8     >>   25 LOAD_GLOBAL              3 (True)
             28 RETURN_VALUE        

>>> 
```

最慢的方式：

```
def a():
    a = object()
    if a:
        return True
    return False

>>> 
>>> timeit(a, number=1000000)
... 0.157824993134
>>> 
>>> dis(a)
  5           0 LOAD_GLOBAL              0 (object)
              3 CALL_FUNCTION            0
              6 STORE_FAST               0 (a)

  6           9 LOAD_FAST                0 (a)
             12 POP_JUMP_IF_FALSE       19

  7          15 LOAD_GLOBAL              1 (True)
             18 RETURN_VALUE        

  8     >>   19 LOAD_GLOBAL              2 (False)
             22 RETURN_VALUE        

>>> 
```

超级慢的方式：

```
def a():
    a = object()
    if a is not None:
        return True
    return False

>>> 
>>> timeit(a, number=1000000)
... 0.181853055954
>>> 
>>> dis(a)
  5           0 LOAD_GLOBAL              0 (object)
              3 CALL_FUNCTION            0
              6 STORE_FAST               0 (a)

  6           9 LOAD_FAST                0 (a)
             12 LOAD_CONST               0 (None)
             15 COMPARE_OP               9 (is not)
             18 POP_JUMP_IF_FALSE       25

  7          21 LOAD_GLOBAL              2 (True)
             24 RETURN_VALUE        

  8     >>   25 LOAD_GLOBAL              3 (False)
             28 RETURN_VALUE        

>>> 
        
```



## 测试用例 9

更快的方式：

```
def a():
    a = [1, 2, 3, 4, 5]
    s = 0
    for p, v in enumerate(a):
        s += p
        s += v
    return s

>>> 
>>> timeit(a, number=1000000)
... 0.738418102264
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (1)
              3 LOAD_CONST               2 (2)
              6 LOAD_CONST               3 (3)
              9 LOAD_CONST               4 (4)
             12 LOAD_CONST               5 (5)
             15 BUILD_LIST               5
             18 STORE_FAST               0 (a)

  6          21 LOAD_CONST               6 (0)
             24 STORE_FAST               1 (s)

  7          27 SETUP_LOOP              46 (to 76)
             30 LOAD_GLOBAL              0 (enumerate)
             33 LOAD_FAST                0 (a)
             36 CALL_FUNCTION            1
             39 GET_ITER            
        >>   40 FOR_ITER                32 (to 75)
             43 UNPACK_SEQUENCE          2
             46 STORE_FAST               2 (p)
             49 STORE_FAST               3 (v)

  8          52 LOAD_FAST                1 (s)
             55 LOAD_FAST                2 (p)
             58 INPLACE_ADD         
             59 STORE_FAST               1 (s)

  9          62 LOAD_FAST                1 (s)
             65 LOAD_FAST                3 (v)
             68 INPLACE_ADD         
             69 STORE_FAST               1 (s)
             72 JUMP_ABSOLUTE           40
        >>   75 POP_BLOCK           

 10     >>   76 LOAD_FAST                1 (s)
             79 RETURN_VALUE        

>>> 
```

更慢的方式：

```
def a():
    a = [1, 2, 3, 4, 5]
    s = 0
    for i in range(len(a)):
        s += i
        s += a[i]
    return s

>>> 
>>> timeit(a, number=1000000)
... 0.800552845001
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (1)
              3 LOAD_CONST               2 (2)
              6 LOAD_CONST               3 (3)
              9 LOAD_CONST               4 (4)
             12 LOAD_CONST               5 (5)
             15 BUILD_LIST               5
             18 STORE_FAST               0 (a)

  6          21 LOAD_CONST               6 (0)
             24 STORE_FAST               1 (s)

  7          27 SETUP_LOOP              50 (to 80)
             30 LOAD_GLOBAL              0 (range)
             33 LOAD_GLOBAL              1 (len)
             36 LOAD_FAST                0 (a)
             39 CALL_FUNCTION            1
             42 CALL_FUNCTION            1
             45 GET_ITER            
        >>   46 FOR_ITER                30 (to 79)
             49 STORE_FAST               2 (i)

  8          52 LOAD_FAST                1 (s)
             55 LOAD_FAST                2 (i)
             58 INPLACE_ADD         
             59 STORE_FAST               1 (s)

  9          62 LOAD_FAST                1 (s)
             65 LOAD_FAST                0 (a)
             68 LOAD_FAST                2 (i)
             71 BINARY_SUBSCR       
             72 INPLACE_ADD         
             73 STORE_FAST               1 (s)
             76 JUMP_ABSOLUTE           46
        >>   79 POP_BLOCK           

 10     >>   80 LOAD_FAST                1 (s)
             83 RETURN_VALUE        

>>> 
        
```

## 测试用例 10

更快的方式：

```
def a():
    r = ''
    for i in range(10):
        r += str(i)
    return r

>>> 
>>> timeit(a, number=1000000)
... 1.63575387001
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 ('')
              3 STORE_FAST               0 (r)

  6           6 SETUP_LOOP              36 (to 45)
              9 LOAD_GLOBAL              0 (range)
             12 LOAD_CONST               2 (10)
             15 CALL_FUNCTION            1
             18 GET_ITER            
        >>   19 FOR_ITER                22 (to 44)
             22 STORE_FAST               1 (i)

  7          25 LOAD_FAST                0 (r)
             28 LOAD_GLOBAL              1 (str)
             31 LOAD_FAST                1 (i)
             34 CALL_FUNCTION            1
             37 INPLACE_ADD         
             38 STORE_FAST               0 (r)
             41 JUMP_ABSOLUTE           19
        >>   44 POP_BLOCK           

  8     >>   45 LOAD_FAST                0 (r)
             48 RETURN_VALUE        

>>> 
        
```

更慢的方式：

```
def a():
    r = []
    for i in range(10):
        r.append(str(i))
    return ''.join(r)

>>> 
>>> timeit(a, number=1000000)
... 2.1280169487
>>> 
>>> dis(a)
  5           0 BUILD_LIST               0
              3 STORE_FAST               0 (r)

  6           6 SETUP_LOOP              39 (to 48)
              9 LOAD_GLOBAL              0 (range)
             12 LOAD_CONST               1 (10)
             15 CALL_FUNCTION            1
             18 GET_ITER            
        >>   19 FOR_ITER                25 (to 47)
             22 STORE_FAST               1 (i)

  7          25 LOAD_FAST                0 (r)
             28 LOAD_ATTR                1 (append)
             31 LOAD_GLOBAL              2 (str)
             34 LOAD_FAST                1 (i)
             37 CALL_FUNCTION            1
             40 CALL_FUNCTION            1
             43 POP_TOP             
             44 JUMP_ABSOLUTE           19
        >>   47 POP_BLOCK           

  8     >>   48 LOAD_CONST               2 ('')
             51 LOAD_ATTR                3 (join)
             54 LOAD_FAST                0 (r)
             57 CALL_FUNCTION            1
             60 RETURN_VALUE        

>>> 

```

## 测试用例 11

更快的方式：

```
def a():
    a = 5
    b = 2
    c = 3
    return "%s" % (a*(b+c))

>>> 
>>> timeit(a, number=1000000)
... 0.229720830917
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (5)
              3 STORE_FAST               0 (a)

  6           6 LOAD_CONST               2 (2)
              9 STORE_FAST               1 (b)

  7          12 LOAD_CONST               3 (3)
             15 STORE_FAST               2 (c)

  8          18 LOAD_CONST               4 ('%s')
             21 LOAD_FAST                0 (a)
             24 LOAD_FAST                1 (b)
             27 LOAD_FAST                2 (c)
             30 BINARY_ADD          
             31 BINARY_MULTIPLY     
             32 BINARY_MODULO       
             33 RETURN_VALUE        

>>> 
        
```

更慢的方式：

```

def a():
    a = 5
    b = 2
    c = 3
    return str(a*(b+c))

>>> 
>>> timeit(a, number=1000000)
... 0.25043797493
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (5)
              3 STORE_FAST               0 (a)

  6           6 LOAD_CONST               2 (2)
              9 STORE_FAST               1 (b)

  7          12 LOAD_CONST               3 (3)
             15 STORE_FAST               2 (c)

  8          18 LOAD_GLOBAL              0 (str)
             21 LOAD_FAST                0 (a)
             24 LOAD_FAST                1 (b)
             27 LOAD_FAST                2 (c)
             30 BINARY_ADD          
             31 BINARY_MULTIPLY     
             32 CALL_FUNCTION            1
             35 RETURN_VALUE        

>>> 
```

最慢的方式：

```
def a():
    a = 5
    b = 2
    c = 3
    return "%d" % (a*(b+c))

>>> 
>>> timeit(a, number=1000000)
... 0.583065986633
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (5)
              3 STORE_FAST               0 (a)

  6           6 LOAD_CONST               2 (2)
              9 STORE_FAST               1 (b)

  7          12 LOAD_CONST               3 (3)
             15 STORE_FAST               2 (c)

  8          18 LOAD_CONST               4 ('%d')
             21 LOAD_FAST                0 (a)
             24 LOAD_FAST                1 (b)
             27 LOAD_FAST                2 (c)
             30 BINARY_ADD          
             31 BINARY_MULTIPLY     
             32 BINARY_MODULO       
             33 RETURN_VALUE        

>>> 
        

```

## 测试用例 12

更快的方式：

```
def a():
    a = [1, 2, 3, 4, 5]
    return len(a)

>>> 
>>> timeit(a, number=1000000)
... 0.194615840912
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (1)
              3 LOAD_CONST               2 (2)
              6 LOAD_CONST               3 (3)
              9 LOAD_CONST               4 (4)
             12 LOAD_CONST               5 (5)
             15 BUILD_LIST               5
             18 STORE_FAST               0 (a)

  6          21 LOAD_GLOBAL              0 (len)
             24 LOAD_FAST                0 (a)
             27 CALL_FUNCTION            1
             30 RETURN_VALUE        

>>> 
        

```

更慢的方式：

```
def a():
    a = [1, 2, 3, 4, 5]
    return a.__len__()

>>> 
>>> timeit(a, number=1000000)
... 0.220345020294
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (1)
              3 LOAD_CONST               2 (2)
              6 LOAD_CONST               3 (3)
              9 LOAD_CONST               4 (4)
             12 LOAD_CONST               5 (5)
             15 BUILD_LIST               5
             18 STORE_FAST               0 (a)

  6          21 LOAD_FAST                0 (a)
             24 LOAD_ATTR                0 (__len__)
             27 CALL_FUNCTION            0
             30 RETURN_VALUE        

>>> 
```

## 测试用例 13

更快的方式：

```
def a():
    a = 1
    b = 2
    c = 2
    d = 5
    return (a+b+c)*d

>>> 
>>> timeit(a, number=1000000)
... 0.170273065567
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (1)
              3 STORE_FAST               0 (a)

  6           6 LOAD_CONST               2 (2)
              9 STORE_FAST               1 (b)

  7          12 LOAD_CONST               2 (2)
             15 STORE_FAST               2 (c)

  8          18 LOAD_CONST               3 (5)
             21 STORE_FAST               3 (d)

  9          24 LOAD_FAST                0 (a)
             27 LOAD_FAST                1 (b)
             30 BINARY_ADD          
             31 LOAD_FAST                2 (c)
             34 BINARY_ADD          
             35 LOAD_FAST                3 (d)
             38 BINARY_MULTIPLY     
             39 RETURN_VALUE        

>>> 
        
```

更慢的方式：

```
def a():
    a = 1
    b = 2
    c = 2
    d = 5
    return (a.__add__(b.__add__(c))).__mul__(d)

>>> 
>>> timeit(a, number=1000000)
... 0.363096952438
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (1)
              3 STORE_FAST               0 (a)

  6           6 LOAD_CONST               2 (2)
              9 STORE_FAST               1 (b)

  7          12 LOAD_CONST               2 (2)
             15 STORE_FAST               2 (c)

  8          18 LOAD_CONST               3 (5)
             21 STORE_FAST               3 (d)

  9          24 LOAD_FAST                0 (a)
             27 LOAD_ATTR                0 (__add__)
             30 LOAD_FAST                1 (b)
             33 LOAD_ATTR                0 (__add__)
             36 LOAD_FAST                2 (c)
             39 CALL_FUNCTION            1
             42 CALL_FUNCTION            1
             45 LOAD_ATTR                1 (__mul__)
             48 LOAD_FAST                3 (d)
             51 CALL_FUNCTION            1
             54 RETURN_VALUE        

>>> 
```

## 测试用例 14

更快的方式：

```
class Z():

    def __init__(self, v):
        self.v = v

    def __mul__(self, o):
        return Z(self.v * o.v)

    def __add__(self, o):
        return Z(self.v + o.v)


def a():
    a = Z(5)
    b = Z(2)
    c = Z(3)
    return (b.__add__(c)).__mul__(a)

>>> 
>>> timeit(a, number=1000000)
... 1.7775759697
>>> 
>>> dis(a)
 17           0 LOAD_GLOBAL              0 (Z)
              3 LOAD_CONST               1 (5)
              6 CALL_FUNCTION            1
              9 STORE_FAST               0 (a)

 18          12 LOAD_GLOBAL              0 (Z)
             15 LOAD_CONST               2 (2)
             18 CALL_FUNCTION            1
             21 STORE_FAST               1 (b)

 19          24 LOAD_GLOBAL              0 (Z)
             27 LOAD_CONST               3 (3)
             30 CALL_FUNCTION            1
             33 STORE_FAST               2 (c)

 20          36 LOAD_FAST                1 (b)
             39 LOAD_ATTR                1 (__add__)
             42 LOAD_FAST                2 (c)
             45 CALL_FUNCTION            1
             48 LOAD_ATTR                2 (__mul__)
             51 LOAD_FAST                0 (a)
             54 CALL_FUNCTION            1
             57 RETURN_VALUE        

>>> 
        
```

更慢的方式：

```
class Z():

    def __init__(self, v):
        self.v = v

    def __mul__(self, o):
        return Z(self.v * o.v)

    def __add__(self, o):
        return Z(self.v + o.v)


def a():
    a = Z(5)
    b = Z(2)
    c = Z(3)
    return (b + c) * a

>>> 
>>> timeit(a, number=1000000)
... 2.79053497314
>>> 
>>> dis(a)
 17           0 LOAD_GLOBAL              0 (Z)
              3 LOAD_CONST               1 (5)
              6 CALL_FUNCTION            1
              9 STORE_FAST               0 (a)

 18          12 LOAD_GLOBAL              0 (Z)
             15 LOAD_CONST               2 (2)
             18 CALL_FUNCTION            1
             21 STORE_FAST               1 (b)

 19          24 LOAD_GLOBAL              0 (Z)
             27 LOAD_CONST               3 (3)
             30 CALL_FUNCTION            1
             33 STORE_FAST               2 (c)

 20          36 LOAD_FAST                1 (b)
             39 LOAD_FAST                2 (c)
             42 BINARY_ADD          
             43 LOAD_FAST                0 (a)
             46 BINARY_MULTIPLY     
             47 RETURN_VALUE        

>>> 
```

## 测试用例 15

更快的方式：

```
def a():
    s = 0
    for i in range(50000):
        s += i
    return s


number = 100000

>>> 
>>> timeit(a, number=100000)
... 177.31340313
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (0)
              3 STORE_FAST               0 (s)

  6           6 SETUP_LOOP              30 (to 39)
              9 LOAD_GLOBAL              0 (range)
             12 LOAD_CONST               2 (50000)
             15 CALL_FUNCTION            1
             18 GET_ITER            
        >>   19 FOR_ITER                16 (to 38)
             22 STORE_FAST               1 (i)

  7          25 LOAD_FAST                0 (s)
             28 LOAD_FAST                1 (i)
             31 INPLACE_ADD         
             32 STORE_FAST               0 (s)
             35 JUMP_ABSOLUTE           19
        >>   38 POP_BLOCK           

  8     >>   39 LOAD_FAST                0 (s)
             42 RETURN_VALUE        

>>> 
```

更慢的方式：

```
def a():
    return sum(i for i in range(50000))


number = 100000

>>> 
>>> timeit(a, number=100000)
... 215.36849308
>>> 
>>> dis(a)
  5           0 LOAD_GLOBAL              0 (sum)
              3 LOAD_CONST               1 (<code object <genexpr> at 0x7f7357e854b0, file "/tmp/__pfw_test15_t2.py", line 5>)
              6 MAKE_FUNCTION            0
              9 LOAD_GLOBAL              1 (range)
             12 LOAD_CONST               2 (50000)
             15 CALL_FUNCTION            1
             18 GET_ITER            
             19 CALL_FUNCTION            1
             22 CALL_FUNCTION            1
             25 RETURN_VALUE        

>>> 
        
```

## 测试用例 16

更快的方式：

```
def a():
    return [i for i in range(1000)]


number = 100000

>>> 
>>> timeit(a, number=100000)
... 3.07614183426
>>> 
>>> dis(a)
  5           0 BUILD_LIST               0
              3 LOAD_GLOBAL              0 (range)
              6 LOAD_CONST               1 (1000)
              9 CALL_FUNCTION            1
             12 GET_ITER            
        >>   13 FOR_ITER                12 (to 28)
             16 STORE_FAST               0 (i)
             19 LOAD_FAST                0 (i)
             22 LIST_APPEND              2
             25 JUMP_ABSOLUTE           13
        >>   28 RETURN_VALUE        

>>> 
        
```

更慢的方式：

```
def a():
    l = []
    for i in range(1000):
        l.append(i)
    return l


number = 100000

>>> 
>>> timeit(a, number=100000)
... 6.29185986519
>>> 
>>> dis(a)
  5           0 BUILD_LIST               0
              3 STORE_FAST               0 (l)

  6           6 SETUP_LOOP              33 (to 42)
              9 LOAD_GLOBAL              0 (range)
             12 LOAD_CONST               1 (1000)
             15 CALL_FUNCTION            1
             18 GET_ITER            
        >>   19 FOR_ITER                19 (to 41)
             22 STORE_FAST               1 (i)

  7          25 LOAD_FAST                0 (l)
             28 LOAD_ATTR                1 (append)
             31 LOAD_FAST                1 (i)
             34 CALL_FUNCTION            1
             37 POP_TOP             
             38 JUMP_ABSOLUTE           19
        >>   41 POP_BLOCK           

  8     >>   42 LOAD_FAST                0 (l)
             45 RETURN_VALUE        

>>> 
```

## 测试用例 17

更快的方式：

```
def a():
    return {str(i): i*2 for i in range(100)}

>>> 
>>> timeit(a, number=1000000)
... 17.8982248306
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (<code object <dictcomp> at 0x7f81ea7854b0, file "/tmp/__pfw_test17_t3.py", line 5>)
              3 MAKE_FUNCTION            0
              6 LOAD_GLOBAL              0 (range)
              9 LOAD_CONST               2 (100)
             12 CALL_FUNCTION            1
             15 GET_ITER            
             16 CALL_FUNCTION            1
             19 RETURN_VALUE        

>>> 
        
```

更慢的方式：

```
def a():
    d = {}
    for i in range(100):
        d[str(i)] = i*2
    return d

>>> 
>>> timeit(a, number=1000000)
... 18.513548851
>>> 
>>> dis(a)
  5           0 BUILD_MAP                0
              3 STORE_FAST               0 (d)

  6           6 SETUP_LOOP              40 (to 49)
              9 LOAD_GLOBAL              0 (range)
             12 LOAD_CONST               1 (100)
             15 CALL_FUNCTION            1
             18 GET_ITER            
        >>   19 FOR_ITER                26 (to 48)
             22 STORE_FAST               1 (i)

  7          25 LOAD_FAST                1 (i)
             28 LOAD_CONST               2 (2)
             31 BINARY_MULTIPLY     
             32 LOAD_FAST                0 (d)
             35 LOAD_GLOBAL              1 (str)
             38 LOAD_FAST                1 (i)
             41 CALL_FUNCTION            1
             44 STORE_SUBSCR        
             45 JUMP_ABSOLUTE           19
        >>   48 POP_BLOCK           

  8     >>   49 LOAD_FAST                0 (d)
             52 RETURN_VALUE        

>>> 
        
```

最慢的方式：

```
def a():
    d = {}
    for i in range(100):
        d.update({str(i): i*2})
    return d

>>> 
>>> timeit(a, number=1000000)
... 36.9908900261
>>> 
>>> dis(a)
  5           0 BUILD_MAP                0
              3 STORE_FAST               0 (d)

  6           6 SETUP_LOOP              50 (to 59)
              9 LOAD_GLOBAL              0 (range)
             12 LOAD_CONST               1 (100)
             15 CALL_FUNCTION            1
             18 GET_ITER            
        >>   19 FOR_ITER                36 (to 58)
             22 STORE_FAST               1 (i)

  7          25 LOAD_FAST                0 (d)
             28 LOAD_ATTR                1 (update)
             31 BUILD_MAP                1
             34 LOAD_FAST                1 (i)
             37 LOAD_CONST               2 (2)
             40 BINARY_MULTIPLY     
             41 LOAD_GLOBAL              2 (str)
             44 LOAD_FAST                1 (i)
             47 CALL_FUNCTION            1
             50 STORE_MAP           
             51 CALL_FUNCTION            1
             54 POP_TOP             
             55 JUMP_ABSOLUTE           19
        >>   58 POP_BLOCK           

  8     >>   59 LOAD_FAST                0 (d)
             62 RETURN_VALUE        

>>> 
        
```

## 测试用例 18

更快的方式：

```
def a():
    l = range(50, -20, -2)
    return {p: v for p, v in enumerate(l)}

>>> 
>>> timeit(a, number=1000000)
... 2.82184195518
>>> 
>>> dis(a)
  5           0 LOAD_GLOBAL              0 (range)
              3 LOAD_CONST               1 (50)
              6 LOAD_CONST               2 (-20)
              9 LOAD_CONST               3 (-2)
             12 CALL_FUNCTION            3
             15 STORE_FAST               0 (l)

  6          18 LOAD_CONST               4 (<code object <dictcomp> at 0x7ff850f7a4b0, file "/tmp/__pfw_test18_t3.py", line 6>)
             21 MAKE_FUNCTION            0
             24 LOAD_GLOBAL              1 (enumerate)
             27 LOAD_FAST                0 (l)
             30 CALL_FUNCTION            1
             33 GET_ITER            
             34 CALL_FUNCTION            1
             37 RETURN_VALUE        

>>> 
```

更慢的方式：

```
def a():
    l = range(50, -20, -2)
    d = {}
    for p, v in enumerate(l):
        d[p] = v
    return d

>>> 
>>> timeit(a, number=1000000)
... 3.04933786392
>>> 
>>> dis(a)
  5           0 LOAD_GLOBAL              0 (range)
              3 LOAD_CONST               1 (50)
              6 LOAD_CONST               2 (-20)
              9 LOAD_CONST               3 (-2)
             12 CALL_FUNCTION            3
             15 STORE_FAST               0 (l)

  6          18 BUILD_MAP                0
             21 STORE_FAST               1 (d)

  7          24 SETUP_LOOP              36 (to 63)
             27 LOAD_GLOBAL              1 (enumerate)
             30 LOAD_FAST                0 (l)
             33 CALL_FUNCTION            1
             36 GET_ITER            
        >>   37 FOR_ITER                22 (to 62)
             40 UNPACK_SEQUENCE          2
             43 STORE_FAST               2 (p)
             46 STORE_FAST               3 (v)

  8          49 LOAD_FAST                3 (v)
             52 LOAD_FAST                1 (d)
             55 LOAD_FAST                2 (p)
             58 STORE_SUBSCR        
             59 JUMP_ABSOLUTE           37
        >>   62 POP_BLOCK           

  9     >>   63 LOAD_FAST                1 (d)
             66 RETURN_VALUE        

>>> 
        
```

最慢的方式：

```
def a():
    l = range(50, -20, -2)
    d = {}
    for p, v in enumerate(l):
        d.update({p: v})
    return d

>>> 
>>> timeit(a, number=1000000)
... 9.87823104858
>>> 
>>> dis(a)
  5           0 LOAD_GLOBAL              0 (range)
              3 LOAD_CONST               1 (50)
              6 LOAD_CONST               2 (-20)
              9 LOAD_CONST               3 (-2)
             12 CALL_FUNCTION            3
             15 STORE_FAST               0 (l)

  6          18 BUILD_MAP                0
             21 STORE_FAST               1 (d)

  7          24 SETUP_LOOP              46 (to 73)
             27 LOAD_GLOBAL              1 (enumerate)
             30 LOAD_FAST                0 (l)
             33 CALL_FUNCTION            1
             36 GET_ITER            
        >>   37 FOR_ITER                32 (to 72)
             40 UNPACK_SEQUENCE          2
             43 STORE_FAST               2 (p)
             46 STORE_FAST               3 (v)

  8          49 LOAD_FAST                1 (d)
             52 LOAD_ATTR                2 (update)
             55 BUILD_MAP                1
             58 LOAD_FAST                3 (v)
             61 LOAD_FAST                2 (p)
             64 STORE_MAP           
             65 CALL_FUNCTION            1
             68 POP_TOP             
             69 JUMP_ABSOLUTE           37
        >>   72 POP_BLOCK           

  9     >>   73 LOAD_FAST                1 (d)
             76 RETURN_VALUE        

>>> 
```

## 测试用例 19

更快的方式：

```
def a():
    a = 1
    if a == 1 or a == 2 or a == 3:
        return True
    return False

>>> 
>>> timeit(a, number=1000000)
... 0.100009918213
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (1)
              3 STORE_FAST               0 (a)

  6           6 LOAD_FAST                0 (a)
              9 LOAD_CONST               1 (1)
             12 COMPARE_OP               2 (==)
             15 POP_JUMP_IF_TRUE        42
             18 LOAD_FAST                0 (a)
             21 LOAD_CONST               2 (2)
             24 COMPARE_OP               2 (==)
             27 POP_JUMP_IF_TRUE        42
             30 LOAD_FAST                0 (a)
             33 LOAD_CONST               3 (3)
             36 COMPARE_OP               2 (==)
             39 POP_JUMP_IF_FALSE       46

  7     >>   42 LOAD_GLOBAL              0 (True)
             45 RETURN_VALUE        

  8     >>   46 LOAD_GLOBAL              1 (False)
             49 RETURN_VALUE        

>>> 
        
```

更慢的方式：

```
def a():
    a = 1
    if a in (1, 2, 3):
        return True
    return False

>>> 
>>> timeit(a, number=1000000)
... 0.105048894882
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (1)
              3 STORE_FAST               0 (a)

  6           6 LOAD_FAST                0 (a)
              9 LOAD_CONST               4 ((1, 2, 3))
             12 COMPARE_OP               6 (in)
             15 POP_JUMP_IF_FALSE       22

  7          18 LOAD_GLOBAL              0 (True)
             21 RETURN_VALUE        

  8     >>   22 LOAD_GLOBAL              1 (False)
             25 RETURN_VALUE        

>>> 
        
```



## 测试用例 20

更快的方式：

```

```

更慢的方式：

```

```

最慢的方式：

```

```

## 测试用例 21

更快的方式：

```
def a():
    return ''.join(map(str, xrange(10)))

>>> 
>>> timeit(a, number=1000000)
... 1.28661704063
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 ('')
              3 LOAD_ATTR                0 (join)
              6 LOAD_GLOBAL              1 (map)
              9 LOAD_GLOBAL              2 (str)
             12 LOAD_GLOBAL              3 (xrange)
             15 LOAD_CONST               2 (10)
             18 CALL_FUNCTION            1
             21 CALL_FUNCTION            2
             24 CALL_FUNCTION            1
             27 RETURN_VALUE        

>>> 
        
```

更慢的方式：

```
def a():
    return ''.join(map(str, range(10)))

>>> 
>>> timeit(a, number=1000000)
... 1.29610896111
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 ('')
              3 LOAD_ATTR                0 (join)
              6 LOAD_GLOBAL              1 (map)
              9 LOAD_GLOBAL              2 (str)
             12 LOAD_GLOBAL              3 (range)
             15 LOAD_CONST               2 (10)
             18 CALL_FUNCTION            1
             21 CALL_FUNCTION            2
             24 CALL_FUNCTION            1
             27 RETURN_VALUE        

>>> 
        
```

最慢的方式：

```
def a():
    r = ''
    for i in range(10):
        r = '%s%s' % (r, str(i))
    return r

>>> 
>>> timeit(a, number=1000000)
... 2.74961709976
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 ('')
              3 STORE_FAST               0 (r)

  6           6 SETUP_LOOP              42 (to 51)
              9 LOAD_GLOBAL              0 (range)
             12 LOAD_CONST               2 (10)
             15 CALL_FUNCTION            1
             18 GET_ITER            
        >>   19 FOR_ITER                28 (to 50)
             22 STORE_FAST               1 (i)

  7          25 LOAD_CONST               3 ('%s%s')
             28 LOAD_FAST                0 (r)
             31 LOAD_GLOBAL              1 (str)
             34 LOAD_FAST                1 (i)
             37 CALL_FUNCTION            1
             40 BUILD_TUPLE              2
             43 BINARY_MODULO       
             44 STORE_FAST               0 (r)
             47 JUMP_ABSOLUTE           19
        >>   50 POP_BLOCK           

  8     >>   51 LOAD_FAST                0 (r)
             54 RETURN_VALUE        

>>> 
```

## 测试用例 22

更快的方式：

```
def a(n=25):
    a, b = 0, 1
    for i in range(n):
        x = a + b
        a = b
        b = x
    return a

>>> 
>>> timeit(a, number=1000000)
... 1.34775996208
>>> 
>>> dis(a)
  5           0 LOAD_CONST               3 ((0, 1))
              3 UNPACK_SEQUENCE          2
              6 STORE_FAST               1 (a)
              9 STORE_FAST               2 (b)

  6          12 SETUP_LOOP              42 (to 57)
             15 LOAD_GLOBAL              0 (range)
             18 LOAD_FAST                0 (n)
             21 CALL_FUNCTION            1
             24 GET_ITER            
        >>   25 FOR_ITER                28 (to 56)
             28 STORE_FAST               3 (i)

  7          31 LOAD_FAST                1 (a)
             34 LOAD_FAST                2 (b)
             37 BINARY_ADD          
             38 STORE_FAST               4 (x)

  8          41 LOAD_FAST                2 (b)
             44 STORE_FAST               1 (a)

  9          47 LOAD_FAST                4 (x)
             50 STORE_FAST               2 (b)
             53 JUMP_ABSOLUTE           25
        >>   56 POP_BLOCK           

 10     >>   57 LOAD_FAST                1 (a)
             60 RETURN_VALUE        

>>> 
```

更慢的方式：

```
def a(n=25):
    a, b = 0, 1
    for i in range(n):
        a, b = b, a + b
    return a

>>> 
>>> timeit(a, number=1000000)
... 1.46058106422
>>> 
>>> dis(a)
  5           0 LOAD_CONST               3 ((0, 1))
              3 UNPACK_SEQUENCE          2
              6 STORE_FAST               1 (a)
              9 STORE_FAST               2 (b)

  6          12 SETUP_LOOP              37 (to 52)
             15 LOAD_GLOBAL              0 (range)
             18 LOAD_FAST                0 (n)
             21 CALL_FUNCTION            1
             24 GET_ITER            
        >>   25 FOR_ITER                23 (to 51)
             28 STORE_FAST               3 (i)

  7          31 LOAD_FAST                2 (b)
             34 LOAD_FAST                1 (a)
             37 LOAD_FAST                2 (b)
             40 BINARY_ADD          
             41 ROT_TWO             
             42 STORE_FAST               1 (a)
             45 STORE_FAST               2 (b)
             48 JUMP_ABSOLUTE           25
        >>   51 POP_BLOCK           

  8     >>   52 LOAD_FAST                1 (a)
             55 RETURN_VALUE        

>>> 
        
```



## 测试用例 23

更快的方式：

```
def a():
    x, y, z, w, k = 0, 0, 0, 0, 0
    return x, y, z, w, k

>>> 
>>> timeit(a, number=1000000)
... 0.168544054031
>>> 
>>> dis(a)
  5           0 LOAD_CONST               2 ((0, 0, 0, 0, 0))
              3 UNPACK_SEQUENCE          5
              6 STORE_FAST               0 (x)
              9 STORE_FAST               1 (y)
             12 STORE_FAST               2 (z)
             15 STORE_FAST               3 (w)
             18 STORE_FAST               4 (k)

  6          21 LOAD_FAST                0 (x)
             24 LOAD_FAST                1 (y)
             27 LOAD_FAST                2 (z)
             30 LOAD_FAST                3 (w)
             33 LOAD_FAST                4 (k)
             36 BUILD_TUPLE              5
             39 RETURN_VALUE        

>>> 
        
```

更慢的方式：

```
def a():
    x = 0
    y = 0
    z = 0
    w = 0
    k = 0
    return x, y, z, w, k

>>> 
>>> timeit(a, number=1000000)
... 0.174968957901
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (0)
              3 STORE_FAST               0 (x)

  6           6 LOAD_CONST               1 (0)
              9 STORE_FAST               1 (y)

  7          12 LOAD_CONST               1 (0)
             15 STORE_FAST               2 (z)

  8          18 LOAD_CONST               1 (0)
             21 STORE_FAST               3 (w)

  9          24 LOAD_CONST               1 (0)
             27 STORE_FAST               4 (k)

 10          30 LOAD_FAST                0 (x)
             33 LOAD_FAST                1 (y)
             36 LOAD_FAST                2 (z)
             39 LOAD_FAST                3 (w)
             42 LOAD_FAST                4 (k)
             45 BUILD_TUPLE              5
             48 RETURN_VALUE        

>>> 
        
```

最慢的方式：

```
def a():
    x = y = z = w = k = 0
    return x, y, z, w, k

>>> 
>>> timeit(a, number=1000000)
... 0.205312013626
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (0)
              3 DUP_TOP             
              4 STORE_FAST               0 (x)
              7 DUP_TOP             
              8 STORE_FAST               1 (y)
             11 DUP_TOP             
             12 STORE_FAST               2 (z)
             15 DUP_TOP             
             16 STORE_FAST               3 (w)
             19 STORE_FAST               4 (k)

  6          22 LOAD_FAST                0 (x)
             25 LOAD_FAST                1 (y)
             28 LOAD_FAST                2 (z)
             31 LOAD_FAST                3 (w)
             34 LOAD_FAST                4 (k)
             37 BUILD_TUPLE              5
             40 RETURN_VALUE        

>>> 
        
```

## 测试用例 24

更快的方式：

```
def a():
    a = [[1, 2, 3], [2, 3, 4], [4, 5, 6]]
    b = {k: v for x, k, v in a}
    return b

>>> 
>>> timeit(a, number=1000000)
... 0.717478990555
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (1)
              3 LOAD_CONST               2 (2)
              6 LOAD_CONST               3 (3)
              9 BUILD_LIST               3
             12 LOAD_CONST               2 (2)
             15 LOAD_CONST               3 (3)
             18 LOAD_CONST               4 (4)
             21 BUILD_LIST               3
             24 LOAD_CONST               4 (4)
             27 LOAD_CONST               5 (5)
             30 LOAD_CONST               6 (6)
             33 BUILD_LIST               3
             36 BUILD_LIST               3
             39 STORE_FAST               0 (a)

  6          42 LOAD_CONST               7 (<code object <dictcomp> at 0x7f9e6e6b94b0, file "/tmp/__pfw_test24_t3.py", line 6>)
             45 MAKE_FUNCTION            0
             48 LOAD_FAST                0 (a)
             51 GET_ITER            
             52 CALL_FUNCTION            1
             55 STORE_FAST               1 (b)

  7          58 LOAD_FAST                1 (b)
             61 RETURN_VALUE        

>>> 
```

更慢的方式：

```
def a():
    a = [[1, 2, 3], [2, 3, 4], [4, 5, 6]]
    b = {x[1]: x[2] for x in a}
    return b

>>> 
>>> timeit(a, number=1000000)
... 0.744184017181
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (1)
              3 LOAD_CONST               2 (2)
              6 LOAD_CONST               3 (3)
              9 BUILD_LIST               3
             12 LOAD_CONST               2 (2)
             15 LOAD_CONST               3 (3)
             18 LOAD_CONST               4 (4)
             21 BUILD_LIST               3
             24 LOAD_CONST               4 (4)
             27 LOAD_CONST               5 (5)
             30 LOAD_CONST               6 (6)
             33 BUILD_LIST               3
             36 BUILD_LIST               3
             39 STORE_FAST               0 (a)

  6          42 LOAD_CONST               7 (<code object <dictcomp> at 0x7fcc514754b0, file "/tmp/__pfw_test24_t1.py", line 6>)
             45 MAKE_FUNCTION            0
             48 LOAD_FAST                0 (a)
             51 GET_ITER            
             52 CALL_FUNCTION            1
             55 STORE_FAST               1 (b)

  7          58 LOAD_FAST                1 (b)
             61 RETURN_VALUE        

>>> 
        
```

最慢的方式：

```
def a():
    a = [[1, 2, 3], [2, 3, 4], [4, 5, 6]]
    b = dict((x, y)for w, x, y in a)
    return b

>>> 
>>> timeit(a, number=1000000)
... 1.32197403908
>>> 
>>> dis(a)
  5           0 LOAD_CONST               1 (1)
              3 LOAD_CONST               2 (2)
              6 LOAD_CONST               3 (3)
              9 BUILD_LIST               3
             12 LOAD_CONST               2 (2)
             15 LOAD_CONST               3 (3)
             18 LOAD_CONST               4 (4)
             21 BUILD_LIST               3
             24 LOAD_CONST               4 (4)
             27 LOAD_CONST               5 (5)
             30 LOAD_CONST               6 (6)
             33 BUILD_LIST               3
             36 BUILD_LIST               3
             39 STORE_FAST               0 (a)

  6          42 LOAD_GLOBAL              0 (dict)
             45 LOAD_CONST               7 (<code object <genexpr> at 0x7f5ea06174b0, file "/tmp/__pfw_test24_t2.py", line 6>)
             48 MAKE_FUNCTION            0
             51 LOAD_FAST                0 (a)
             54 GET_ITER            
             55 CALL_FUNCTION            1
             58 CALL_FUNCTION            1
             61 STORE_FAST               1 (b)

  7          64 LOAD_FAST                1 (b)
             67 RETURN_VALUE        

>>> 
    
```


## 测试用例 25

更快的方式：

```
def a(int=int, str=str, range=range):
    for i in range(500):
        int(str(i))


number = 100000

>>> 
>>> timeit(a, number=100000)
... 20.4009640217
>>> 
>>> dis(a)
  5           0 SETUP_LOOP              36 (to 39)
              3 LOAD_FAST                2 (range)
              6 LOAD_CONST               1 (500)
              9 CALL_FUNCTION            1
             12 GET_ITER            
        >>   13 FOR_ITER                22 (to 38)
             16 STORE_FAST               3 (i)

  6          19 LOAD_FAST                0 (int)
             22 LOAD_FAST                1 (str)
             25 LOAD_FAST                3 (i)
             28 CALL_FUNCTION            1
             31 CALL_FUNCTION            1
             34 POP_TOP             
             35 JUMP_ABSOLUTE           13
        >>   38 POP_BLOCK           
        >>   39 LOAD_CONST               0 (None)
             42 RETURN_VALUE        

>>> 
```

更慢的方式：

```
def a():
    for i in range(500):
        int(str(i))


number = 100000

>>> 
>>> timeit(a, number=100000)
... 21.7259840965
>>> 
>>> dis(a)
  5           0 SETUP_LOOP              36 (to 39)
              3 LOAD_GLOBAL              0 (range)
              6 LOAD_CONST               1 (500)
              9 CALL_FUNCTION            1
             12 GET_ITER            
        >>   13 FOR_ITER                22 (to 38)
             16 STORE_FAST               0 (i)

  6          19 LOAD_GLOBAL              1 (int)
             22 LOAD_GLOBAL              2 (str)
             25 LOAD_FAST                0 (i)
             28 CALL_FUNCTION            1
             31 CALL_FUNCTION            1
             34 POP_TOP             
             35 JUMP_ABSOLUTE           13
        >>   38 POP_BLOCK           
        >>   39 LOAD_CONST               0 (None)
             42 RETURN_VALUE        

>>> 
```



  [1]: http://pythonfasterway.uni.me/



