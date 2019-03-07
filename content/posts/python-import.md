---
title: "Python Import"
date: 2019-03-07T20:25:41+08:00
draft: false
---

## 1. 模块搜索

搜索的目标：

- 名为 `<module-name>.py` 的文件
- 或名为 `<module-name>` 的文件夹

在下面的路径列表中进行搜索：

1. `sys.builtin_module_names`
2. `sys.path`

注意内建模块与标准库是有区别的，内建模块直接在编译时链接在解释器中，使用 `sys.builtin_module_names` 表示，优先搜索。

另一些标准库模块在 `sys.path` 包含的目录中。因此可以在当前目录创建同名模块，覆盖掉这些非内建标准库模块。

## `sys.path`

`sys.path`在解释器启动时被初始化，具体内容与解释器的启动方式有关：

- The directory containing the input script (or the current directory when no file is specified).
- `PYTHONPATH` (a list of directory names, with the same syntax as the shell variable PATH).
- The installation-dependent default.

在启动解释器时:

- 如果第一个参数是 python 脚本的路径，则脚本所在目录会被加入到 `sys.path` 的开头，并且脚本被当作 main module 执行。
- 如果未指定脚本路径启动解释器，则 `sys.path` 第一个元素是一个空字符串，在遇到空字符串时解释器会在当前工作目录搜索模块。
- 如果使用 `python -m module-name` 启动解释器，则当前工作目录会被添加到 `sys.path` 的开头，并且 `module-name` 所指向的模块被作为 `__main__` 模块执行。

执行示例：
```
# sl @ r3 in ~/slnyanyanya/python-import/test [10:55:44]
$ python start.py
['/Users/sl/slnyanyanya/python-import/test',
'/Users/sl/.pyenv/versions/3.7.0/lib/python37.zip',
'/Users/sl/.pyenv/versions/3.7.0/lib/python3.7',
'/Users/sl/.pyenv/versions/3.7.0/lib/python3.7/lib-dynload',
'/Users/sl/.pyenv/versions/3.7.0/lib/python3.7/site-packages']

# sl @ r3 in ~/slnyanyanya/python-import/test [10:55:47]
$ python
Python 3.7.0 (default, Sep 22 2018, 01:49:18)
[Clang 9.1.0 (clang-902.0.39.2)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import sys
>>> sys.path
['', '/Users/sl/.pyenv/versions/3.7.0/lib/python37.zip',
'/Users/sl/.pyenv/versions/3.7.0/lib/python3.7', 
'/Users/sl/.pyenv/versions/3.7.0/lib/python3.7/lib-dynload', 
'/Users/sl/.pyenv/versions/3.7.0/lib/python3.7/site-packages']
```

## 2. 关于 `sys.path` 需要注意的点

目录结构如下：

```
$ tree .
.
├── other.py
├── packA
│   ├── a1.py
│   ├── a2.py
│   └── subA
│       ├── sa1.py
│       └── sa2.py
├── packB
│   ├── __init__.py
│   ├── b1.py
│   └── b2.py
└── start.py
```

**1. 当执行一个脚本时，如果不使用 `python -m`，那么 import 系统不在乎「当前工作目录」，只在乎被执行脚本所在的目录。**

执行示例：

```
# sl @ r3 in ~/slnyanyanya/python-import/test [11:18:28]
$ python packA/subA/sa1.py
['/Users/sl/slnyanyanya/python-import/test/packA/subA', 
'/Users/sl/.pyenv/versions/3.7.0/lib/python37.zip', 
'/Users/sl/.pyenv/versions/3.7.0/lib/python3.7', 
'/Users/sl/.pyenv/versions/3.7.0/lib/python3.7/lib-dynload', 
'/Users/sl/.pyenv/versions/3.7.0/lib/python3.7/site-packages']
```

在 test 目录下执行 test/packA/subA/sa1.py，`sys.path` 第一项是 sa1.py 所在的路径。

**2. 在解释器启动之后，`sys.path` 可以被所有模块共享。**

如果我们使用 `python start.py` 运行项目根目录的脚本，那么其他所有 package 中的模块，在 import 其他模块时，可视范围都和 start.py 相同。

例如我们可以在 `packA/subA/sa2.py` 中直接 import `other.py` 中的变量。

