---
layout: post
title:  "python中优雅打印protobuf"
author: kevin
categories: python
tags: python protobuf text-format
---
* content
{:toc}

### 背景
项目中使用了python protobuf来进行协议交互，发现在打日志时使用默认的方式会有换行，这在容器内采集日志到ELK会很恶心。

查询pb手册并没有特别**简单**的打印类似c++中的ShortDebugString()的方法，只能自己处理了。

可以想象到肉眼识别pb msg对象一个个修改是很痛苦的，那么有没有不用修改一行代码即可达成目标的方案呢？
**结论是有的**，过程有点小曲折。

### 目标
* 希望以类似C++的ShortDebugString()形式打印，去掉其newline等，便于日志采集。

### 现状
python库中提供了两种方式：
* 使用`json`格式。
> `json_format.MessageToJson(pb_msg))`

* 使用`text_format`中函数
> `text_format.MessageToString(pb_msg, as_one_line=True)`

然而这两者都会让代码写起来不美观。而pb自带的`__str__`方法使用的是`text_format.MessageToString(pb_msg)`是不指定as_one_line的，又没办法去参数化定制，怎么办？


### 方案

#### 尝试一
直接修改源码，从源头上改变__str__的行为。这感觉入侵过深了，不过不妨一试。
阅读源码发现python的__str__方法是通过反射由descriptor在python_message.py中注入的。具体代码如下：
```cpp
def _AddStrMethod(message_descriptor, cls):
  """Helper for _AddMessageMethods()."""
  def __str__(self):
    return text_format.MessageToString(self)
  cls.__str__ = __str__


def _AddReprMethod(message_descriptor, cls):
  """Helper for _AddMessageMethods()."""
  def __repr__(self):
    return text_format.MessageToString(self)
  cls.__repr__ = __repr__

```
想象中将其修改为`return text_format.MessageToString(self, as_one_line=True)`即可。

现实是，修改后并不生效！原因是，pip安装的protobuf的实现是cpp版本的，具体可查看：
```python
>>> from google.protobuf.internal import api_implementation
>>> print api_implementation.Type()
cpp
>>> print api_implementation.Version()
2
```
故修改python_message.py无效。

继续看到在`reflection.py`中有判断Type()来决定是使用cpp还是python的message实现，是否可以hack修改这里呢？
```cpp
if api_implementation.Type() != 'cpp':
    from google.protobuf.pyext import cpp_message as message_impl
else:
    from google.protobuf.internal import python_message as message_imp
```

结论依然不行：（可能实现有缺陷）
>  File "/Library/Python/2.7/site-packages/google/protobuf/internal/python_message.py", line 162, in __init__
    _AttachFieldHelpers(cls, field)
  File "/Library/Python/2.7/site-packages/google/protobuf/internal/python_message.py", line 303, in _AttachFieldHelpers
    field_descriptor._encoder = field_encoder
AttributeError: 'google.protobuf.pyext._message.FieldDescriptor' object has no attribute '_encoder'

**结论：方案一失败**

#### 尝试二
不能从源头替换，那么从最终Message中替换呢？我们知道pb的反射肯定会有个地方存储各Message类型信息。阅读python-pb代码后发现主要在文件`symbol_database.py`中。

并没有提供一个可迭代的遍历message的方法，只能强行访问private来搞了：`symbol_database.Default()._classes`

```python
def pbShortDebugStr(self):
    return text_format.MessageToString(self, as_one_line=True)

def ReplacePbStr():
    for k in symbol_database.Default()._classes.keys():
        v = symbol_database.Default()._classes[k]
        v.__repr__ = pbShortDebugStr
        v.__str__ = pbShortDebugStr

```

现在可以愉快的使用如下来打印pb结构了。
```python
print("pb_msg:", pb_msg)
```

然而这并不完美，遗留一个潜在风险：这里既然是替换db中的message set，它的Register是有先后或者说可动态的，而我们执行此函数的时机应该在什么时候呢？否则必有漏网之鱼？

**结论： 方案二有风险**

#### 尝试三
既然都这样了，何不直接修改注册Message的地方，具体位置如下(symbol_database.py)：
```python
class SymbolDatabase(message_factory.MessageFactory):
    """A database of Python generated symbols."""

    def RegisterMessage(self, message):
        """Registers the given message type in the local database.

        Calls to GetSymbol() and GetMessages() will return messages registered here.

        Args:
          message: a message.Message, to be registered.

        Returns:
          The provided message.
        """

        def pbShortDebugStr(self): #新增
            return text_format.MessageToString(self, as_one_line=True) #新增
        message.__str__ = pbShortDebugStr   #新增
        message.__repr__ = pbShortDebugStr  # 新增

        desc = message.DESCRIPTOR
        self._classes[desc] = message
        self.RegisterMessageDescriptor(desc)
        return message
```
因为库是由pip安装的，故我们写个脚本去追加相关代码（上面标注新增的行）。

```bash
#!/bin/bash

DIST_DIR=`pip show protobuf  | grep Location | awk '{print $2}'`
DST_FILE=`find $DIST_DIR -type f -name "symbol_database.py"`

#import
sed -i '/from google.protobuf import message_factory/a from google.protobuf import text_format' $DST_FILE

#因为python的对齐，这里以desc=message.DESCRIPTOR为参照，以它的缩进方式去对齐代码
sed -i 's/\(.*\)\(desc = message.DESCRIPTOR\)/\1def pbShortDebugStr(self):\n\1    return text_format.MessageToString(self, as_one_line=True)\n\n\1message.__str__ = pbShortDebugStr\n\1message.__repr__ = pbShortDebugStr\n\1\2/' $DST_FILE

```

将脚本集成至dockerfile中，在进入ENTRYPOINT之前先RUN即可。

这样，虽然不算完美无公害，也算暂时较好的解决了问题。

如果有其它思路欢迎探讨！
