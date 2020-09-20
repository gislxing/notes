# Python 基础学习笔记

## 1. 初识Python

### 安装Python

先确认电脑上的`python`版本为`python3`，打开命令行窗口，并输入`python -V`，显示`Python 3.x.x`就可以，如果没有则下载`python3`的最新版本安装（安装时记得将python添加到环境变量中）

进制

### 创建简单的Python列表

可以按照以下的方式创建Python列表：

```python
movies = ["aa", "bb", "cc"， 1537]
```

**Python 的变量标示符没有类型，还可以包含任何类型的数据**

**Python 中的单引号和双引号表示相同的作用**

可以使用标准的中括号偏移量来访问列表中的数据项

```python
print(movies[0])
```

使用以下列表方法可操作列表：

```python
//给列表的末尾追加一个元素
movies.append("dd")
//给列表末尾追加多个元素
movies.extend(["ff", "tt"])

//删除列表末尾的元素
movies.pop()

//删除列表0号位元素
movies.pop(0)

//删除元素"bb"
movies.remove("bb")

//在列表1号位置之前插入一个元素
movies.insert(1, "ii")
```

### for 循环

Python使用for循环迭代处理列表

```python
for tmp in movies:
	print(tmp)
```

### 数据类型判断

Python 中使用`instance()`判断一个变量是否指定的类型

```python
movies = ["aa", "bb", "cc"， 1537]
//返回true
isinstance(movies, list)
```

### 创建函数

Python 中定义函数的一般格式如下

```python
def 函数名称(参数列表):
	函数代码处理逻辑
```

## 2. 发布代码

单行注释以`#`开始

多行注释以三个引号（单引号或者双引号）包围

1. 创建一个`.py`文件，并将该文件放到 一个文件夹中，该文件存放你要发布的代码

2. 在该文件夹创建`setup.py`文件，存放发布用到的元数据

   ```python
   from distutils.core import setup
   
   setup(
       name = "netster",
       version = "1.0.0",
       py_modules = ["nester"],
       author = "zbs",
       author_email = "zbs@qq.com",
       url = "",
       description = "A simple demo"
       )
   ```

3. 在该文件中打开一个命令行窗口构建发布文件，输入命令`python setup.py sdist`

4. 将发布模块安装到Python本地副本中，输入命令`python setup.py install`

5. 发布成功后，在代码中使用`import netster`即可使用刚才发布的代码

6. 使用时必须加上命令空间，Python中默认的命名空间与模块名称同名

### 迭代器

Python 内置函数`range(int)`可生成一个迭代器并返回一个在指定范围内的数字

```python
# 输出数字1-9
for i in range(10):
	print(i)
```

### 可选参数

Python 支持可选参数，如果需要将一个参数变为可选的，只需要给该参数赋一个缺省值即可

`print("\t", end="")`输出制表符（TAB）但是不换行

```python
# 该方法的第二个参数为可选
def print_l(data, level = 0):
	for i in range(level):
		print("\t", end="")
	print(data)
```

## 3. 文件与异常

Python 中使用`try`来处理运行时异常

```python
try:
	# 业务逻辑
except IOError as e:
	# pass是忽略这个错误
	pass
finally:
    # 必须执行的代码
```

使用`open()`BIF函数打开磁盘上的文件，返回一个迭代器从文件读取数据，一次读取一行

`readline()`方法一次读取一行数据

`seek()`方法可是将文件回退到开始的位置

`close()`关闭一个打开的文件

`split()`拆分字符串

Python中不可改变的常量列表称为`元组`，一旦将数据赋值到元组则不能再改变

`help()`可以在IDLE中查看文档

`find()`在一个字符串中查找子串

`not`关键字将一个条件取反

`try/except/finally`异常处理

`pass`是空语句什么也不做

## 4. 持久存储

`open()`打开文件时默认以只读（r）模式打开，如果要以其他模式打开，则需要指定

```python
# 以可写模式打开文件
data = open("test.txt", "w")
# 将aaa写入文件,这个将覆盖原文件中的内容，如果需要追加以a的方式打开文件
print("aaa", file=data)
```

`xx.strip()`去除字符串两边的空字符串

`print()`内置函数的file参数控制将数据输出到哪里

`finally`组总会执行

回向`except`组传入一个异常对象，并使用`as`关键字赋值到一个变量

`str()`内置函数可以用来将任何对象转换为字符串

`locals()`内置函数返回当前作用域中的变量集合

`in`操作符用于检查成员关系

`+`操作符用于字符串时将连接两个字符串，拥有数字时则会相加

`with`语句会自动处理所有已打开文件的关闭工作，即使出现异常

```python
with open("xx.txt") as data:
	print(str(data))
```

`sys.stdout`是Python中的标准输出

标准库的`pickle`模块可以高效的将Python数据对象以二进制的形式保存到磁盘文件或者从磁盘读取

`pickle.dump()`函数将数据保存到磁盘

`pickle.load()`函数从磁盘读取数据

## 5. 处理数据

`sort()`方法可以在原地修改列表的顺序（默认升序排列）

`sorted()`内置函数复制排序，返回一个排序号的列表（默认升序排列）

向`sort()`或者`sorted()`传入`reverse=True`可以降序排列数据

列表推导：

```python
# 如果有以下结构的代码则可以使用列表推导重写
new_l = []
for tmp in old_l:
	new_l.append(len(tmp))

# 使用列表推导重写上述代码如下
new_l = [len(tmp) for tmp in old_l]
```

访问一个列表中的多个数据项可以使用切片

```python
# 返回列表中前3项数据
my_list[0:3]
```

`set()`方法可以创建一个集合，集合是一个无序且不重复的列表

## 6. 定制数据对象

使用`dict()`或者`{}`可以创建一个空字典

要访问一个名为`person`字典中与键`name`关联的值，可以使用中括号记法`person["name"]`

Python字典会随着数据的增加而动态扩大

可以先创建一个空字典`person = {}`或者`person = dict()`，然后再增加数据`person["name"] = "perl"`

`class`关键字定义一个类

可以在类中定义`_init_()`方法来初始化对象

类中定义的所有方法的第一个参数都必须是`self`，从而将数据与对象关联

使用类中的每个属性时，其前面必须有`self`：`self.name`

类可以从零开始出创建，也可以从Python的内置类或者自定义类继承

## 7. web开发

cgi模块的`fieldStorage()`方法可以从cgi脚本访问发送到web服务器的数据

标准`os`库的`environ`字典可以获得程序的环境设置

`SQLite`数据库作为`sqlite3`标准库包含在python中

`connect()`方法可以建立与数据库的一个连接

`cursor()`方法可以一个已有连接与数据库交互

`execute()`可以通过一个已有连接向数据库发送一个`SQL`查询

`commit()`方法提交修改

`rollback()`方法取消对数据库的修改

`close()`关闭与数据库的连接

## 8. 扩展web应用