```
# other.py
name = 'gg'

# packA/subA/sa1.py
from other import name
cap_name = name.upper()

# start.py
from packA.subA.sa2 import cap_name
print(cap_name)
```

无论我们在哪个工作目录执行 `start.py`，都能够成功输出 `'GG'`。

## 3. package

在 python3 中，每个文件夹都被视为 package，无论其中是否含有 `__init__.py`。

当一个 package 首次被 import 时，解释器尝试执行文件夹中的 `__init__.py`，如果该文件存在的话。并且 `__init__.py` 中的 name 位于 package 的 namespace 中。

需要注意的是：

**1. 当解释器执行某个 python 脚本时（无论是否使用 `python -m`），该脚本所在的目录并不被视为一个 package，因此不会执行同目录下的 `__init__.py`。**

例如：

```
# packB/__init__.py
print('hallo from packB/__init__')

# packB/b1.py
print('hallo from b1')

# other.py
import packB
name = 'gg'
```

执行 `b1.py`，输出:

```
# sl @ r3 in ~/slnyanyanya/python-import/test [11:41:03]
$ python packB/b1.py
hello from b1
```

`packB/__init__.py` 中的代码并没有被调用，因为我们直接执行 b1.py。

执行 `other.py`，输出：

```
# sl @ r3 in ~/slnyanyanya/python-import/test [11:45:55]
$ python other.py
__init__ of packB!
```

`packB/__init__.py` 中的代码被调用，因为我们 import 了 packB。

**2. 解释器在执行某个脚本时，该脚本被视为 top-level module，无法从 top-level module 的 parent directory 进行相对导入。**

因为 `sys.path` 只会把被执行脚本所在目录加入 `sys.path`，无法访问被执行脚本所在目录父目录的内容。

如果想要在被执行脚本中 import 父目录的内容，只能修改 `PYTHONPATH` 或者 `sys.path`。

## 4. 实践

由于 `sys.path` 的第一项由被执行脚本所在位置决定，因此 import 系统搜索路径与__被 import 模块和被执行脚本之间的位置关系__有关。

如果我们的项目是作为一整个 project 来运行，能够确定每次执行的是哪个脚本，那么被 import 模块与实际执行脚本之间的位置关系就是固定不变的。
例如我们的可执行脚本在项目根目录（如上面的 start.py），那么我们在写 `import` 语句时就能默认项目根目录位于 `sys.path` 之中，使用 `from packA.subA import sa1` 这样的语句进行 import。

如果我们的项目是作为一个 package 用于被他人 import，那么我们就需要注意我们的项目与他人代码之间的位置关系。如果位置不对，我们写好的 import 语句就会失效。

- 最好的情况是 package 文件夹被安装在常用目录，比如 site-packages 中。（通过 `pip install`，这也是编写 package 推荐的做法）假设 package 名字为 X，此时 package X 内部的 `import` 语句就可以写成 `from X.packA import a1`。
- package X 作为一个文件夹放在与被执行脚本相同的目录中。此时被执行脚本可以「直接看见」 package X 文件夹。package X 内部的 `import` 语句和上面一样，也可以写成 `from X.packA import a1`。
- 最麻烦的情况是：package X 不在常用目录，和可执行脚本也不在同一目录。此时需要修改 `PYTHONPATH` 或者 `sys.path`。

最后一种情况的例子：

[The Hitchhiker's Guide to Python](https://docs.python-guide.org/writing/structure/#test-suite)  推荐的 project layout：

```
README.rst
LICENSE
setup.py
requirements.txt
sample/__init__.py
sample/core.py
sample/helpers.py
docs/conf.py
docs/index.rst
tests/test_basic.py
tests/test_advanced.py
```

测试脚本是可执行脚本，但是位于 `tests` 文件夹。
推荐的解决方案是：在测试脚本中显式地修改 `sys.path`。

## 5. conclusion

综上所述，我们在编写 import 语句时，最好从整个 package 的根目录开始 import。

例如第 2 节提到的目录结构中，如果packB/b1.py 要引用 packA/subA/sa2.py，就应该写成:

```
from packA.subA.sa2 import name
```

在测试时，在测试脚本中手动修改 `sys.path`，把 package 根目录的路径加入其中，防止模块搜索不到。

Reference: [The Definitive Guide to Python import Statements](https://chrisyeh96.github.io/2017/08/08/definitive-guide-python-imports.html)