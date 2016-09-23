title: python 3.5 模块
date: 2016/8/9 10:07:12  
categories: python
tags:
	- 学习笔记

---

当时学的是2.x版本，有点过时了，现在看了[廖雪峰的博客](http://www.liaoxuefeng.com/)的3.5版本的语法介绍，做一些摘录。

<!--more-->





Python又引入了按目录来组织模块的方法，称为包（Package）。引入了包以后，只要顶层的包名不与别人冲突，那所有模块都不会与别人冲突。
请注意，每一个包目录下面都会有一个__init\_\_.py的文件，这个文件是必须存在的，否则，Python就把这个目录当成普通目录，而不是一个包。__init\_\_.py可以是空文件，也可以有Python代码，因为__init__.py本身就是一个模块

### 使用模块
```python
import sys
def test():
	pass
if __name__=='__main__':
	test()
```
导入sys模块后,就有了变量sys指向该模块，利用sys这个变量，就可以访问sys模块的所有功能。
sys模块有一个argv变量，用list存储了**命令行的所有参数**。argv至少有一个元素，因为第一个参数永远是该.py文件的名称。
在命令行运行该模块文件时，Python解释器把一个特殊变量__name__置为__main__，而如果在其他地方导入该模块时，if判断将失败，因此，这种if测试可以让一个模块通过命令行运行时执行一些额外的代码，最常见的就是运行测试。

作用域：
- 正常的函数和变量名是公开的（public），可以被直接引用
- 类似__xxx__这样的变量是特殊变量，可以被直接引用，但是有特殊用途。
- 类似_xxx和__xxx这样的函数或变量就是非公开的（private），不应该被直接引用。

### 安装第三方模块
使用pip3 install XXX 安装第三方库
当我们试图加载一个模块时，Python会在指定的路径下搜索对应的.py文件，如果找不到，就会报错。
搜索路径存放在sys模块的path变量中
```python
import sys
print(sys.path)
```
当要添加自己的搜索目录时可以
1. 直接修改sys.path:**sys.path.append('/Users/xxx/xxx')**
2. 设置环境变量PYTHONPATH


最后再次感谢廖老师的辛勤劳动。
