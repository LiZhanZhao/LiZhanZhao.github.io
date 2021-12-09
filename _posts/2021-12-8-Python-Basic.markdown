---
layout:     post
title:      "Python 基础总结"
subtitle:   ""
date:       2021-12-08 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Python
---



### 调用CMD窗口

- 命令窗口(Windows 使用 win+R 调出 cmd 运行框)



### 查看 Python 版本

- python --version



### Python 运行文件

- hello.py 

- ```python
#!/usr/bin/python3  
  
print("Hello, World!")
  ```

- python hello.py



### 集成开发环境 IDE

- PyCharm [下载地址](https://www.jetbrains.com/pycharm/download/)



###  *#!/usr/bin/python3*

- 脚本语言的第一行，目的就是指出，你想要你的这个文件中的代码用什么可执行程序去运行它，就这么简单。

  **#!/usr/bin/python3** 是告诉操作系统执行这个脚本的时候，调用 /usr/bin 下的 python3 解释器；

  **#!/usr/bin/env python3** 这种用法是为了防止操作系统用户没有将 python3 装在默认的 /usr/bin 路径里。当系统看到这一行的时候，首先会到 env 设置里查找 python3 的安装路径，再调用对应路径下的解释器程序完成操作。

  **#!/usr/bin/python3** 相当于写死了 **python3** 路径;

  **#!/usr/bin/env python3** 会去环境设置寻找 python3 目录，**推荐这种写法**。



### 注释

```python
#!/usr/bin/python3
 
# 第一个注释
# 第二个注释
 
'''
第三注释
第四注释
'''
 
"""
第五注释
第六注释
"""
print ("Hello, Python!")
```



### 行与缩进

- python最具特色的就是使用缩进来表示代码块，不需要使用大括号 **{}**，缩进的空格数是可变的，但是同一个代码块的语句必须包含相同的缩进空格数

```python
if True:
    print ("Answer")
    print ("True")
else:
    print ("Answer")
  print ("False")    # 缩进不一致，会导致运行错误
```



### import 与 from...import

- 在 python 用 **import** 或者 **from...import** 来导入相应的模块。

  将整个模块(somemodule)导入，格式为： **import somemodule**

  从某个模块中导入某个函数,格式为： **from somemodule import somefunction**

  从某个模块中导入多个函数,格式为： **from somemodule import firstfunc, secondfunc, thirdfunc**

  将某个模块中的全部函数导入，格式为： **from somemodule import \***



- ```
  import sys
  print('================Python import mode==========================')
  print ('命令行参数为:')
  for i in sys.argv:
      print (i)
  print ('\n python 路径为',sys.path)
  
  
  from sys import argv,path  #  导入特定的成员
   
  print('================python from import===================================')
  print('path:',path) # 因为已经导入path成员，所以此处引用时不需要加sys.path
  
  ```

  

### 赋值

- Python 中的变量不需要声明。每个变量在使用前都必须赋值，变量赋值以后该变量才会被创建。

  在 Python 中，变量就是变量，它没有类型，我们所说的"类型"是变量所指的内存中对象的类型。

  等号（=）用来给变量赋值。

  等号（=）运算符左边是一个变量名,等号（=）运算符右边是存储在变量中的值



- ```python
  #!/usr/bin/python3
  
  counter = 100          # 整型变量
  miles   = 1000.0       # 浮点型变量
  name    = "runoob"     # 字符串
  
  print (counter)
  print (miles)
  print (name)
  ```





### 标准数据类型

- Python3 中有六个标准的数据类型：

  - Number（数字）

  - String（字符串）

  - List（列表）

  - Tuple（元组）

  - Set（集合）

  - Dictionary（字典）

    

- Python3 的六个标准数据类型中：

  - **不可变数据（3 个）：**Number（数字）、String（字符串）、Tuple（元组）；

  - **可变数据（3 个）：**List（列表）、Dictionary（字典）、Set（集合）。



#### Number（数字）

- Python3 支持 **int、float、bool、complex（复数）**。

  在Python 3里，只有一种整数类型 int，表示为长整型，没有 python2 中的 **Long**。

- 数值运算

  - 1、Python可以同时为多个变量赋值，如a, b = 1, 2。
  - 2、一个变量可以通过赋值指向不同类型的对象。
  - 3、数值的除法包含两个运算符：**/** 返回一个浮点数，**//** 返回一个整数。
  - 4、在混合计算时，Python会把整型转换成为浮点数。
  - 2 ** 5 # 乘方
  - 2 / 4   # 除法，得到一个浮点数
  - 2 // 4  # 除法，得到一个整数



#### String（字符串）

- Python中的字符串用单引号 **'** 或双引号 **"** 括起来，同时使用反斜杠 \ 转义特殊字符。
- 如果你不想让反斜杠发生转义，可以在字符串前面添加一个 **r**，表示原始字符串
- 字符串的截取的语法格式如下：**变量[头下标:尾下标]**
- 索引值以 0 为开始值，-1 为从末尾的开始位置

- 加号 **+** 是字符串的连接符， 星号 ***** 表示复制当前字符串，与之结合的数字为复制的次数

- ```python
  #!/usr/bin/python3
  
  str = 'Runoob'
  
  print (str)          # 输出字符串
  print (str[0:-1])    # 输出第一个到倒数第二个的所有字符
  print (str[0])       # 输出字符串第一个字符
  print (str[2:5])     # 输出从第三个开始到第五个的字符
  print (str[2:])      # 输出从第三个开始的后的所有字符
  print (str * 2)      # 输出字符串两次，也可以写成 print (2 * str)
  print (str + "TEST") # 连接字符串
  
  输出：
  
  Runoob
  Runoo
  R
  noo
  noob
  RunoobRunoob
  RunoobTEST
  ```



#### List（列表）

- List（列表） 是 Python 中使用最频繁的数据类型。

  列表可以完成大多数集合类的数据结构实现。列表中元素的类型可以不相同，它支持数字，字符串甚至可以包含列表（所谓嵌套）。

  列表是写在方括号 **[]** 之间、用逗号分隔开的元素列表。

  和字符串一样，列表同样可以被索引和截取，列表被截取后返回一个包含所需元素的新列表。



- 列表截取的语法格式如下：索引值以 **0** 为开始值，**-1** 为从末尾的开始位置

  ```
  变量[头下标:尾下标]
  ```

  

- ```python
  #!/usr/bin/python3
  
  list = [ 'abcd', 786 , 2.23, 'runoob', 70.2 ]
  tinylist = [123, 'runoob']
  
  print (list)            # 输出完整列表
  print (list[0])         # 输出列表第一个元素
  print (list[1:3])       # 从第二个开始输出到第三个元素
  print (list[2:])        # 输出从第三个元素开始的所有元素
  print (tinylist * 2)    # 输出两次列表
  print (list + tinylist) # 连接列表
  
  
  ['abcd', 786, 2.23, 'runoob', 70.2]
  abcd
  [786, 2.23]
  [2.23, 'runoob', 70.2]
  [123, 'runoob', 123, 'runoob']
  ['abcd', 786, 2.23, 'runoob', 70.2, 123, 'runoob']
  ```



- 与Python字符串不一样的是，列表中的元素是可以改变的：
- 1、List写在方括号之间，元素用逗号隔开。
- 2、和字符串一样，list可以被索引和切片。
- 3、List可以使用+操作符进行拼接。
- 4、List中的元素是可以改变的。

- Python 列表截取可以接收第三个参数，参数作用是截取的步长，以下实例在索引 1 到索引 4 的位置并设置为步长为 2（间隔一个位置）来截取字符串

- ```
  letters = ['r', 'u', 'n', 'o', 'o', 'b']
  letters[1:4:2]
  ['u','o']
  ```



#### Tuple（元组）

- 元组（tuple）与列表类似，不同之处在于元组的元素不能修改。元组写在小括号 **()** 里，元素之间用逗号隔开，元组中的元素类型也可以不相同：

- 元组与字符串类似，可以被索引且下标索引从0开始，-1 为从末尾开始的位置。

  

- ```python
  #!/usr/bin/python3
  
  tuple = ( 'abcd', 786 , 2.23, 'runoob', 70.2  )
  tinytuple = (123, 'runoob')
  
  print (tuple)             # 输出完整元组
  print (tuple[0])          # 输出元组的第一个元素
  print (tuple[1:3])        # 输出从第二个元素开始到第三个元素
  print (tuple[2:])         # 输出从第三个元素开始的所有元素
  print (tinytuple * 2)     # 输出两次元组
  print (tuple + tinytuple) # 连接元组
  
  输出：
  ('abcd', 786, 2.23, 'runoob', 70.2)
  abcd
  (786, 2.23)
  (2.23, 'runoob', 70.2)
  (123, 'runoob', 123, 'runoob')
  ('abcd', 786, 2.23, 'runoob', 70.2, 123, 'runoob')
  ```

  

- ```python
  >>> tup = (1, 2, 3, 4, 5, 6)
  >>> print(tup[0])
  1
  >>> print(tup[1:5])
  (2, 3, 4, 5)
  >>> tup[0] = 11  # 修改元组元素的操作是非法的
  Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
  TypeError: 'tuple' object does not support item assignment
  >>>
  ```

  

- 虽然tuple的元素 **不可改变**，但它可以包含可变的对象，比如list列表。

- 构造包含 0 个或 1 个元素的元组比较特殊，所以有一些额外的语法规则：

- ```python
  tup1 = ()    # 空元组
  tup2 = (20,) # 一个元素，需要在元素后添加逗号
  ```





#### Set（集合）

- 可以使用大括号 **{ }** 或者 **set()** 函数创建集合，注意：创建一个空集合必须用 **set()** 而不是 **{ }**，因为 **{ }** 是用来创建一个空字典。

- ```python
  #!/usr/bin/python3
  
  sites = {1, 'Taobao', 'Runoob', 'Facebook', 'Zhihu', 'Baidu'}
  
  print(sites)   # 输出集合，重复的元素被自动去掉
  
  # 成员测试
  if 1 in sites :
      print('1 在集合中')
  else :
      print('1 不在集合中')
  
  
  # set可以进行集合运算
  a = set('abracadabra')
  b = set('alacazam')
  
  print(a)
  
  print(a - b)     # a 和 b 的差集
  
  print(a | b)     # a 和 b 的并集
  
  print(a & b)     # a 和 b 的交集
  
  print(a ^ b)     # a 和 b 中不同时存在的元素
  ```

  

#### Dictionary（字典）

- 字典（dictionary）是Python中另一个非常有用的内置数据类型。

  列表是有序的对象集合，字典是无序的对象集合。两者之间的区别在于：字典当中的元素是通过键来存取的，而不是通过偏移存取。

  字典是一种映射类型，字典用 **{ }** 标识，它是一个无序的 **键(key) : 值(value)** 的集合。

  键(key)必须使用不可变类型。

  在同一个字典中，键(key)必须是唯一的。

  

- ```python
  #!/usr/bin/python3
  
  dict = {}
  dict['one'] = "1 - aaaa"
  dict[2]     = "2 - bbbb"
  
  tinydict = {'name': 'runoob','code':1, 'site': 'www.runoob.com'}
  
  
  print (dict['one'])       # 输出键为 'one' 的值
  print (dict[2])           # 输出键为 2 的值
  print (tinydict)          # 输出完整的字典
  print (tinydict.keys())   # 输出所有键
  print (tinydict.values()) # 输出所有值
  
  输出
  1 - aaaa
  2 - bbbb
  {'name': 'runoob', 'code': 1, 'site': 'www.runoob.com'}
  dict_keys(['name', 'code', 'site'])
  dict_values(['runoob', 1, 'www.runoob.com'])
  ```



- 构造函数 dict() 可以直接从键值对序列中构建字典如下：

- ```python
  dict([('Runoob', 1), ('Google', 2), ('Taobao', 3)])
  {'Runoob': 1, 'Google': 2, 'Taobao': 3}
  
  {x: x**2 for x in (2, 4, 6)}
  {2: 4, 4: 16, 6: 36}
  
  dict(Runoob=1, Google=2, Taobao=3)
  {'Runoob': 1, 'Google': 2, 'Taobao': 3}
  ```

  

- 1、字典是一种映射类型，它的元素是键值对。

  

- 2、字典的关键字必须为不可变类型，且不能重复。

  

- 3、创建空字典使用 **{ }**。



### 数据类型转换

|                             函数                             | 描述                                                |
| :----------------------------------------------------------: | :-------------------------------------------------- |
| [int(x [,base\])](https://www.runoob.com/python3/python-func-int.html) | 将x转换为一个整数                                   |
| [float(x)](https://www.runoob.com/python3/python-func-float.html) | 将x转换到一个浮点数                                 |
| [complex(real [,imag\])](https://www.runoob.com/python3/python-func-complex.html) | 创建一个复数                                        |
| [str(x)](https://www.runoob.com/python3/python-func-str.html) | 将对象 x 转换为字符串                               |
| [repr(x)](https://www.runoob.com/python3/python-func-repr.html) | 将对象 x 转换为表达式字符串                         |
| [eval(str)](https://www.runoob.com/python3/python-func-eval.html) | 用来计算在字符串中的有效Python表达式,并返回一个对象 |
| [tuple(s)](https://www.runoob.com/python3/python3-func-tuple.html) | 将序列 s 转换为一个元组                             |
| [list(s)](https://www.runoob.com/python3/python3-att-list-list.html) | 将序列 s 转换为一个列表                             |
| [set(s)](https://www.runoob.com/python3/python-func-set.html) | 转换为可变集合                                      |
| [dict(d)](https://www.runoob.com/python3/python-func-dict.html) | 创建一个字典。d 必须是一个 (key, value)元组序列。   |
| [frozenset(s)](https://www.runoob.com/python3/python-func-frozenset.html) | 转换为不可变集合                                    |
| [chr(x)](https://www.runoob.com/python3/python-func-chr.html) | 将一个整数转换为一个字符                            |
| [ord(x)](https://www.runoob.com/python3/python-func-ord.html) | 将一个字符转换为它的整数值                          |
| [hex(x)](https://www.runoob.com/python3/python-func-hex.html) | 将一个整数转换为一个十六进制字符串                  |
| [oct(x)](https://www.runoob.com/python3/python-func-oct.html) | 将一个整数转换为一个八进制字符串                    |
