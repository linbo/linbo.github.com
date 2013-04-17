---
layout: post
title: "[翻译] 理解Python的With语句"
description: ""
category: 翻译
tags: 
    - python
    - 工
---
{% include JB/setup %}

缘由：
---

[张教主登高一呼](http://zuroc.42qu.com/11136091 "资助翻译")，众壮士纷纷揭竿而起，群之响应，不过一周尔尔，[**周刊**](http://python.42qu.com/ "周刊")第一期已新鲜出炉。

好玩，还有钱拿，土鳖我发现竟然没有支付宝账号，马上行动，遂得此一拙作，得稿费若钱。



With语句是什么？
----------

Python’s with statement provides a very convenient way of dealing with the situation where you have to do a setup and teardown to make something happen. A very good example for this is the situation where you want to gain a handler to a file, read data from the file and the close the file handler.

有一些任务，可能事先需要设置，事后做清理工作。对于这种场景，Python的with语句提供了一种非常方便的处理方式。一个很好的例子是文件处理，你需要获取一个文件句柄，从文件中读取数据，然后关闭文件句柄。

Without the with statement, one would write something along the lines of:

如果不用with语句，代码如下：

```python
file = open("/tmp/foo.txt")
data = file.read()
file.close()
```

There are two annoying things here. First, you end up forgetting to close the file handler. The second is how to handle exceptions that may occur once the file handler has been obtained. One could write something like this to get around this:

这里有两个问题。一是可能忘记关闭文件句柄；二是文件读取数据发生异常，没有进行任何处理。下面是处理异常的加强版本：

```python
file = open("/tmp/foo.txt")
try:
    data = file.read()
finally:
    file.close()
```

While this works well, it is unnecessarily verbose. This is where with is useful. The good thing about with apart from the better syntax is that it is very good handling exceptions. The above code would look like this, when using with:

虽然这段代码运行良好，但是太冗长了。这时候就是with一展身手的时候了。除了有更优雅的语法，with还可以很好的处理上下文环境产生的异常。下面是with版本的代码：

```python
with open("/tmp /foo.txt") as file:
    data = file.read()
```

with如何工作？
------ ---

while this might look like magic, the way Python handles with is more clever than magic. The basic idea is that the statement after with has to evaluate an object that responds to an \_\_enter\_\_() as well as an \_\_exit\_\_() function.

这看起来充满魔法，但不仅仅是魔法，Python对with的处理还很聪明。基本思想是with所求值的对象必须有一个\_\_enter\_\_()方法，一个\_\_exit\_\_()方法。

After the statement that follows with is evaluated, the \_\_enter\_\_() function on the resulting object is called. The value returned by this function is assigned to the variable following as. After every statement in the block is evaluated, the \_\_exit\_\_() function is called.

紧跟with后面的语句被求值后，返回对象的\_\_enter\_\_()方法被调用，这个方法的返回值将被赋值给as后面的变量。当with后面的代码块全部被执行完之后，将调用前面返回对象的\_\_exit\_\_()方法。

This can be demonstrated with the following example:

下面例子可以具体说明with如何工作：

```python
#!/usr/bin/env python
# with_example01.py

class Sample:
    def __enter__(self):
        print "In __enter__()"
        return "Foo"

    def __exit__(self, type, value, trace):
        print "In __exit__()"

            
def get_sample():
    return Sample()


with get_sample() as sample:
    print "sample:", sample
```

When executed, this will result in:  
行代码，输出如下

```bash
bash-3.2$ ./with_example01.py
In __enter__()
sample: Foo
In __exit__()
```

As you can see,
The \_\_enter\_\_() function is executed
The value returned by it - in this case "Foo" is assigned to sample
The body of the block is executed, thereby printing the value of sample ie. "Foo"
The \_\_exit\_\_() function is called.
What makes with really powerful is the fact that it can handle exceptions. You would have noticed that the \_\_exit\_\_() function for Sample takes three arguments - val, type and trace. These are useful in exception handling. Let’s see how this works by modifying the above example.

正如你看到的，

1. \_\_enter\_\_()方法被执行
1. \_\_enter\_\_()方法返回的值 - 这个例子中是"Foo"，赋值给变量'sample'
1. 执行代码块，打印变量"sample"的值为 "Foo"
1. \_\_exit\_\_()方法被调用

with真正强大之处是它可以处理异常。可能你已经注意到Sample类的\_\_exit\_\_方法有三个参数- val, type 和 trace。 这些参数在异常处理中相当有用。我们来改一下代码，看看具体如何工作的。

```python
#!/usr/bin/env python
# with_example02.py


class Sample:
    def __enter__(self):
        return self

    def __exit__(self, type, value, trace):
        print "type:", type
        print "value:", value
        print "trace:", trace
        
    def do_something(self):
        bar = 1/0
        return bar + 10

with Sample() as sample:
    sample.do_something()
```

Notice how in this example, instead of get\_sample(), with takes Sample(). It does not matter, as long as the statement that follows with evaluates to an object that has an \_\_enter\_\_() and \_\_exit\_\_() functions. In this case, Sample()’s \_\_enter\_\_() returns the newly created instance of Sample and that is what gets passed to sample.

这个例子中，with后面的get\_sample()变成了Sample()。这没有任何关系，只要紧跟with后面的语句所返回的对象有 \_\_enter\_\_()和\_\_exit\_\_()方法即可。此例中，Sample()的\_\_enter\_\_()方法返回新创建的Sample对象，并赋值给变量sample。

When executed:

代码执行后：

```bash
bash-3.2$ ./with_example02.py
type: <type 'exceptions.ZeroDivisionError'>
value: integer division or modulo by zero
trace: <traceback object at 0x1004a8128>
Traceback (most recent call last):
  File "./with_example02.py", line 19, in <module>
    sample.do_somet hing()
  File "./with_example02.py", line 15, in do_something
    bar = 1/0
ZeroDivisionError: integer division or modulo by zero
```

Essentially, if there are exceptions being thrown from anywhere inside the block, the \_\_exit\_\_() function for the object is called. As you can see, the type, value and the stack trace associated with the exception thrown is passed to this function. In this case, you can see that there was a ZeroDivisionError exception being thrown. People implementing libraries can write code that clean up resources, close files etc. in their \_\_exit\_\_() functions.

实际上，在with后面的代码块抛出任何异常时，\_\_exit\_\_()方法被执行。正如例子所示，异常抛出时，与之关联的type，value和stack trace传给\_\_exit\_\_()方法，因此抛出的ZeroDivisionError异常被打印出来了。开发库时，清理资源，关闭文件等等操作，都可以放在\_\_exit\_\_方法当中。

Thus, Python’s with is a nifty construct that makes code a little less verbose and makes cleaning up during exceptions a bit easier.

因此，Python的with语句是提供一个有效的机制，让代码更简练，同时在异常产生时，清理工作更简单。

I have put the code examples given here on [Github](https://github.com/sdqali/python_dojo/tree/master/with Github "Github").

示例代码可以在[Github](https://github.com/sdqali/python_dojo/tree/master/with Github "Github")上面找到。

译注：[本文原文见](http://blog.sdqali.in/blog/2012/07/09/understanding-pythons-with/ "link")
