## 	frida技术栈

Frida是个大杀器，感觉xxx加固，xxx加密都没用，毕竟需要APK去访问服务、获取数据，都需要APK有完整的信息，而这些信息、代码经过各种加密，还是放在APK里面。说白了，就是门锁紧了，钥匙藏在门口某个地方，也许就是地垫下面

说了这么多，那么`hook`到底是什么呢？

```
hook，中文翻译为“钩子”。我的理解是，无论是什么进程，还是函数，都能把它勾过来，然后“加工”一番，再扔出去（执行）。
```

还有一个问题，我们为什么可以hook呢？

```
现在应用一般分为B/S架构，C/S架构。

B/S架构，例如应用场景①中，一般我们需要x-sign加密函数会在某一个js文件中，浏览器都帮你解析好了，慢慢找就好了。

C/S架构中，例如应用场景②中，一般我们需要x-sign加密函数会在apk文件里面，然后慢慢找需要hook的地方。
```



### 简介

`Frida` 是一款强大的动态分析工具，它可以用于对应用程序进行运行时的操作和修改。以下是 Frida 实现的基本原理

1. frida使用的是**动态二进制插桩技术**（**DBI**）

**DBI能做什么？**

> （1）访问进程的内存
> （2）在应用程序运行时覆盖一些功能
> （3）从导入的类中调用函数
> （4）在堆上查找对象实例并使用这些对象实例
> （5）Hook，跟踪和拦截函数等等



2. 动态插桩：`Frida` 使用了 `Just-in-Time (JIT)` 编译技术，可以在应用程序运行时注入 `JavaScript` 脚本，实现对目标应用程序的动态分析和修改。

```java
// 比如源代码
public static int getCalc(int a, int b) {
	return a + b;
}
    
// 插桩后的
public static int getCalc(int a, int b) {
	return 1000;
}
```

顾名思义，在程序源代码的基础上增加（注入）额外的代码，从而达到预期目的或者功能；

**总结：** `Frida`是二进制动态插桩技术，我的理解就是把一段代码动态的插入程序中，但最终不会改变原有的程序，但我们通过动态的插入，可以快速简单的分析出我们想要的那段源代码。



### 1 frida安装

如果安卓版本比较低的话，最新版`frida`不是很稳定。推荐安卓7、8安装`frida 12.8.0`版本，安卓`10/frida14`，安卓`12/frida16`。

官网地址：https://github.com/frida/frida/releases

注：如果不知道怎么下，查看Android手机设备设置

```shell
adb shell getprop ro.product.cpu.abi
```

![image-20230824141112458](C:\Users\Administrator\Desktop\安卓逆向\笔记\images\image-20230824141112458.png)



在`PC`上安装`Python`的运行环境，安装完成后执行下面的命令安装`frida`

荐安卓7、8安装frida 12.8.0版本，安卓10/frida14，安卓12/frida16。

```
frida-tools==9.2.5
pip install frida==14.2.18
```

**注意**：·！！！server与client版本必须保持一致！！！·

目前用的最多的是`frida，frida-ps, frida-trace`， 按照名字基本就可以看出功能了，第一个就是hook命令，第二个是查看进程的，第三个是跟踪函数调用。



一般来讲模拟器使用的是`x86或x86_64`的架构，而真机就是`arm或arm64`架构。这里返回的是` x86_64`，那么我们下载`x86_64`的。

 下载后解压出来只有一个文件，需要把它推送到模拟器中` /data/loacl/tmp `下

```javascript
推送运行命令：adb push fserver /data/local/tmp/

现在执行：adb shell

切换管理员：su

进入目录：cd /data/local/tmp

改下权限：chmod 777 /f14-server

然后就可以运行了：./f14-server
```

1 端口转发

```
adb forward tcp:27042 tcp:27042
```

1.1 避免检测，可以不使用这个端口启动

```python
./frida-server -l 0.0.0.0:6666

# 端口转发
adb forward tcp:6666 tcp:6666
```



### 2 frida基本使用

##### 2.1 列出运行的APP

```
frida-ps -Ua
```

##### 2.2 列表所有的APP

```
frida-ps -Uai
```

##### 2.3 杀死进程

```
frida-kill -U 包名
```

执行 `frida-ps -U` 命令成功输出进程列表

![image-20230824151318627](images\image-20230824151318627.png)



### 3 frida hook 方法

下面是frida客户端命令行的参数帮助

```vhdl
Usage: frida [options] target

Options:
  --version             show program's version number and exit
  -h, --help            show this help message and exit
  -D ID, --device=ID    connect to device with the given ID
  -U, --usb             connect to USB device
  -R, --remote          connect to remote frida-server
  -H HOST, --host=HOST  connect to remote frida-server on HOST
  -f FILE, --file=FILE  spawn FILE
  -n NAME, --attach-name=NAME
                        attach to NAME
  -p PID, --attach-pid=PID 
                        attach to PID
  --debug               enable the Node.js compatible script debugger
  --enable-jit          enable JIT
  -l SCRIPT, --load=SCRIPT
                        load SCRIPT
  -c CODESHARE_URI, --codeshare=CODESHARE_URI
                        load CODESHARE_URI
  -e CODE, --eval=CODE  evaluate CODE
  -q                    quiet mode (no prompt) and quit after -l and -e
  --no-pause            automatically start main thread after startup
  -o LOGFILE, --output=LOGFILE
                        output to log file
```

#### 3.1 Frida两种操作模式

| 操作模式  | 说明                         |
| --------- | ---------------------------- |
| CLI命令行 | Javascript脚本注入进程       |
| RPC       | Python进行Javascript脚本注入 |

> Frida操作APP的两种方式
>

| 方式名称 | 方式说明                                                     | CLI下启动方式  |
| -------- | ------------------------------------------------------------ | -------------- |
| spwan    | 将启动APP的权利交由Frida来控制。不管APP是否启动，都会重新启动APP。 | -f参数指定包名 |
| attach   | 建立在目标APP已经启动的情况下，Frida通过ptrace注入程序从而执行Hook的操作 | 不加-f参数     |

##### 3.1.1 `attach`模式

**JavaScript脚本**

 将一个脚本注入到`Android`目标进程，即需要App处于启动状态， 核心原理是ptrace修改进程内存

```java
frida -U -l myhook.js com.xxx.xxxx
```

参数解释：

- -U 指定对USB设备操作
- -l 指定加载一个Javascript脚本
- 最后指定一个进程名，如果想指定进程pid,用`-p`选项。正在运行的进程可以用`frida-ps -U`命令查看

**python脚本**

```python
import sys
import time
import frida

def on_message(message,data):
    print("message",message)
    print("data",data)

device = frida.get_usb_device()
# 获取目标应用程序的PID
pid = device.get_process("com.luoge.com").pid
# 附加到目标进程
session = device.attach(pid)
# 读取Frida脚本
js_cpde = '''
 Java.perform(function () {
      console.log(Java.use("android.util.Log").getStackTraceString(
        Java.use("java.lang.Throwable").$new()
    ));
})
'''

# 创建Frida脚本并加载
script = session.create_script(js_cpde)
 #  设置当脚本有消息时的回调函数
script.on("message",on_message)
 # 加载脚本到目标进程
script.load() 
# 阻塞主线程，以保持脚本运行
sys.stdin.read()  
```



##### 3.1.2`spawn`模式

**JavaScript脚本**

启动一个新的进程并挂起，在启动的同时注入frida代码，注入完成后调用resume恢复进程。

```java
frida -U -l myhook.js -f com.xxx.xxxx --no-pause
```

参数解释：

- -f 指定一个进程，重启它并注入脚本
- --no-pause 自动运行程序

这种注入脚本的方法，常用于hook在App就启动期就执行的函数。

> frida运行过程中，执行`%resume`重新注入，执行`%reload`来重新加载脚本；执行`exit`结束脚本注入

**python脚本**

```python
import frida

def on_message(message,data):
    print("message",message)
    print("data",data)
    
# 通过Spawn模式启动一个新的应用程序进程，并在该进程中加载Frida脚本
device = frida.get_usb_device()
pid = device.spawn(["com.luoge.com"])
# 恢复应用程序的执行
device.resume(pid)
session = device.attach(pid)

with open("script.js") as f:
    script = session.create_script(f.read())
    
script.on("message",on_message)
script.load()
# 阻塞主线程，以保持脚本运行
sys.stdin.read()
```



##### 3.1.3 转发端口启动

启动服务端

```
./frida-server  -l 0.0.0.0:8881
```

端口转发

```
adb forward tcp:8881 tcp:8881
```

然后直接`cmd`使用该`DroidSSLUnpinning.js`文件

```
frida -H 127.0.0.1:8881 -f com.package.name -l DroidSSLUnpinning.js
```

**python脚本启动**

```python
# -*- coding: UTF-8 -*-

import frida, sys

jsCode = """
console.log("test");
"""

def message(message, data):
    if message['type'] == 'send':
        print(f"[*] {message['payload']}")
    else:
        print(message)
# ./fs120800 -l "0.0.0.0:6666"
# adb wifi 10.0.0.23
process = frida.get_device_manager().add_remote_device('127.0.0.1:6666').attach('com.kevin.app')
script = process.create_script(jsCode)
script.on("message",message)
script.load()

```



#### 3.2 Hook Java方法

##### 3.2.0 frida 关键词解析

1. `Java.use()`：该方法用于获取 Java 类对象，以便于在 JavaScript 中对其进行操作。例如，`Java.use('java.util.ArrayList')` 会返回 `ArrayList` 类的代理对象，从而可以使用 Frida 来访问和修改该类的属性和方法。
2. `Java.perform()`：该方法用于在 Frida 的 JavaScript 环境中执行代码块。这个代码块中可以包含对 Java 类的修改、Hook 方法的实现等操作。
3. `implementation` 属性：用于指定要 Hook 的方法的新实现。通过设置 `implementation` 属性，可以在方法执行前后添加自定义的逻辑。例如，`targetMethod.implementation = function() { ... }` 可以将 `targetMethod` 替换为自定义的实现。
4. `this` 和参数：在自定义的方法实现中，可以使用 `this` 关键字表示原始方法的调用和属性访问。此外，可以使用传入的参数对方法的输入进行修改或记录。
5. 调用原始方法：在自定义的方法实现中，通过调用原始方法，可以确保原始方法的行为仍然被执行。例如，`this.targetMethod.apply(this, arguments)` 可以调用原始的 `targetMethod` 方法。



重载函数常用的类型

![image-20230922154632329](images\image-20230922154632329.png)

**2.1 载入类**

`Java.use`方法用于加载一个`Java`类，相当于Java中的`Class.forName()`。比如要加载一个`String`类：

```java
var StringClass = Java.use("java.lang.String");
```

加载内部类：

```java
var MyClass_InnerClass = Java.use("com.luoyesiqiu.MyClass$InnerClass");
```

其中`InnerClass`是`MyClass`的内部类

##### 3.2.1 hook静态方法和实例方法

静态方法是指属于类而不是类的实例的方法。可以直接通过类名调用静态方法，而无需创建类的实例。

在Java中，使用`static`关键字来声明一个静态方法。静态方法可以在没有实例化类的情况下被调用，它们通常用于执行与类相关的操作或提供公共实用程序方法。

下面是一个示例，展示如何定义和调用静态方法：

```javascript
function demo1(){
        var Utils = Java.use("com.luoge.com.Utils");
      // 应用 hook 函数到静态方法
        Utils.getCalc.implementation = function(arg1, arg2){
            console.log("arg1===>", arg1);
            console.log("arg2===>", arg2);
            // 调用原始方法
            var result = this.getCalc(arg1, arg2);
            console.log("result===>", result);
            // return result;
            return 10000;
        }
}
```

##### 3.2.2  hook 重载方法

重载方法（`Method Overloading`）是指在同一个类中，可以定义多个方法名相同但参数列表（包括参数类型和参数个数）不同的方法。重载方法可以提供更灵活的方法调用方式，使得在不同的情况下使用相同的方法名来执行不同的操作。

下面是一个示例，展示了如何在Java中进行方法重载：

```java
public class MyClass {
    public void myMethod(int num) {
        // 执行针对整数参数的操作
    }
    
    public void myMethod(String str) {
        // 执行针对字符串参数的操作
    }
    
    public void myMethod(int num1, int num2) {
        // 执行针对两个整数参数的操作
    }
}
```

程序报错的时候、可以借助overload覆写

**3.3.2.1 hook所有重载方法**

要在Frida中钩子（hook）所有重载方法，你可以使用Java的`overloads`函数。这个函数可以用于模糊匹配方法名称并找到所有的重载方法。

下面是一个示例代码，演示如何在Frida中钩子所有重载方法：

```javascript
  var targetClass = Java.use('com.luoge.com.Utils');

  // 使用 overloads 函数获取所有重载方法
  var methods = targetClass['getOver'].overloads;
    console.log(methods)
    console.log(methods.length,'多少个重载方法')

  // 遍历所有的重载方法并进行钩子
  methods.forEach(function (method) {
    method.implementation = function () {
      console.log('方法被调用:', method);
      // 在这里可以添加你的自定义逻辑
      // 调用原始方法
      return method.apply(this, arguments);
    };
  });
```

运行后、能看到这个结果

```
方法被调用: function e() {
    [native code]
}
方法被调用: function e() {
    [native code]
}
方法被调用: function e() {
    [native code]
}
```

`overloads`方法返回的内容是三个重载方法，数组格式返回的。每循环一个重载方法、使用`implementation`进行覆写，这样就可以循环完成操作。

参数是重载方法的核心，可以借助`arguments`特性进行后续操作，当重载方法拥有不同的参数个数时，使用不同的代码进行覆写。

##### 3.2.3 frida hook 构造方法

1. 使用 `Java.use()` 方法来获取目标类的实例。
2. 使用 `.overload(...)` 选择要 hook 的构造方法。
3. 使用 `.implementation` 将 hook 函数应用到构造方法上。

```java
function demo6(){
    var money = Java.use("com.luoge.com.Money");
    money.$init.overload('java.lang.String', 'int').implementation = function(str, num){
        console.log(str, num);
        str = "欧元";
        num = 2000;
        this.$init(str, num);
    }
}
```

##### 3.2.4 hook 主动调用

主动调用分2种情况，一种是静态方法、另一种是实例方法，如果是Hook，是不区分静态和实例的

```java
function demo7(){
        var Utils = Java.use("com.luoge.com.Utils");
      // 应用 hook 函数到静态方法
       Utils.setFlag('66666')
}
```

2、**主动调用实例方法**

- `onMatch`方法是在Frida脚本中用于匹配特定条件时触发的回调函数。当满足特定的条件时，比如某个函数被调用或者某个字符串被传递等，`onMatch`方法可以执行相应的操作。
- `onComplete`方法是在`Frida`脚本执行完毕时触发的回调函数。通常用于在所有操作完成后做一些清理工作或输出总结信息。

```java
function demo8(){
        var money = Java.use("com.luoge.com.Money");
        //非静态字段的修改
        Java.choose("com.luoge.com.Money", {
            // 每找一次调用一次
            onMatch: function(obj){
                console.log(obj.getInfo())
            },
            onComplete: function(){
                console.log('内存中Money操作完毕')
            }
        });
}
```

**`onMatch: function(obj) {...}`：** 当找到符合条件的对象时，会调用 `onMatch` 回调函数。在这里，每次找到一个 `com.luoge.com.Money` 类的对象，都会执行 `onMatch` 中的逻辑。`obj` 是代表找到的对象的引用，你可以在这里对找到的对象进行操作。

**`onComplete: function() {...}`：** 当搜索完成时，会调用 `onComplete` 回调函数。在这里，表示内存中的 `com.luoge.com.Money` 类对象的搜索操作完成时，会执行 `onComplete` 中的逻辑。



#### 3.3 Hook Java类

获取和修改类的字段、`hook`内部类、枚举所有加载的类。

```java
function demo9(){
        var money = Java.use("com.luoge.com.Money");
        console.log(JSON.stringify(money.flag));
        money.flag.value = "xialuo";
        console.log(money.flag.value,'修改之后的结果');

        //获取已有对象的方法，非静态字段的修改
        Java.choose("com.luoge.com.Money", {
            onMatch: function(obj){
                obj._name.value = "ouyuan"; //字段名与方方法名相同 前面需要加个下划线
                console.log(obj._name.value )
                obj.num.value = 150000;
                console.log(obj.num.value)
            },
            onComplete: function(){
				console.log("complete!!")
            }
        });
}
```



##### 3.3.1 hook内部类

要hook这个类、需要在类和内部类名之间加上$字符  采用这个分割

```java
var innerClass = Java.use("com.luoge.com.Money.$innerClass")
```

hook内部类

可以使用`InnerClass.$init ` 来进行查找

```javascript
var InnerClass = Java.use('com.luoge.com.Money$innerClass');
// 优化构造函数的 hook
InnerClass.$init.overload('java.lang.String', 'int').implementation = function(name, num) {
    console.log('Hooked innerClass constructor');
    console.log('Parameter name:', name);
    console.log('Parameter num:', num);
    // 调用原始构造函数
    this.$init(name, num);
    console.log(this.outPrint())
    return this.$init(name, num);
};
```



**3.3.2 枚举加载类和方法**

枚举所有加载类

```java
console.log(Java.enumerateLoadedClassesSync().join('\n'))
```

hook类里面的方法

`getDeclaredMethods`  获取该类的所有方法，返回一个方法数组

`getName();`: 获取当前方法的名称。

```java
var money = Java.use("com.luoge.com.Money");
var methods = money.class.getDeclaredMethods()
    console.log(methods)
    for (var i=0;i<methods.length; i++){
        console.log(methods[i].getName())
    }
```

注：无法获取构造方法

##### 3.3.2 hook类的所有方法

首先枚举类的所有方法和hook类的所有重载方法写出来

```java
function demo12(){
        var md5 = Java.use("com.luoge.com.Utils");
        var methods = md5.class.getDeclaredMethods();
        for(var j = 0; j < methods.length; j++){
            var methodName = methods[j].getName();
            console.log(methodName);

            for(var k = 0; k < md5[methodName].overloads.length; k++){
				
                md5[methodName].overloads[k].implementation = function(){
                    for(var i = 0; i < arguments.length; i++){
                        console.log(arguments[i]);
                    }
                    // 调用原始方法，并将原始方法的返回值返回。
                    return this[methodName].apply(this, arguments);
                }
            }

        }
}
```

##### 3.3.3 hook Java特殊类型

在Java中，`Map` 是一个接口，它代表了键值对的集合。它提供了一种将键映射到值的方式，其中每个键只能映射到一个值。`Map` 接口是`java.util`包中的一部分，它是Java中用于实现键值对存储和操作的主要数据结构之一。

`Map` 接口定义了一系列方法，包括：

- `put(key, value)`：将指定的键值对添加到映射中。
- `get(key)`：返回与指定键关联的值。
- `containsKey(key)`：判断映射中是否存在指定的键。
- `containsValue(value)`：判断映射中是否存在指定的值。
- `remove(key)`：从映射中删除与指定键关联的键值对。
- `size()`：返回映射中键值对的数量。
- `keySet()`：返回映射中所有键的集合。
- `values()`：返回映射中所有值的集合。

```java
function demo13(){
        var BufferMap = Java.use("com.luoge.com.BufferMap");
        console.log(BufferMap);
        BufferMap.show.implementation = function(map){
            console.log(JSON.stringify(map));
            //Java map的遍历
            var key = map.keySet(); // 这一行获取了Map对象的键集合，这个键集合是一个迭代器
            var it = key.iterator();
            var result = "";
            while(it.hasNext()){
                var keystr = it.next();
                var valuestr = map.get(keystr);
                result += valuestr;
            }
            console.log(result);
            // return result;

        // 添加数据返回
        map.put("name", "xialuo");
        map.put("age", "18");
        var result = this.show(map);
        // 在这里可以对原始方法的返回值进行修改或者额外的操作
        return result;
    }
}
```

#### 3.4 hook okhttp

##### 3.4.1 hook URL

```javascript
// hook请求地址
function hookOkhttpURl() {
    var Builder = Java.use('okhttp3.Request$Builder');
    Builder.url.overload('okhttp3.HttpUrl').implementation = function (a) {
        console.log('a: ' + a)
        var res = this.url(a);
        return res;
    }
}
```

##### 3.4.2 hook 请求和响应

```javascript
var OkHttpClient = Java.use("okhttp3.OkHttpClient");
OkHttpClient.newCall.overload("okhttp3.Request").implementation = function (request) {
    console.log("HTTP Request -> " + request.url().toString());
    var call = this.newCall(request); // 获取新的 Call 对象
    var response = call.execute(); // 调用新的 Call 对象的 execute 方法
    console.log("HTTP Response -> " + response.body().string());
    return call
}
```

##### 3.4.3 hook头部

```javascript
var Builder = Java.use("okhttp3.Request$Builder");
Builder["addHeader"].implementation = function (str, str2) {
    console.log("key: " + str)
    console.log("val: " + str2)
    // showStacks()
    var result = this["addHeader"](str, str2);
    return result;
};
```



#### 3.5 hook实战

app见课件提供文件

##### App抓包分析

打开app输入信息、可以看见这个包`user/login`发的post请求。里面包含的表单值是加密的

![image-20230922162502974](images\image-20230922162502974.png)

##### 反编译逆向分析

`提示：可以把反编译的Java代码，复制到IDEA分析运行，这样会提高逆向的效率；`
加密参数的关键词：“Encrypt”，点击"转到"来到加密代码处

![image-20230922163154539](images\image-20230922163154539.png)



点击进去，选择`paraMap`可以发现这个位置

![image-20230922163428160](images\image-20230922163428160.png)

#####  使用Frida 辅助分析下

 **hook调试分析**

```javascript
function hooks(){
  Java.perform(function () {
      var JsonRequest = Java.use("com.dodonew.online.http.JsonRequest")
      JsonRequest.paraMap.implementation = function (args){
          console.log('args')
          return this.paraMap(args)
      }
    })
}
```

这里看不到任何的输出结果

```javascript
JsonRequest.addRequestMap.overload('java.util.Map','int').implementation = function (ar1,ar2){
    console.log('ar1',ar1,ar2)
    return this.addRequestMap(ar1,ar2)
  }
```

点击这个方法、发现里面是有结果数据的，继续分析这个算法、可以看到里面也有加密值，核心逻辑为`  String encrypt = RequestUtil.encodeDesMap(RequestUtil.paraMap(addMap, Config.BASE_APPEND, "sign"), this.desKey, this.desIV);`

```javascript
var hooksClass = Java.use("com.dodonew.online.http.RequestUtil");
hooksClass.encodeDesMap.overload('java.lang.String', 'java.lang.String', 'java.lang.String').implementation = function (data, desKey, desIV) {
    console.log("DES加密参数1为："+data);
    console.log("DES加密参数2为："+desKey);
    console.log("DES加密参数3为："+desIV);
    var result = this.encodeDesMap(data, desKey, desIV);
    console.log("encrypt加密结果："+result);
    return result;
}
```

![image-20230922165459487](images\image-20230922165459487.png)

参数1需要确认 `sign` 参数，先分析这个参数来源，追进去方法、可以发现参数在这个位置

![image-20230922191110754](images\image-20230922191110754.png)

```javascript
var Utils = Java.use("com.dodonew.online.util.Utils");
    Utils.md5.implementation = function (a){
    console.log('sign-->',a)
    return this.md5(a)
}
```

使用`Hook`继续对`sign`进行处理

##### 处理结果

从这里可以看到、key是进行了一次md5加密

![image-20230923135849510](images\image-20230923135849510.png)



采集`JavaScript`算法模拟得到结果

```javascript
var CryptoJS = require('crypto-js')

data = '{"equtype":"ANDROID","loginImei":"Androidnull","sign":"1B8C2D425B253379C63F92DD133A5B1A","timeStamp":"1695447943003","userPwd":"112222","username":"13535353535"}'
desKey = CryptoJS.MD5('65102933').toString()
desIv = '32028092'

function desEncrypt() {
    var key = CryptoJS.enc.Hex.parse(desKey),
        iv = CryptoJS.enc.Utf8.parse(desIv),
        srcs = CryptoJS.enc.Utf8.parse(data),
        // CBC 加密模式，Pkcs7 填充方式
        encrypted = CryptoJS.DES.encrypt(srcs, key, {
            iv: iv,
            mode: CryptoJS.mode.CBC,
            padding: CryptoJS.pad.Pkcs7
        });
    return encrypted.toString();
}
```

#### 3.6 其他调试技巧

app提交数据一般都存放在集合里面，本章使用`hook HashMap`的put方法来定位代码的关键位置试试

```java
function main(){
    Java.perform(function () {
    var hashMap = Java.use("java.util.HashMap");
    hashMap.put.implementation = function (a,b){
         console.log('输出--》',a,b)
         return this.put(a,b)
    }
  })
}
```

![image-20230923154619853](images\image-20230923154619853.png)

与`HashMap`一样常用的方法还有`LinkedHashMap` 、`ConcurrentHashMap`，如果hook不到，可以尝试hook这个两个也可以

**输出堆栈信息**

通过 `android.util.Log` 输出当前线程的堆栈跟踪信息。

```javascript
function showStacks() {
    Java.perform(function () {
      console.log(Java.use("android.util.Log").getStackTraceString(
        Java.use("java.lang.Throwable").$new()
    ));
})
}
```

借助异常输出hook的堆栈信息、从而定位方法出现的位置

```javascript
function main(){
    Java.perform(function () {
    var hashMap = Java.use("java.util.HashMap");
    hashMap.put.implementation = function (a,b){
         if(a =="username"){
         	showStacks()
         }
         return this.put(a,b)
    }
  })
}
```

##### hook用户输入

从EditText组件获取用户输入信息，需要判断是否为空，通常是这个方法`isEmpty`

`TextUtils` 是 Android 中的一个实用工具类，位于 `android.text` 包中。它包含了一系列用于处理文本的静态方法，用于进行字符串的操作和比较。以下是一些 `TextUtils` 类的常见用途：

**空字符串检查：** `TextUtils` 可以用于检查字符串是否为 `null` 或空字符串。例如：

```java
String myString = "Hello, World!";
if (TextUtils.isEmpty(myString)) {
    // 字符串为空
} else {
    // 字符串不为空
}

```

针对这个特性，可以编写`Frida`脚本进行调试

```javascript
Java.perform(function () {
    var TextUtils = Java.use("android.text.TextUtils");
    TextUtils.isEmpty.implementation = function (aa){
    console.log('TextUtils--》',aa)
    return this.isEmpty(aa)
	}
})
```

<img src="C:\Users\XL\AppData\Roaming\Typora\typora-user-images\image-20231204151049316.png" alt="image-20231204151049316" style="zoom:50%;" /><img src="C:\Users\XL\AppData\Roaming\Typora\typora-user-images\image-20231204151109357.png" alt="image-20231204151109357" style="zoom:50%;" />



#####  hook json数据

这个方法是使用`Frida`工具来hook `JSONObject`类的相关方法，以实现在运行过程中对其进行调试或修改的目的。

`JSONObject` 是` Android` 的 `JSON API`，可用于解析和生成` JSON` 数据。它提供了多个方法，可用于将键值对添加到 JSON 对象中、从对象中获取特定键的值以及从字符串创建 JSON 对象等。

`Frida` 是一款针对`iOS、Android、macOS、Windows 和 Linux`等平台的动态二进制插桩工具。它允许你拦截另一个进程中执行的函数，并在运行时监视和修改其行为。使用Frida，你可以以高度交互和可扩展的方式，探测和篡改应用程序的内部运行状况。

在这个例子中，我们使用`Frida`来hook `JSONObject`类的相关方法。我们首先使用`Java.use()`函数获取`JSONObject`类，并使用`overload()`方法指定要`hook`的具体方法。然后，在`hook`的实现中，我们可以记录参数、修改参数或返回值，并调用原始的方法以确保其正常功能。

正常`object`操作

```javascript
JSONObject jsonObject = new JSONObject();
jsonObject.put("key1", "value1");
jsonObject.put("key2", 123);
```

编写`Frida`脚本

```javascript
var JSONObject = Java.use('org.json.JSONObject');
// Hook JSONObject.put() 方法
JSONObject.put.overload('java.lang.String', 'java.lang.Object').implementation = function(key, value) {
    console.log('Hooked JSONObject.put()');
    console.log('Key: ' + key.toString());
    console.log('Value: ' + value.toString());
    // 可在此处对参数进行修改或记录
    // 调用原始的put()方法
    return this.put(key, value);
};
```

一般再`Java`里面从`JSONObject`里面取值都是采用`get`或者`getString`方法

```javascript
// Hook JSONObject.getString() 方法
JSONObject.getString.overload('java.lang.String').implementation = function(key) {
    console.log('Hooked JSONObject.getString()');
    console.log('Key: ' + key.toString());
    // 调用原始的getString()方法
    var result = this.getString(key);
    // 可在此处对返回值进行修改或记录
    return result;
};
```

`hook JSONObject`类的put方法，定位到数据提交的地方，`hook getString`方法帝定位到返回响应解析位置。

##### hook排序算法

表单信息签名问题，通常会采用各种算法，比如`md5\sha\mac`算法等，消息摘要特点

+ 明文不一样、加密的结果也不同
+ 加密结果不可逆
+ 加密结果长度一致

一般app在对参数加密时，会先对数据进行排序，因为结果不可逆，服务端需要接收数据复现算法来对比加密结果，而排序算法可以确保不会因为参数乱序而导致结果不同。

比较常见的排序算法有`Collections`的`sort`方法、`Arrays`的`sort`方法，也有`app`自写但是情况较少。

```javascript
Java.perform(function() {
  var Collections = Java.use('java.util.Collections');
  // Hook sort() 方法
  Collections.sort.overload("java.util.List").implementation = function(list) {
    console.log('Hooked Collections.sort()');
    console.log('List: ' + list.toString());
    // 可在此处对参数进行修改或记录
   // 使用 Java.cast 进行类型转换 将list转换成ArrayList类型
    var res = Java.cast(list,Java.use("java.util.ArrayList"))
    console.log('List list-->',res)
      // 调用原始的 sort() 方法
      return this.sort(list);
  };

  Collections.sort.overload("java.util.List","java.util.Comparator").implementation = function(a,b) {
    console.log('Hooked Collections.sort()');
    showStacks()
    var res = Java.cast(a,Java.use("java.util.ArrayList"))
    console.log('Comparator list-->',res)
      return    this.sort(a,b);
  };

});
```

**注：**想看到集合里面的内容，可以使用`Java.cast`进行向下转型

##### hook String字符串转换

数据加密之前、会先将字符串转字节，这个时候可能会用到`String`类的`getBytes`方法，这个时候可以借助`hook`进行操作排查数据

```javascript
Java.perform(function() {
    var StringClass = Java.use('java.lang.String');

    // Hook String 类的构造函数
    StringClass.getBytes.overload().implementation = function () {
        console.log('Original Value');
        // 可在此处修改传入的字符串参数
        var res =  this.getBytes();
        var newString = StringClass.$new(res)
        // 输出修改后的值
        console.log('Modified Value: ' + newString);
        return res;
    };

    // Hook String 类的静态方法
    StringClass.getBytes.overload('java.lang.String').implementation = function (obj) {
        console.log('Hooked String.valueOf()');
        // 可在此处修改传入的对象参数
            showStacks()
        var res =  this.getBytes(obj);
        var newString = StringClass.$new(res,obj)
        // 输出修改后的结果
        console.log('getBytes: ' + newString)
        return res
    }

})
```

#####  hook StringBuilder定位字符串

在分析一些APP的时候，经常看到字符容器`StringBuilder`,可以尝试`hook toString`方法来进行定位关键位置

```javascript
Java.perform(function() {
    // 获取 StringBuilder 类并定义需要 Hook 的方法名
var stringBuilderClass = Java.use("java.lang.StringBuilder");
stringBuilderClass.toString.implementation = function (){
    var res = this.toString.apply(this,arguments)
    if (res == "username=13535353535"){
        showStacks()
        console.log('tostring is called ',res)
    }
    return res
}
})
```

##### hook点击按钮进行定位

再分析APP的时候，有些时候也可以从入口开始

找到SDK文件，在sdk\tools\bin目录下有一个`uiautomatorviewer`，可以使用他进行查看`id`

<img src="C:\Users\XL\AppData\Roaming\Typora\typora-user-images\image-20231204171004037.png" alt="image-20231204171004037" style="zoom: 67%;" />

```javascript
var btn_login_id = Java.use("com.dodonew.online.R$id").btn_login.value;
console.log(btn_login_id,'1111111111111')
var View = Java.use('android.view.View');
View.setOnClickListener.implementation = function(listener) {
    console.log(this.getId(),"22222222222")
    if (this.getId() === btn_login_id){
        showStacks()
        console.log("view.id-->" + this.getId())
    }
    // 调用原始的setOnClickListener方法
    return  this.setOnClickListener(listener);
};
```



#### 3.7 hook算法自吐

在本章节里面，会给大家讲解怎么开发一个算法自吐框架

MD5算法结构

```java
public static String md5_1(String input) throws Exception{
    MessageDigest md5 = MessageDigest.getInstance("MD5");
    md5.update((input + "xialuo").getBytes());
    byte[] digest = md5.digest();
    StringBuilder sb = new StringBuilder();
    for (byte b : digest) {
        // 这里使用了String.format()方法来将一个字节表示为两位的十六进制字符串。
        sb.append(String.format("%02x", b & 0xff));
    }
    return sb.toString();
}
```

1. 首先，通过调用`MessageDigest.getInstance("MD5")`获取MD5算法的实例，返回一个`MessageDigest`对象。
2. 然后，使用`md5.update()`方法将待计算哈希值的字符串转换为字节数组，这里将字符串转换为字节数组，然后写入到 MD5 算法的内部缓冲区中。
3. 调用`md5.digest()` 方法计算出MD5哈希值，返回一个字节数组。
4. 使用`StringBuilder`创建一个字符串构建器对象，用于拼接字节数组中的每个字节的十六进制表示。
5. 通过遍历字节数组，将每个字节转换为两位的十六进制字符串，并追加到字符串构建器中。
6. 最后，使用`sb.toString()`方法将字符串构建器中的内容转换为最终的MD5哈希值字符串，并将其返回。

```javascript
String.format("%02x", b & 0xff)
```

具体来说，`%02x`是一个格式化字符串，其中：

- `%`：表示格式化字符串的起始标记。
- `0`：表示使用零填充不足的位数。
- `2`：表示最少需要两个字符宽度。
- `x`：表示以十六进制形式输出。

而`b & 0xff`则是对字节进行位运算，通过与操作将字节的高位补零，确保结果是一个正整数。这一步是为了确保每个字节都能正确地转换为两位的十六进制字符串

##### MD5载要算法

###### hook getInstance

查看算法类型

```java
var MessageDigest = Java.use("java.security.MessageDigest")
MessageDigest.getInstance.overload('java.lang.String').implementation = function (a) {
	console.log("[*]算法是--》" + a + "《---")
return this.getInstance(a)
}
```

###### **hook update 方法**

在Java中，`MessageDigest.update()`方法用于将数据添加到消息摘要计算中。如果要`Hook`该方法来修改数据内容或记录数据

```javascript
MessageDigest.update.implementation = function (input) {
    console.log(input)
}
```

![image-20231103160054412](images\image-20231103160054412.png)

他这里有4个重载方法，分别是对字节、字节数组、字节数组偏移量、指定的`ByteBuffer`，最好的方式是对他们全hook一下

编写脚本拿到明文数据

```java
Java.perform(function(){
    var ByteString = Java.use("com.android.okhttp.okio.ByteString")
        function toBase64(data){
        console.log("[xl]base64-->",ByteString.of(data).base64());
    }
    function toHex(data){
        console.log("[xl]hex-->",ByteString.of(data).hex());
    }
    function toUtf8(data){
        console.log("[xl]utf8-->",ByteString.of(data).utf8());
    }
    // 测试样例
    // toUtf8([-27, -92, -113, -26, -76, -101])
    // demo1()
})
```

将字节数组参数和加密结果以上边的形式打印出来，能极大的减少分析难度

######  **hook digest 方法**

```java
MessageDigest.digest.implementation = function (input) {
	console.log(input)
}
```

![image-20231103191108180](images\image-20231103191108180.png)

1、通过执行最后的操作来完成哈希运算，返回字节数组

2、使用指定的字节数组对载要进行最终更新，然后完成计算

3、通过执行最后的操作来完成哈希运算，返回字节数组，有三个参数，字节数组、offset、len

![image-20231103194237457](images\image-20231103194237457.png)



##### HMAC算法

算法示例

```java
public static String mac_1(String args) throws Exception {
    String str = "HmacSHA1";
    SecretKey key = new SecretKeySpec("FridaHook".getBytes(), str);
    Mac mac = Mac.getInstance(str);
    mac.init(key);
    mac.update(args.getBytes());
    return Utils.byteToHexString(mac.doFinal());
}
```

1. 创建了一个`SecretKey`对象`key`，使用`SecretKeySpec`类的构造方法。该构造方法接受两个参数，第一个参数是密钥的字节数组，这里使用`FridaHook`字符串的字节数组作为密钥；第二个参数是指定的算法，这里传入了之前定义的`str`。
2. 使用`Mac.getInstance(str)`方法获取指定算法的Mac对象，这里传入了之前定义的`str`。
3. 调用`mac.init(key)`方法初始化Mac对象，传入之前创建的密钥`key`。
4. 调用`mac.update(args.getBytes())`方法将要计算的数据写入Mac对象中，这里传入了`args`字符串的字节数组。
5. 调用`mac.doFinal()`方法计算MAC值。该方法返回一个字符串

可以`hook SecretKeySpec` 或者` init`方法

```javascript
var mac = Java.use("javax.crypto.Mac");
	mac.init.implementation = function (){
}
```

报错信息如下

```javascript
Error: init(): has more than one overload, use .overload(<signature>) to choose from:
        .overload('java.security.Key')
        .overload('java.security.Key', 'java.security.spec.AlgorithmParameterSpec')
```

可以发现有两个重载方法

```javascript
// 使用给定的建初始化此mac对象
void init(key key)
// 使用给的的建和算法参数初始化此mac对象
void init(key key,AlgorithmPaameterSpec params)
```

编写hook脚本

```
 key.getEncoded();
```

通过这个方法能拿到key对象的密钥字节数组，获取的数据是key。

 ######  hook updete 方法

###### hook doFinal 方法



##### DES算法hook

```java
public static String des_1(String args) throws Exception {
        SecretKey secretKey = SecretKeyFactory.getInstance("DES").generateSecret(new DESKeySpec("12345678".getBytes()));
        AlgorithmParameterSpec iv = new IvParameterSpec("87654321".getBytes());
        Cipher cipher = Cipher.getInstance("DES/CBC/PKCS5Padding");
        cipher.init(1, secretKey, iv);
        cipher.update(args.getBytes());
        return Base64.encodeToString(cipher.doFinal(), 0);
    }
```

DES算法 我们需要知道的就是入参、IV、KEY、密文

根据这些信息去hook就好啦

![image-20231103223810908](images\image-20231103223810908.png)



### 4 objection工具

#### 安装地址

```text
https://github.com/sensepost/objection/releases
https://github.com/frida/frida/releases?page=7
```

##### 安装须知

由于不同版本得`Frida`支持得`API`不同， `objection`需要安装`frida`版本发布之后更新得版本，且版本要靠近，比如我装的`frida`版本是`14.2.18`，发布时间是`20.12.20`，那`objection`需要安装`1.11.0`这个版本。

```
pip install objection == 1.11.0
```

####  基本操作

#####  应用注入

```javascript
objection -g package explore 
```

**指定ip和端口的连接**

```javascript
objection -N -h 192.168.0.0 -p 9999 -g packageName explore
```

**spawn启动**

```javascript
objection -g packageName explore --startup-command "android hooking watch class 'com.xx.classname"
```

**启动activity或service**

```javascript
// 获取activities
android hooking list activities 
// 切换activities
android intent launch_activity 强制启动界面
```

**spawn启动-打印参数、返回值、函数调用栈**

```javascript
objection -g packageName explore --startup-command "android hooking watch class_method 'com.classname.methedname' --dump-args --dump-return --dump-backtrace"
–-dump-args: 显示参数; 
--dump-return: 显示返回值; 
--dump-backtrace: 显示堆栈
```

测试豆瓣签名信息注入

```javascript
objection -g com.douban.frodo  explore --startup-command "android hooking watch class_method com.douban.frodo.network.ApiSignatureHelper.a --dump-args --dump-return --dump-backtrace"
```



**基于frida的objection及其插件wallbreaker 命令列表**

```javascript
!           # 执行操作系统的命令(注意:不是在所连接device上执行命令)
android     # 执行指定的 Android 命令
        clipboard
                monitor
        deoptimize    # Force the VM to execute everything in the interpreter
        heap
                evaluate    # 在 Java 类中执行 JavaScript 脚本。
                execute     # 在 Java 类中执行 方法。android heap execute 实例ID 实例方法
                print
                search
                        instances  # 在当前Android heap上搜索类的实例。android heap search instances 类名
        hooking
                generate
                        class    #  A generic hook manager for Classes
                        simple   #  Simple hooks for each Class method
                get
                        current_activity    #  获取当前 前景(foregrounded) activity
                list
                        activities    # 列出已经登记的 Activities
                        class_loaders # 列出已经登记的 class loaders 
                        class_methods # 列出一个类上的可用的方法
                        classes       # 列出当前载入的所有类
                        receivers     # 列出已经登记的 BroadcastReceivers
                        services      # 列出已经登记的 Services
                search
                        classes 关键字    # 搜索与名称匹配的Java类
                        methods 关键字    # 搜索与名称匹配的Java方法
                set
                        return_value    # 设置一个方法的返回值。只支持布尔返回
                watch
                        class           # Watches for invocations of all methods in a class
                        class_method    # Watches for invocations of a specific class method
        intent
                launch_activity    # 使用Intent启动Activity类
                launch_service     # Launch a Service class using an Intent
        keystore
                clear    # 清除 Android KeyStore
                list     # 列出 Android KeyStore 中的条目
                watch    # 监视 Android KeyStore 的使用
        proxy    
                set      # 为应用程序设置代理
        root
            disable    # 试图禁用 root 检测
            simulate   # 试图模拟已经 root 的环境
        shell_exec     # 执行shell命令
        sslpinning
                disable    # 尝试禁用 SSL pinning 在各种 Java libraries/classes
        ui
            FLAG_SECURE    # Control FLAG_SECURE of the current Activity
            screenshot     # 在当前 Activity 进行截图
cd          # 改变当前工作目录
commands        
        clear    # 清除当前会话命令的历史记录
        history  # 列出当前会话命令历史记录
        save     # 将在此会话中运行的所有惟一命令保存到一个文件中
env         # 打印环境信息
evaluate    # 执行 JavaScript。( Evaluate JavaScript within the agent )
exit        # 退出
file
        cat         # 打印文件内容
        download    # 下载一个文件
        http        
                start    # Start's an HTTP server in the current working directory
                status   # Get the status of the HTTP server
                stop     # Stop's a running HTTP server
        upload           # 上传一个文件
frida       # 获取关于 frida 环境的信息
import      # 从完整路径导入 frida 脚本并运行
ios         执行指定的 ios 命令
        bundles
        cookies
        heap
        hooking
        info
        jailbreak
        keychain
        monitor
        nsurlcredentialstorage
        nsuserdefaults
        pasteboard
        plist
        sslpinning
        ui
jobs
        kill    # 结束一个任务。这个操作不会写在卸载或者退出当前脚本
        list    # 列出当前所有的任务
ls              # 列出当前工作目录下的文件
memory
        dump
                all 文件名                      # Dump 当前进程的整个内存
                from_base 起始地址 字节数 文件  # 将(x)个字节的内存从基址转储到文件
        list
                exports    # List the exports of a module. (列出模块的导出)
                modules    # List loaded modules in the current process. (列出当前进程中已加载的模块)
        search    # 搜索模块。用法：memory search "<pattern eg: 41 41 41 ?? 41>" (--string) (--offsets-only)
        write     # 将原始字节写入内存地址。小心使用!
ping        # ping agent
plugin      
        load    # 载入插接
pwd             # 打印当前工作目录
reconnect       # 重新连接 device
rm              # 从 device 上删除文件 
sqlite          # sqlite 数据库命令
        connect  # 连接到SQLite数据库文件
ui
        alert    # 显示警报消息，可选地指定要显示的消息。(目前iOS崩溃)

```



##### memory指令

`objection memory` 是一个在移动应用程序渗透测试工具 `Objection `中可用的指令。`Objection `是一款基于` Frida `的运行时探查和干扰工具，用于帮助安全测试人员在移动应用程序上进行安全评估。

`objection memory` 命令用于查看和操纵进程的内存数据。它提供了多个子命令来执行不同的操作，例如查看内存中的字符串、搜索内存、修改内存中的值等。

以下是一些常用的 `objection memory` 子命令：

- `search <value>`: 在进程内存中搜索包含指定值的内存位置。
- `string <address>`: 从指定内存地址开始，打印内存中的 ASCII 字符串。
- `dump <address> <length>`: 从指定的内存地址开始，以十六进制格式打印指定长度的内存内容。
- `write <address> <value>`: 将指定的值写入到指定的内存地址中。
- `alloc <size>`: 在进程内存中分配指定大小的内存块。
- `dealloc <address>`: 释放指定地址的内存块。

这些指令可以用于检查应用程序的内存状态、搜索特定的敏感数据，甚至修改应用程序的运行时行为

```python
 # 枚举内存中所有模块
memory list modules 
memory list modules --json xx.log

# 列举so文件的导出方法
memory list exports libEGL_adreno.so

# 获取类示例ID
search instances search instances com.xx.xx.class

android sslpinning disable
```

##### 内存漫游

```python
# 枚举加载的activitie
android hooking list activities    

# 列出内存中所有的类
android hooking list classes

# 在内存中所有已加载的类中搜索包含特定关键词的类
android hooking search classes key 
例: android hooking search classes Signature
# 在内存中所有已加载的类中搜索包含特定关键词的方法
android hooking search methods key

# 列出类的所有方法
android hooking list class_methods 包名.类名
例: android hooking list class_methods com.douban.frodo.network.ApiSignatureHelper

# hook类的所有方法
android hooking watch class 包名.类名.方法
例：android hooking watch class_method com.douban.frodo.network.ApiSignatureHelper.a

# hook方法的参数、返回值和调用栈(–dump-args: 显示参数; --dump-return: 显示返回值; --dump-backtrace: 显示堆栈)
android hooking watch class_method 包名.类名.方法 --dump-args --dump-return --dump-backtrace

# 默认会Hook方法的所有重载
android hooking watch class_method 包名.类名.方法

# 如果只需hook其中一个重载函数 指定参数类型 多个参数用逗号分隔
android hooking watch class_method 包名.类名.方法 "参数1,参数2"

# 搜索堆中的实例
android heap search instances 包名.类名 --fresh

# 设置返回值（只支持bool类型）// 检测 su 特定抓包 证书检测 版本检测
android hooking set return_value com.xxx.xxx.methodName false  
```



#### app实战

+  海南航空962

#####  抓包分析

![image-20220920203345591](images\image-20220920203345591.png)

###### 绕过SSL验证

启动objection

```
objection -g com.rytong.hnair explore 
```

输入SSL绕过命令

```
android sslpinning disable
```

##### jadx分析

打开jadx进行搜索，发现参数在这里面

![image-20220920203041927](images\image-20220920203041927.png)



然后来到了这个，可以看到`i6.b("hnairSign", signRequest);`，其中hnairSign 是该`String signRequest = signRequest(aVar);`方法返回来的。

![image-20230919221446700](images\image-20230919221446700.png)

紧接着跟进 `signRequest(aVar)` 函数看一看，来到这里

![image-20230919221513934](images\image-20230919221513934.png)



可以看到 **signRequest** 函数的入参是**v.a aVar** ，返回值是**String**，并且是由该(**String**) **i.p(HNASignature.getHNASignature(headersForSign, queryForSign, requestBodyForSign, str, a9), new String[]{">>"}).get(0)**; 返回来的。

在这里我们看到了**HNASignature.getHNASignature()**，根据函数命名规范，大胆的猜测下，这里就是最核心的加密代码。

**getHNASignature()**函数的入参是**headersForSign, queryForSign, requestBodyForSign, str, a9** 几个连续的**String**，具体是什么，目前不知道，不过先放置在这里。

继续跟进去，来到这里。

![image-20230919221638695](images\image-20230919221638695.png)



#####  使用`objection`大法

```python
android hooking watch class_method com.rytong.hnair.HNASignature.getHNASignature --dump-args --dump-return
```

- android 代表Android程序
- hooking 我要hook了
- watch 看什么东西
- class_method 看类方法
- com.rytong.hnair.HNASignature.getHNASignature 对应的 `类.方法`
- --dump-args 查看参数
- --dump-return 查看返回值

于是乎只要调用到了我们hook的函数,就会有

![image-20220920203835996](images\image-20220920203835996.png)



### 5  抓包专题

> 参考：https://blog.csdn.net/m0_45406092/article/details/114638087

#### 5.1 单向证书验证

APP: 斑马2.3.12
SLL证书验证

<img src="images\image-20231026193452699.png" alt="image-20231026193452699" style="zoom:80%;" />

**SSL层握手协议**

```
1.Client Hello 发给服务端：一串随机数 和 客户端支持哪些加密算法 一般是RSA加密
2.Server Hello 回客户端：一串随机数 和 确定使用哪种加密算法
3.Certificate: 服务端把要公匙证书发给客户端
4.Client key Exchange: 客户端把自己的证书又发给服务端
5.Session Ticket：开始传输数据/用key 给http数据加密 传输给服务端
```

```
HTTPS的特点：1.数据加密，不再明文传播2.使用数字证书（CA Certificate Authority）(赛门铁克)，做身份校验，防止数据被截获，只有合法证书持有者才能读取数据
```

![image-20231025172251774](images\image-20231025172251774.png)



目前App用到的网络请求库都是 `OKHttp3`，这个库有一个 api：`CertificatePinner`这个 API 的作用是

> 用于自定义证书验证策略，以增强网络安全性。在进行TLS/SSL连接时，服务器会提供一个SSL证书，客户端需要验证该证书的有效性以确保连接的安全性。CertificatePinner允许你指定哪些SSL证书是可接受的，从而有效地限制了哪些服务器可以与你的应用程序通信。
>
> 具体来说，CertificatePinner允许你指定一组预期的证书公钥或证书固定哈希值（SHA-256），以便与服务器提供的证书进行比较。如果服务器提供的证书与你指定的公钥或哈希值不匹配，则连接会被拒绝，从而防止中间人攻击和其他安全风险。

除了这个 API 可以用来防止中间人攻击，`OKHttp3`还有其他防止中间人攻击的方法，如`X509TrustManager`

> X509TrustManager 是Java安全架构中的一个接口，位于 javax.net.ssl 包中，主要用于处理 SSL（Secure Sockets Layer）和后续的 TLS（Transport Layer Security）协议中的证书信任管理功能。在基于SSL/TLS的网络通信中，特别是HTTPS连接，X509TrustManager扮演着至关重要的角色，它的主要职责是验证远程主机提供的X.509证书链的有效性。



##### ssl pinning解决办法

+ 使用`Hook`手段 `Hook APP`端网络请求库对 `ssl`证书的判断方法
  借助`frida` 程序 `DroidSSLUnpinning `https://github.com/WooyunDota/DroidSSLUnpinning
+ 逆向`APP` 扣出里面的证书 放到`charles`里 让`charles`使用真实证书做代理

运行脚本进行`hook`操作

```java
frida -U -f org.zxq.teleri --no-pause -l DroidSSLUnpinning.js
```

![image-20231025172815323](images\image-20231025172815323.png)



运行完之后，可以发现能正常抓包数据



#### 5.2 双向证书验证

在绝大部分的情况下，TLS 都是客户端认证服务端的真实性的，但是在一些非常注重安全的场景下（例如匿名社交），部分 APP 会开启 TLS 的双向验证，也就是说服务端也要验证客户端的真实性。

在这种情况，客户端必然内置了一套公钥证书和私钥。相对于服务端，APP 有很大的砸壳风险，所以公钥证书和私钥一般都是极其隐蔽的，比如说写到 .so 里，隐藏在一个混淆的妈都不认识的随机数算法函数里

https://blog.csdn.net/qq_41866988/article/details/130977936

##### 证书处理

![image-20231026151437203](images\image-20231026151437203.png)

dump下的证书在这个路径

![image-20240430155617871](C:\Users\XL\AppData\Roaming\Typora\typora-user-images\image-20240430155617871.png)

然后刷手机、去目录里面查看证书文件

![image-20231026151519449](images\image-20231026151519449.png)

导出证书文件

```
adb pull /sdcard/Download/client_keystore__2023_10_26_14_46_47_73.p12 D:\works\out_codes
```

**携带证书抓包**

需要在抓包工具里面携带S验证C的证书，记得域名抓包的这个地址。

![image-20231106195056634](images\image-20231106195056634.png)

这里配好，现在已经解决S验证码C了，但是C验证码S还需要处理下，此时可以运行之前的脚本给他处理下。

![image-20231106195346712](images\image-20231106195346712.png)

携带证书请求试一下效果

```python
headers = {"User-Agent":"Mozilla/5.0 (Linux; Android 11; M2007J22C Build/RP1A.200720.011; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/90.0.4430.210 Mobile Safari/537.36 mztapk","Accept":"*/*","Referer":"https://app.mmzztt.com",}
import requests
from requests_pkcs12 import Pkcs12Adapter
session = requests.session()
url = "https://api.iimzt.com/app/list/posts?type=post&order=last&paged=1"
# 指定双向认证网站的pfx证书文件，并附带证书的密码
session.mount(url, Pkcs12Adapter(pkcs12_filename='./client_keystore__2023_10_26_14_46_47_73.p12', pkcs12_password="hooker"))
# 使用session调用get/post方法
response = session.get(url, headers=headers,verify=False)
print(response.status_code)
print(response.text)
```

![image-20231026151659421](images\image-20231026151659421.png)

很不幸，这个返回结果被加密了。。。。

##### 对算法进行hook

```shell
frida -U com.mmzztt.app --no-pause -l sanfa.js
```

![image-20231026173950102](images\image-20231026173950102.png)

**使用Python还原**

```python
# -*- coding: utf-8 -*-
import hashlib
import json

from Crypto.Cipher import AES
headers = {"User-Agent":"Mozilla/5.0 (Linux; Android 11; M2007J22C Build/RP1A.200720.011; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/90.0.4430.210 Mobile Safari/537.36 mztapk","Accept":"*/*","Referer":"https://app.mmzztt.com",}
import requests
from requests_pkcs12 import Pkcs12Adapter
session = requests.session()
url = "https://api.iimzt.com/app/list/posts?type=post&order=last&paged=1"
# 指定双向认证网站的pfx证书文件，并附带证书的密码
session.mount(url, Pkcs12Adapter(pkcs12_filename='./client_keystore__2023_10_26_14_46_47_73.p12', pkcs12_password="hooker"))
# 使用session调用get/post方法
response = session.get(url, headers=headers,verify=False)
print(response.status_code)
print(response.text)

items = json.loads(response.text)
key = items['last']

def decrypt(data, key):
    md5 = hashlib.md5(str(key).encode("UTF-8")).hexdigest()
    key = md5[8:24]
    iv = '0809060801020609'
    enc_data = bytearray.fromhex(data)
    aes = AES.new(key.encode(), AES.MODE_CBC, iv.encode())
    decrypted = aes.decrypt(enc_data).strip()
    text = decrypted.decode()
    return text

data = items.get('data')
text = decrypt(data, key)
print(json.loads(text))
```



#### 5.3 闲鱼实战

```
7.8.8版本
```

使用Charles、Fiddle等抓包工具对淘系App进行抓包时，你会发现总是抓不到包，出现请求不走Charles代理的情况。这是因为淘系app底层网络通信的协议并不是普通的http协议，而是自己实现的一套私有协议Spdy。

![image-20231031181743210](images\image-20231031181743210.png)



![image-20231031181811945](images\image-20231031181811945.png)





使用Frida处理一下

```java
Java.perform(function () {
    var SwitchConfig = Java.use('mtopsdk.mtop.global.SwitchConfig');
    SwitchConfig.A.overload().implementation = function () {
        return false;
    }
});
```

然后使用命令`frida -U -l xy_hook.js --no-pause -f com.taobao.idlefish`启动App进行抓包。抓包效果如下，可以看到已经能抓到了。

抓包之后的结果展示

![image-20231031182017016](images\image-20231031182017016.png)



**xposed模块编写**

```java
package com.example.sdpy_xposed;

import android.util.Log;
import de.robv.android.xposed.XC_MethodHook;
import de.robv.android.xposed.XposedHelpers;
import de.robv.android.xposed.IXposedHookLoadPackage;
import de.robv.android.xposed.callbacks.XC_LoadPackage.LoadPackageParam;

public class Hook implements IXposedHookLoadPackage {
    public void handleLoadPackage(final LoadPackageParam lpparam) throws Throwable {
        Log.d("XP", "模块挂载中......");
        if (lpparam.packageName.equals("com.taobao.idlefish")) {
            Log.i("XP", "进入apk");
            Class<?> clazz = XposedHelpers.findClass("mtopsdk.mtop.global.SwitchConfig", lpparam.classLoader);
            XposedHelpers.findAndHookMethod(clazz, "A", new XC_MethodHook() {
                public void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    Log.i("XP", "isSocketConnected 已被Hook");
                    param.setResult((Object) false);
                }
            });
        }
        ;
    }
}
```

参数处理

1.阿里系通用这一套加密算法，主要是`x-sign，x-sgext，x_mini_wua，x_umt`这四个加密参数，解决了其中一个`app`，其他的比如淘X，咸X等app都相差不大了，改改参数，或者替换不同的方法名称就行；

![image-20231101223815662](images\image-20231101223815662.png)

使用**Frida hook**之后的结果

![image-20231101224015341](images\image-20231101224015341.png)



#### 5.4 拼多多实战

APP版本：6.26

抓包分析

![image-20240508171219859](C:\Users\XL\AppData\Roaming\Typora\typora-user-images\image-20240508171219859.png)

前言：拼多多charles抓包分析发现跟商品相关的请求头里都带了一个`anti-token`的字段且每次都不一样,那么下面的操作就从分析`anti-token`开始了

1、使用`frida hook hashmap`测试

![image-20240506133428946](images\image-20240506133428946.png)

2、使用jadx反编译跟进到源码查看，发现这个方法也能hook到入参和返回值，是他就无疑了

![image-20240506134110949](images\image-20240506134110949.png)

验证一下方法hook，可以看到有入参和返回值，入参是请求地址和一个布尔值，返回值是一个对象，需要循环提取。

![image-20240506134421427](images\image-20240506134421427.png)

继续往里面跟进，看一下是什么算法进行加密的，详情页会走这个逻辑生成结果，搜索页不会走这个逻辑。

![image-20240506140008683](images\image-20240506140008683.png)<img src="images\image-20240506140018875.png" alt="image-20240506140018875" style="zoom: 67%;" />

找到搜索页加密参数生成的位置，根据f()方法进行计算anti-token.

![image-20240506140159028](images\image-20240506140159028.png)

SecureNative.deviceInfo2()方法生成,传入的str为pdd生成的固定id 一个字符串.

![image-20240506140715109](images\image-20240506140715109.png)

![image-20240506140919328](images\image-20240506140919328.png)

验证一下处理的结果，发现和抓包工具能对比上

![image-20240506141243116](images\image-20240506141243116.png)

结合frida rpc测试数据

![image-20240506210112636](images\image-20240506210112636.png)

![image-20240506204848456](images\image-20240506204848456.png)



#### 5.5  使用r0capture抓包

> 地址：https://github.com/r0ysue/r0capture
>

**简介**

+ 仅限安卓平台，测试安卓真机7、8、9、10、11 可用 ；
+ 无视所有证书校验或绑定，不用考虑任何证书的事情；
+ 通杀`TCP/IP`四层模型中的应用层中的全部协议；
+ 通杀协议包括：`Http,WebSocket,Ftp,Xmpp,Imap,Smtp,Protobuf`等等、以及它们的`SSL`版本；
+ 通杀所有应用层框架，包括`HttpUrlConnection、Okhttp1/3/4、Retrofit/Volley`等等；



高版本使用猛兼容安卓**12/13**

> 参考：https://ecapture.cc/zh/examples/android.html

```
安卓设备的内核版本只有在5.10版本上才可以进行无任何修改的开箱抓包操作。
如果你不知道你的设备的内核版本，可以打开手机，在关于页面中看到内核信息。 
或者手机连接电脑，执行adb shell cat /proc/version或者adb shell uname -a进行查看。
```

> 新版： https://github.com/gojue/ecapture/releases 

使用方法

```
adb push ecapture /data/local/tmp/
adb shell chmod 777 /data/local/tmp/ecapture
```

启动方式

```
ecapture目前已经适配安卓13，执行下面的命令就可以开启全局抓包：
$ adb root
# adb shell /data/local/tmp/ecapture tls
```



**5.1.1 查看运行APP的包名**

```
adb shell am monitor
```

**5.1.2 脚本抓包**

`Spawn` 模式，直接抓包

```
python r0capture.py -U -f 包名
```

`Attach `模式，将抓包内容保存成`pcap`格式文件

```
python r0capture.py -U 包名 -p 文件名.pcap
```

**5.1.3 wireshark进行分析**

![image-20231026194745636](images\image-20231026194745636.png)

可以对这个数据，直接解析成请求，里面是一长串`curl`



#### 5.6  某房实战

```
app:贝壳找房
版本 2.66.0
```

**分析开始**

抓包分析

![image-20240506212652038](C:\Users\XL\AppData\Roaming\Typora\typora-user-images\image-20240506212652038.png)



**打开jadx-gui 直接搜索关键词  "Authorization":**

![image-20231109164007549](C:\Users\Administrator\Desktop\安卓逆向\笔记\images\image-20231109164007549.png)

**Frida直接验证该拦截器是否是我们要的:**

```javascript
let inteceptor = Java.use("com.bk.base.netimpl.interceptor.d");
inteceptor.signRequest.implementation = function (v1, v2) {
    let res = this.signRequest(v1, v2);
    console.log("签名结果: " + res);
    return res;
}
```

**阅读代码可以发现参数是`signString`，通过`getSignString`方法返回的，这个时候可以对他进行跟进**

```java
    String signString = a.ji().getSignString(sb2.toString(), hashMap);
        newBuilder.url(build);
        newBuilder.header("Authorization", signString);
```

**对这个方法hook一下**

<img src="images\image-20231109160504767.png" alt="image-20231109160504767" style="zoom:80%;" />

```javascript
let getSig = Java.use("com.bk.base.netimpl.a");
getSig.getSignString.implementation = function (v1, v2) {
    console.log("参数1： " + v1);
    var it = v2.keySet().iterator();
    let result = "";
    while(it.hasNext()){
        var keystr = it.next().toString();
        var valuestr = v2.get(keystr).toString();
        result += keystr + valuestr + "        ";
    }
    console.log("参数2： " + result);
    return this.getSignString(v1,v2);
}
```

**接下来重要的一步就是 DeviceUtil.SHA1ToString 这个函数的参数是如何计算而来的(传入的类似一个MD5的值):**

```java
sb3.append(sb);
LjLogUtil.d(str3, sb3.toString());
String SHA1ToString = DeviceUtil.SHA1ToString(sb.toString());
```

```javascript
let encrypt = Java.use("com.bk.base.util.bk.DeviceUtil");
encrypt.SHA1ToString.implementation = function (v1) {
    let res = this.SHA1ToString(v1)
    console.log("加密参数为: " + v1 + "加密结果为: " + res)
    return res;
}
//加密参数为: d5e343d453aecca8b14b2dc687c381ca 加密结果为:48b82883cf47ac02ed32aef24fea684851c7dab9
```

**由于是get请求不携带参数的话不存在for循环拼接参数,所以这里有理由怀疑的是 origin=d5e343d453aecca8b14b2dc687c381ca 就是他的httpAppSecret(重启APP 看一下这个httpAppSecret 是否会变):**

```
d5e343d453aecca8b14b2dc687c381ca   基本上不会变
```

**整个流程就分析完了,我们最终知道的是如果是get请求 默认的 appSecret d5e343d453aecca8b14b2dc687c381ca SHA1加密后 在前面拼接 "20180111_android:" 然后在Base64编码即可,如果get请求中携带了参数也需要参与加密**

```
20180111_android:ddad2a926f37569e2c2c7a7e6a960fc4f14ef0b8
```

**如果是post请求 用默认的appSecret + key=value(这里要字典排序key)然后SHA1 后 在参数前面拼接 "20180111_android:" 得到最终的字符串后 Base64编码即可**

![image-20231109165136938](images\image-20231109165136938.png)



### 6 APP加固

**什么是加壳**

所谓“壳”，本人的理解是为app添加了一身保护自己的“壳”。从而保护自己核心代码算法,提高破解/盗版/二次打包的难度，还可以缓解代码注入/动态调试/内存注入攻击

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xb3L3wnAiatjfdrVVRXYXNy6T54s2gOFEicw2p3X9JAka5RMw8p3pLxVHqpKTB9uO0AjURs5rUZUkgzViaMbcAXkA/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**为什么要脱壳**

想要app反编译。我们就需要为其脱掉保护自身的“壳”。从而实现对app的反编译及打包。

**怎样脱壳**

目前市面上的脱壳工具还是很多的。直接拖入想要脱壳的app，一键脱壳。我们在安卓市场随便下载一款工具。加载到脱壳工具检查是否有壳。

![img](images\企业微信截图_17028981734742.png)

**壳的发展**

壳的种类⾮常多，根据其种类不同，使⽤的技术也不同，这⾥稍微简单分个类：

+ ⼀代整体型壳：采⽤ Dex 整体加密，动态加载运⾏的机制；
+ ⼆代函数抽取型壳：粒度更细，将⽅法单独抽取出来，加密保存，解密执⾏；
+ 三代 VMP、Dex2C 壳：独⽴虚拟机解释执⾏、语义等价语法迁移，强度最⾼。



#### 6.1 frida-Dexdump

暴力破解法` frida-dexdump `的使用介绍

提示：以下是本篇文章正文内容，下面案例可供参考

利⽤ frida 的搜索内存，通过匹配 **DEX** ⽂件的特征，例如 **DEX** ⽂件的 文件头 中的 **magic** --- **dex.035** 这个特征。 **frida-Dexdump** 便是

这种脱壳⽅法的代表作。

+ 对于于完整的 dex，采⽤暴⼒搜索 dex035 即可找到。
+ ⽽对于抹头的 dex，通过匹配⼀些特征来找到，然后⾃动修复⽂件头。

**1. 工具安装**

> 开源地址：https://github.com/hluwa/FRIDA-DEXDump

```python
pip install frida-dexdump
```

**2.原理**
利用`frida hook libart.so`中的`OpenMemory`方法，拿到内存中`dex`的地址，计算出`dex`文件的大小，从内存中将`dex`导出。

**3.上手**

- ```
  frida-dexdump -FU //直接注入前台应用
  ```

- ```
  frida-dexdump -U -f com.app.pkgname
  ```

<img src="images\image-20231026204838035.png" alt="image-20231026204838035" style="zoom:80%;" />

4、查找文件

`GDA4（Game Developer Assistant 4）`是一款用于`Android`游戏开发的工具。它提供了许多功能和特性，帮助开发者分析和修改`Android`游戏。

以下是`GDA4`工具的一些主要功能：

1. 反编译：`GDA4`可以将已安装在`Android`设备上的游戏应用反编译为可读的代码，以便开发者进行分析和修改。
2. 资源查看器：`GDA4`提供了一个资源查看器，可以浏览游戏应用中的各种资源文件，如图片、声音和视频等。
3. 脚本编辑器：`GDA4`内置了一个脚本编辑器，开发者可以使用该编辑器创建和编辑脚本文件，以实现自定义的游戏逻辑和功能。
4. 数据库浏览器：`GDA4`可以浏览游戏应用中的数据库文件，开发者可以查看和修改游戏中存储的数据。
5. `Hook`框架：`GDA4`提供了一个`Hook`框架，开发者可以使用该框架在游戏运行时注入自己的代码，以实现一些特殊的功能和修改。

![image-20231026205939002](images\image-20231026205939002.png)

`window`使用

```
findstr /s /i /m "com.uzmap.pkg.LauncherUI" *
```

`linux`使用

```
grep -ril   com.uzmap.pkg.LauncherUI ./* 
```

![image-20231026213238326](images\image-20231026213238326.png)

使用`jadx-gui`查看对应数据

```
jadx-gui .\classes06.dex
```

#### 6.2 数据解密实战

工具：达川观察、360加固

#####  **抓包分析**

![image-20240429171228222](images\image-20240429171228222.png)



可以发现头部有一个签名参数'sign'，如果想采集数据必须要过这个签名

使用课件`hook算法`脚本对算法进行`hook`,发现定位不到。



##### 脱壳技巧

本次继续使用`frida-dexdump`进行脱壳

```java
frida-dexdump -U -f com.app.pkgname //指定包名
```

定位`dex`文件位置

![image-20240429191610077](images\image-20240429191610077.png)

##### jadx分析

使用`jadx`定位`hook`得算法，查找对应参数位置

![image-20240429191751670](images\image-20240429191751670.png)

编写`hook`脚本

```javascript
function hook_sign(){
    let HttpRequestEntity = Java.use("cn.thecover.lib.http.data.entity.HttpRequestEntity");
    HttpRequestEntity["getSign"].implementation = function (str, str2, str3) {
    console.log(`HttpRequestEntity.getSign is called: str=${str}, str2=${str2}, str3=${str3}`);
    let result = this["getSign"](str, str2, str3);
    console.log(`HttpRequestEntity.getSign result=${result}`);
    return result;
	};
}
```



可以发现，他在SO文件里面加密的，可以`hook`一下`NewStringUTF`,

`NewStringUTF`是`JNI（Java Native Interface）`的一个函数，用于将C/C++字符串转换为Java的UTF-8字符串。在SO文件中，它通常是用于与Java代码进行交互，特别是当你需要从C/C++代码返回一个字符串给Java代码时

```javascript
function hook_NewStringUTF(){

    var artModule = Process.findModuleByName("libart.so");
    var symbols = artModule.enumerateSymbols();
    var newStringUTF = null;
    for (let i = 0; i < symbols.length; i++) {
        let symbol = symbols[i];
        if(symbol.name.indexOf("NewStringUTF") != -1 && symbol.name.indexOf("Check") == -1){
            console.log(symbol.name,symbol.address);
            newStringUTF = symbol.address;
        }
    }

    Interceptor.attach(newStringUTF, {
        onEnter : function(args){

                console.log("\x1B[33m[调用栈]:\x1B[0m\n",Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\n') + '\n');
                console.log("\x1B[33m[加密字符串]:\x1B[0m",args[1].readCString());

        },onLeave : function(retval){
            // console.log("newStringUTF retvl" ,retval)
        }
    })

}
```

![image-20240429200319746](images\image-20240429200319746.png)



**汇编分析**

`import export` :导入表和导出表，表中内容为程序需要的外表函数和可以被外部程序调用的函数（涉及到动态链接的相关知识）

![image-20240429201002019](images\image-20240429201002019.png)

编写脚本hook当前这个方法

```javascript

function hook_so(){
     let message = Module.findBaseAddress("libwtf.so")
    console.log(message)
    let funs = message.add(0x43834);
    Interceptor.attach(funs,{
        onEnter:function (args){
             // 在进入函数时执行的操作
            console.log("arg1-->",args[1].readCString())

        },
        onLeave:function (retval){
            // 捕获返回值
            console.log(retval)

        }
    })
}

```

![image-20240429202101635](images\image-20240429202101635.png)



#### 6.3 SO文件处理

`Frida`是一款用于动态分析和修改应用程序的强大工具。它可以用来`hook`（勾取）应用程序中的函数并进行代码注入，从而实现对函数的监视、修改行为以及数据的读写操作。下面是一些常见的`Frida hook`文件的`API：`

1. `Module.findExportByName(moduleName, exportName)` 

   通过模块名称和导出函数名查找在内存中加载的模块，并返回导出函数的地址。

2. `Module.getExportByName(moduleName, exportName)` 

   通过模块名称和导出函数名获取导出函数的地址，与`Module.findExportByName`类似，但是如果找不到指定的导出函数则会返回null。

3. `Module.findBaseAddress(moduleName)` 

   通过模块名称查找在内存中加载的模块，并返回模块的基址（起始地址）

4. `Interceptor.attach(target, callbacks)` 

   对目标函数进行拦截，并在函数执行前后调用相应的回调函数。

5. `Interceptor.replace(target, replacement)` 

   替换目标函数的实现，将其重定向到自定义的替代函数。

6. `Memory.readPointer(address)` 

   读取指定内存地址处的指针值。

7. `Memory.writePointer(address, value)` 

   向指定内存地址写入指针值。

8. `Memory.readByteArray(address, length)` 

   读取指定内存地址处的字节数组。

9. `Memory.writeByteArray(address, data)` 

   向指定内存地址写入字节数组。

10. `Memory.protect(address, size, prot)` 

    修改指定内存范围的保护属性，如可读、可写、可执行等。

##### **枚举内存中的` so `文件**

用于查看目标 module 是否被正常加载, 使用 `Process.enumerateModules()` 将当前加载的所有 so 文件打印出来

```javascript
function hookSo1(){
    var modules = Process.enumerateModules();
    for (var i in modules) {
        var module = modules[i];
        console.log(module.name)
    }
}
```

获取指定 `so` 文件的基地址

```javascript
function hook_module() {
    var baseAddr = Module.findBaseAddress("libnative-lib.so");
    console.log("baseAddr", baseAddr);
}
```

##### 枚举导入函数

`enumerateImports` 是 Frida 提供的一个方法，用于枚举指定模块的导入函数。该方法将返回一个对象数组，每个对象表示模块导入的一个函数，包含以下字段：

- `name`：函数名称。
- `type`：函数类型，如 `function`、`data`、`unknown` 等。
- `address`：函数地址，即导入表中函数指针的地址。

```javascript
var imports = Module.enumerateImports("libnative-lib.so");
for(var i = 0; i < imports.length; i++){
    console.log(JSON.stringify(imports[i]));
    console.log(imports[i].address);
}
```

##### 枚举导出函数

`enumerateExports` 是 Frida 库提供的一个函数，它允许你枚举给定模块的导出函数和变量

```javascript
var exports = Module.enumerateExports("libnative-lib.so");
for (var i = 0; i < exports.length; i++) {
    if (exports[i].name.includes("Java")) {
        console.log(JSON.stringify(exports[i]));
    }
}
```

通过导出函数名定位` native `方法

```javascript
var addr = Module.findExportByName("libnative-lib.so","Java_com_luoge_com_MainActivity_nativeMessage")
console.log(addr)
```

##### hook有导出函数

`Interceptor.attach()` 是 `Frida` 中的一个函数，用于在目标函数执行前或执行后注入 `JavaScript `代码并进行拦截。它要两个参数：

- `target`：要拦截的函数地址或函数名称。
- `callbacks`：包含 `onEnter` 和/或 `onLeave` 回调函数的对象，用于定义拦截时执行的 JavaScript 代码。

`Interceptor.attach()` 函数的基本用法如下：

```javascript
Interceptor.attach(target, {
  onEnter: function (args) {
    // 在函数执行前执行的代码
  },
  onLeave: function (retval) {
    // 在函数执行后执行的代码
  }
});
```

通过` Intercept `拦截器打印` native `方法参数和返回值, 并修改返回值

- `onEnter`: 函数(args) : 回调函数, 给定一个参数 args, 用于读取或者写入参数作为 `NativePointer` 对象的指针;
- `onLeave`: 函数(retval) : 回调函数给定一个参数 retval, 该参数是包含原始返回值的` NativePointer` 派生对象; 可以调用 `retval.replace(1234)` 以整数 1234 替换返回值, 或者调用`retval.replace(ptr("0x1234"))` 以替换为指针;
- 注意: `retval` 对象会在 `onLeave` 调用中回收, 因此不要将其存储在回调之外使用, 如果需要存储包含的值, 需要制作深拷贝, 如 `ptr(retval.toString())`

```javascript
function find_func_from_exports() {
    var add_c_addr = Module.findExportByName("libnative-lib.so", "add_c");
    console.log("add_c_addr is :",add_c_addr);
    // 添加拦截器
    Interceptor.attach(add_c_addr,{
        // 打印入参
        onEnter: function (args) {
            console.log("add_c called");
            console.log("arg1:",args[0].toInt32());
            console.log("arg2", args[1].toInt32());
        },
        // 打印返回值
        onLeave: function (returnValue) {
            console.log("add_c result is :", returnValue.toInt32());
            // 修改返回值
            returnValue.replace(100);
        }
    })
}
```

#### 6.4 smail语法学习

地址：https://github.com/ollide/intellij-java2smali

**基本类型的表示形式：**

smali数据类型都是用一个字母表示，如果你熟悉Java的数据类型，你会发现表示smali数据类型的字母其实是Java基本数据类型首字母的大写，除boolean类型外，在smail中用大写的”Z”表示boolean类型。

smali是Dalvik的寄存器语言，smali代码是dex反编译而来的·

| 名称        | 注释                       |
| :---------- | :------------------------- |
| .class      | 类名                       |
| .super      | 父类名，继承的上级类名名称 |
| .source     | 源名                       |
| .field      | 变量                       |
| .method     | 方法名                     |
| .register   | 寄存器                     |
| .end method | 方法名的结束               |
| public      | 公有                       |
| protected   | 半公开，只有同一家人才能用 |
| private     | 私有，只能自己使用         |
| .parameter  | 方法参数                   |
| .prologue   | 方法开始                   |
| .line xxx   | 位于第xxx行                |

数据类型对应

| smali类型    | java类型                          | 注释                 |
| :----------- | :-------------------------------- | :------------------- |
| V            | void                              | 无返回值             |
| Z            | boolean                           | 布尔值类型，返回0或1 |
| B            | byte                              | 字节类型，返回字节   |
| S            | short                             | 短整数类型，返回数字 |
| C            | char                              | 字符类型，返回字符   |
| I            | int                               | 整数类型，返回数字   |
| J            | long （64位 需要2个寄存器存储）   | 长整数类型，返回数字 |
| F            | float                             | 单浮点类型，返回数字 |
| D            | double （64位 需要2个寄存器存储） | 双浮点类型，返回数字 |
| string       | String                            | 文本类型，返回字符串 |
| Lxxx/xxx/xxx | object                            | 对象类型，返回对象   |

常用指令

| 关键字       | 注释                                                   |
| :----------- | :----------------------------------------------------- |
| const        | 重写整数属性，真假属性内容，只能是数字类型             |
| const-string | 重写字符串内容                                         |
| const-wide   | 重写长整数类型，多用于修改到期时间。                   |
| return       | 返回指令                                               |
| if-eq        | 全称equal(a=b)，比较寄存器ab内容，相同则跳             |
| if-ne        | 全称not equal(a!=b)，ab内容不相同则跳                  |
| if-eqz       | 全称equal zero(a=0)，z即是0的标记，a等于0则跳          |
| if-nez       | 全称not equal zero(a!=0)，a不等于0则跳                 |
| if-ge        | 全称greater equal(a>=b)，a大于或等于则跳               |
| if-le        | 全称little equal(a<=b)，a小于或等于则跳                |
| goto         | 强制跳到指定位置                                       |
| switch       | 分支跳转，一般会有多个分支线，并根据指令跳转到适当位置 |
| iget         | 获取寄存器数据                                         |

##### 复现Java转smail语法

`java2smali`插件一款`Android Studio`上非常实用的插件。通过该插件可以将`Java`或者`Kotlin`的源文件转为`smali`文件。使用该插件可以方便我们做如下事情:

(1)、对比`java`源文件学习`smali`语法

(2)、`apk`重打包过程中`Smali`插桩	

依次点击**Android Studio**菜单**"File->Settings->plugins"** 搜索 **`java2smali`** 进行安装

![image-20231221154152009](images\image-20231221154152009.png)



##### 字段转编译

```java
private static String flag = "jb666";
public String name;
public int num;
public static boolean run = true;
```

转后的代码

```java
// 首先是类的定义部分：
.class public Lcom/luoge/xp1219/SmailTest;
.super Ljava/lang/Object;
.source "SmailTest.java"
    
// 接着是类的字段定义部分：
# static fields
静态字段

# instance fields
实例字段
    
// 紧接着是类的静态初始化块 <clinit>：
# direct methods
	
    
// 最后是类的构造方法 <init>
  构造方法没有参数，且没有具体的实现，只是调用了父类 java.lang.Object 的构造方法。
```



##### 条件判断

**Java代码样例**

```java
public static String func2(int num) {

    if (num >= 100) {
        return "优秀";
    }  else if(num >= 80){
        return  "普通";
    }else {
        return "及格";
    }

};
```



**条件判断关键字如下**

![image-20231221172932079](images\image-20231221172932079.png)



**smail转完的代码**

```java
const/16 v0, 0x64   //  parseInt("0x64", 16);
```

- `const/16`：表示将一个16位的常量加载到寄存器中。 
- `v0`：表示目标寄存器，这里是寄存器v0。
- `0x64`：表示要加载的常量值，这里是16进制数0x64，即十进制中的100

```java
if-lt p0, v0, :cond_7
```

- `if-lt`：表示条件跳转，即根据两个寄存器中的值进行判断，如果满足条件，则跳转到指定的标记。
- `p0`：表示第一个操作数寄存器，这里是寄存器p0。
- `v0`：表示第二个操作数寄存器，这里是寄存器v0。
- `:cond_7`：表示跳转目标，这里是标记为`:cond_7`的代码块

```java
const-string v0, "\u4f18\u79c0"
```

- `const-string`：表示将一个字符串常量加载到寄存器中。
- `v0`：表示目标寄存器，这里是寄存器v0。
- `"\u4f18\u79c0"`：表示要加载的字符串常量，这里是Unicode编码表示的字符串常量，\u4f18表示"优"，\u79c0表示"秀"，即"优秀"。



```java
.method public static func2(I)Ljava/lang/String;
    .registers 2
    .param p0, "num"    # I

    .prologue
    .line 20
    const/16 v0, 0x64

    if-lt p0, v0, :cond_7

    .line 21
    const-string v0, "\u4f18\u79c0"

    .line 25
    :goto_6
    return-object v0

    .line 22
    :cond_7
    const/16 v0, 0x50

    if-lt p0, v0, :cond_e

    .line 23
    const-string v0, "\u666e\u901a"

    goto :goto_6

    .line 25
    :cond_e
    const-string v0, "\u53ca\u683c"

    goto :goto_6
.end method
```



##### 循环语句

**Java代码示例**

```java
public static void func3(String arg1){
    String[] nameArray = {"夏洛", "aa", "bb"};
    for (int idx = 0; idx < nameArray.length; idx++) {
        String item = nameArray[idx];
        System.out.println(item);
    }
};
```



**smail转完的代码**

```java
const/4 v3, 0x3
```

- `const/4`：表示将一个4位（即1个字节）的常量加载到寄存器中。
- `v3`：表示目标寄存器，这里是寄存器v3。
- `0x3`：表示要加载的常量，这里是16进制数 `0x3`。

```java
new-array v2, v3, [Ljava/lang/String;
```

- `new-array`：表示创建一个新的数组。
- `v2`：表示目标寄存器，这里是寄存器v2。
- `v3`：表示数组长度的寄存器，这里是寄存器v3。
- `[Ljava/lang/String;`：表示数组的类型描述符，这里是字符串类型的数组。

```java
aput-object v4, v2, v3
```

- `aput-object`：表示将一个对象存储到数组中。
- `v4`：表示要存储的对象所在的寄存器，这里是寄存器v4。
- `v2`：表示目标数组所在的寄存器，这里是寄存器v2。
- `v3`：表示要存储的位置索引，这里是寄存器v3。

```java
array-length v3, v2
```

- `array-length`：表示获取数组的长度。
- `v3`：表示目标寄存器，这里是寄存器v3。
- `v2`：表示源数组所在的寄存器，这里是寄存器v2。

```java
 sget-object v3, Ljava/lang/System;->out:Ljava/io/PrintStream;
```

- `sget-object`：表示获取静态对象类型的字段值。
- `v3`：表示目标寄存器，这里是寄存器v3，用于存储获取到的字段值。
- `Ljava/lang/System;->out:Ljava/io/PrintStream;`：表示要获取的静态字段，这里是System类中的静态字段`out`，它的类型是`Ljava/io/PrintStream;`。

```java
add-int/lit8 v0, v0, 0x1
```

- `add-int/lit8`：表示对寄存器中的整数值和一个8位立即数进行相加操作。
- `v0`：表示目标寄存器，这里是寄存器v0，用于存储相加的结果。
- `v0`：表示源操作数寄存器，这里也是寄存器v0，表示要进行相加的原始值。
- `0x1`：表示要相加的立即数。

```java
.method public static func3(Ljava/lang/String;)V
    .registers 6
    .param p0, "arg1"    # Ljava/lang/String;

    .prologue
    .line 31
    const/4 v3, 0x3

    new-array v2, v3, [Ljava/lang/String;

    const/4 v3, 0x0

    const-string v4, "\u590f\u6d1b"

    aput-object v4, v2, v3

    const/4 v3, 0x1

    const-string v4, "aa"

    aput-object v4, v2, v3

    const/4 v3, 0x2

    const-string v4, "bb"

    aput-object v4, v2, v3

    .line 32
    .local v2, "nameArray":[Ljava/lang/String;
    const/4 v0, 0x0

    .local v0, "idx":I
    :goto_13
    array-length v3, v2

    if-ge v0, v3, :cond_20

    .line 33
    aget-object v1, v2, v0

    .line 34
    .local v1, "item":Ljava/lang/String;
    sget-object v3, Ljava/lang/System;->out:Ljava/io/PrintStream;

    invoke-virtual {v3, v1}, Ljava/io/PrintStream;->println(Ljava/lang/String;)V

    .line 32
    add-int/lit8 v0, v0, 0x1

    goto :goto_13

    .line 36
    .end local v1    # "item":Ljava/lang/String;
    :cond_20
    return-void
.end method
```



##### 方法调用

一、方法调用关键字介绍
`smali`语言方法调用关键字主要有以下五种

+ `invoke-virtual`主要用于非私有实例方法的调用。实例方法指不是构造方法、父类方法等的属于这个类的一般方法。
+ `invoke-direct`主要用于构造方法（包括父类的构造方法）和私有方法的调用
+ `invoke-static`主要用于静态方法的调用
+ `invoke-super`主要用于父类成员方法（不包括父类构造方法）的调用
+ `invoke-interface`主要用于接口方法的调用



**实例方法调用**

```java
invoke-virtual {参数}, 方法所属类的全包名路径->方法名(参数类型)方法返回值类型
```

当有多个参数时，格式为`{参数1,参数2}`,使用其他关键字调用时相同



**Java代码示例**

```java
package com.luoge.xp1219;

public class SmailTest {

    //SmailTest类的构造方法，调用getName方法
    public SmailTest(String a){
        getName();//java里默认是this.getName()，不写这个this也会有
    }

    //非私有实例方法getName()
    public String getName(){
        return "hello";
    }
}
```

对应的完整`smali`代码如下，这里参数里有一个`p0`，因为`getName()`方法虽然无参，但是需要当前对象进行调用，即`this.getName()`，`p0`指当前对象，可以理解为`java`里的`this`

```java
.method public constructor <init>(Ljava/lang/String;)V   // 构造方法
    .registers 2
    .param p1, "a"    # Ljava/lang/String;   

    .prologue
    .line 41
    invoke-direct {p0}, Ljava/lang/Object;-><init>()V  // 调用父类Object的构造方法

    .line 42
    invoke-virtual {p0}, Lcom/luoge/xp1219/SmailTest;->getName()Ljava/lang/String;
	//  这里参数里有一个p0，因为getName虽然无参，但是需要当前对象进行调用，即this.getName()
    .line 43
    return-void
.end method


# virtual methods
.method public getName()Ljava/lang/String;
    .registers 2

    .prologue
    .line 47
    const-string v0, "hello"

    return-object v0
.end method

```



**静态方法调用**

```java
invoke-static{参数}, 方法所属类的全包名路径->方法名(参数类型)方法返回值类型
```

**Java代码示例**

```java
package com.luoge.xp1219;

public class SmailTest {

    //SmailTest类的构造方法，调用getName方法
    public SmailTest(String a){
        getNames();//java里默认是this.getName()，不写这个this也会有
    }

    //非私有实例方法getName()
    private static String getNames(){
        return "hello world";
    }
}
```

对应的完整smali代码如下，注意这里{}里没有p0

```java
.method public constructor <init>(Ljava/lang/String;)V   // 构造方法
    .registers 2
    .param p1, "a"    # Ljava/lang/String;   

    .prologue
    .line 41
    invoke-direct {p0}, Ljava/lang/Object;-><init>()V  // 调用父类Object的构造方法

    .line 42
    invoke-static {}, Lcom/luoge/xp1219/SmailTest;->getNames()Ljava/lang/String;
	//  #注意这里{}里没有p0，因为静态方法随类加载不需要创建对象
    .line 43
    return-void
.end method


.method public getNames()Ljava/lang/String;
    .registers 2

    .prologue
    .line 47
    const-string v0, "hello world"

    return-object v0
.end method

```



切割字符

```java
 public static void func3() {

        String url = "http://appapi.yndaily.com/api/v2/articles/10?pageToken=&size=20&headPageSize=&clientVersionCode=409&pjCode=code_ynrb&device_size=1080.0x2236.0&deviceOs=10&channel=qq&deviceModel=Google-Pixel+4&clientVersion=4.0.9&udid=11ec2adac162837a&platform=android";

        // 解析查询参数
        Map<String, String> paramMap = new HashMap<>();
        String[] parts = url.split("\\?");
        if (parts.length > 1) {
            String[] queries = parts[1].split("&");
            for (String query : queries) {
                String[] kv = query.split("=");
                if (kv.length == 2) {
                    try {
                        paramMap.put(kv[0], URLDecoder.decode(kv[1], "UTF-8"));
                    } catch (UnsupportedEncodingException e) {
                        e.printStackTrace();
                    }
                }
            }
        }

        // 使用Base64编码
        String encodedParams = Base64.encodeToString(paramMap.toString().getBytes(), Base64.DEFAULT);

        Log.d("Base64", encodedParams);

    }
```

### 7 实战专题

#### 7.1 商城专场

**IDA工具**

地址：https://www.52pojie.cn/thread-1874203-1-1.html

交互式反汇编器专业版（Interactive Disassembler Professional）常称为 IDA Pro。IDA 是一种递归下降反汇编器，是个逆向分析的神器之一，可以分析x86、x64、ARM、MIPS、Java、.NET等众多平台的程序代码

**以下是IDA的一些主要特点和功能：**

+ 反汇编：IDA可以将二进制文件转换为易于阅读和理解的汇编代码。它支持多种处理器架构，包括x86、ARM、MIPS等。

+ 交互式分析：IDA提供了一个交互式界面，使用户能够浏览和修改反汇编代码，并通过注释、标签等方式添加注释和标记。

+ 反编译：IDA可以尝试将反汇编代码转换为高级语言代码，如C语言。这有助于更好地理解二进制文件的功能和逻辑。

+ 动态调试：IDA可以与调试器集成，允许用户在分析过程中进行动态调试，以便更深入地理解程序的运行过程。

+ 插件支持：IDA支持自定义插件，用户可以根据自己的需求编写插件来扩展IDA的功能。

+ 图形化表示：IDA可以生成控制流图、函数调用图等图形化表示，帮助用户更好地理解程序的结构和逻辑。

+ 导入和导出功能：IDA支持导入和导出多种格式的二进制文件，如ELF、PE、Mach-O等。它还可以与其他工具进行集成，方便数据的共享和分析。

**工具介绍**

Stuuctures：可以查看程序的结构体：

Enums：可以查看枚举信息：

Imports：可以查看到输入函数，导入表即程序中调用到的外面的函数：

Exports：可以查看到输出函数：

**快捷键**

通过F5大致可以看出代码构造

shift+f12 可以查看字符代码格式



APP版本

```
唯品会 7.45
```

##### Java层抓包分析

打开抓包工具 charles进行分析，可以发现对于API采集需要突破当前这个参数，否则不返回信息

![image-20240429210055780](images\image-20240429210055780.png)



##### jadx静态分析

jadx静态分析，打开app搜索关键词`api_sign`，可以发现有参数位置

![image-20240103205519025](C:\Users\XL\AppData\Roaming\Typora\typora-user-images\image-20240103205519025.png)

跟进去上边`str`赋值方法`m52297b`,看看参数是怎么来的

![image-20240103205620638](images\image-20240103205620638.png)

总结：该方法里面传入上下文、MAP类型参数、2个字符串参数，返回值`m52298a` 对参数进行了系列处理，继续跟进这个方法

![image-20240103205752994](C:\Users\XL\AppData\Roaming\Typora\typora-user-images\image-20240103205752994.png)

总结：这里面可以看到调用了` apiSign`方法对参数进行处理，继续跟进到方法里面

![image-20240103210007126](images\image-20240103210007126.png)

总结：该方法对传入的参数继续调用`getMapParamsSign`进行处理、可以跟进这个方法查看

![image-20240103210120150](C:\Users\XL\AppData\Roaming\Typora\typora-user-images\image-20240103210120150.png)

总结：该方法对`MAp`类型数据进行迭代处理，然后还是调用`getSignHash`方法处理数据

![image-20240103210226639](images\image-20240103210226639.png)

总结：该方法调用`m8334gs`

![image-20240103210333377](C:\Users\XL\AppData\Roaming\Typora\typora-user-images\image-20240103210333377.png)

总结：到了这个位置、可以看到参数已经到达边界、有可能这里就是数据返回的位置，可以使用frida hook一下结果，与抓包对比一致。

![image-20240103210556947](images\image-20240103210556947.png)

从中可以看到`clazz`就是`com.vip.vcsp.KeyInfo`类，跟进去这个类继续查找

![image-20240103210958970](images\image-20240103210958970.png)

总结：这里调用`gsNav`方法、该方法是加载了一个`jni`文件，也被称为app里面的`so`文件，由C语言编写的算法，被Java调用。

从上面可以看到so文件名称是`private static final String LibName = "keyinfo";`，一般so文件都会打包在apk文件，可以从里面查找文件，用ida汇编工具进行查询。

##### IDA汇编分析

打开工具，拖进去so文件即可，可以看到这个页面，工具的使用可以看之前的教程，

![image-20240103211404808](images\image-20240103211404808.png)



我们先查询导出函数，看下是否有Java层代码，说明函数是静态注册的

![image-20240103211532159](images\image-20240103211532159.png)

根进去方法、然后按`F5`转伪C代码，从这里面分析业务逻辑

![image-20240103211653544](images\image-20240103211653544.png)

根进去前面的方法、可以看到这个界面，里面对入参进行一系列处理

![image-20240103211820937](images\image-20240103211820937.png)

总结：可以看到一系列对map的操作，但是不急着对这部分分析，接着往下看

![image-20240103212014684](images\image-20240103212014684.png)

总结：可以看到一个`j_getByteHash`方法，通过函数名推断它极大可能就是执行摘要算法的地方。

![image-20240103212047665](images\image-20240103212047665.png)

通过名称推断是sha1算法。使用frida hook一下，可以发现参数和加密结果都能对上，那就无疑了，后续就可以算法还原了。

![image-20240103212346861](images\image-20240103212346861.png)

贴一张拿数据的结果

![image-20240103212517624](images\image-20240103212517624.png)



#### 7.2 资讯专场

**ida代码跟踪分析**

`encryptJavaString`有6个参数，`v4`是`env`不管，`cstr`是要加密的字符串，`hashCode`是`apk`签名，固定字符串和0

![image-20240102193126628](images\image-20240102193126628.png)



这里有一个`rsa_encrypt_key_ex`的方法，这里首先怀疑是`rsa`对称加密。`rsa`加密是有一个密钥的，这里要分析密钥是什么，使用第一个文件密钥试了试，发现不对，可能是`decryptKey`方法对密钥进行处理。

![image-20240102193321433](images\image-20240102193321433.png)



编写`HOOK`脚本调试下

```javascript

Java.perform(function (){
    var n_addr_func_offset = 0x0000024E4;
    let message = Module.findBaseAddress("libutil.so")
    let funs = message.add(n_addr_func_offset);
var out2 = null;
var out2_len = null;
Interceptor.attach(funs,{

    onEnter:function (args){

        console.log("Hook so -decryptKey------");
        console.log("keyInNum", args[0].toInt32());
        console.log( " data-cipher " ,Memory.readUtf8String(args[1]));
        console.log( "data-cipher-len" , args[2].toInt32());
        console.log( "out2", args[3]);
        console.log( "buf_len" , args[4].toInt32());
        out2 =args[3]
        out2_len = args[4]
        console.log("======日三====S=二=E日=三=E=====")
    },
  onLeave: function (retVal) {
            console.log("encrypt---", retVal.toInt32());
            console.log("++Out2++", hexdump(out2));
            console.log("result=====日===========")
    }
})
})
```



![image-20240102193707757](images\image-20240102193707757.png)



### 8 apk反编译修改

#### 8.1 指令打包

下载`apktools`工具

APKTool是一款用于反编译和重新打包Android[应用程序](https://www.yimenapp.com/kb-yimen/tag/应用程序/)的工具。通过APKTool，可以将APK[文件](https://www.yimenapp.com/kb-yimen/tag/文件/)（Android应用程序的安装包）解包为Smali代码和资源文件，然后对其进行修改和分析，并重新打包成可安装的APK文件。

> jadx:https://blog.csdn.net/yiran1919/article/details/132671812

地址：https://bitbucket.org/iBotPeaches/apktool/downloads/

**apktool**工具的作用:

1. 我们可以通过`apktool`去查看apk的`AndroidManifest`、`XML`文件和图片资源文件。
2. 可以修改`apk`中的代码，不过并非源码，而是`smail`文件，这个后面会讲



**APKTool基于以下原理工作**：

1. 反编译DEX文件：Android应用程序的核心代码存储在DEX文件中，APKTool可以将DEX文件转换为更容易阅读和修改的Smali代码。Smali代码类似于Java代码，但使用了不同的语法和指令集。

2. 解码资源文件：Android应用程序的资源文件包括图像、布局、字符串和其他资源，APKTool可以将这些文件解码为易读的XML格式。通过修改这些XML文件，可以改变应用程序的外观和功能。

3. 重新打包应用程序：在对应用程序进行修改后，APKTool将Smali代码和解码后的资源文件重新打包成新的APK文件。这个新的APK文件将具有经过修改的应用程序代码和资源。

4. 签名APK文件：为了保证应用程序的完整性和安全性，APK文件需要进行签名。APKTool可以帮助生成自签名[证书](https://www.yimenapp.com/kb-yimen/tag/证书/)，并将其添加到重新打包的APK文件中。这个自签名的



**反编译apk**

指令说明：https://blog.csdn.net/u013408264/article/details/129485943

```shell
# 反编译
-b ，–no–debug-info  # 防止baksmali打印出调试信息
-f， --force		# 强制删除目标文件目录，执行反编译命令时，强制覆盖存在。
–only-main-classes   # 反编译根目录中的dex文件
# 回编译
-o， --output	# 将反编译后的文件写入到指定的文件路径下
```



**执行指令编译APK文件**

```
java -jar apktool_2.9.0.jar d douban7.18.apk -only-main-classes
```

![image-20231218200929924](images\image-20231218200929924.png)

手动进去`res->values`打开`strings.xml`文件

![image-20231218201254344](images\image-20231218201254344.png)



**重新打包apk**

**注：**打包的时候注意不要有中文路径

```
java -jar   apktool_2.9.0.jar  b douban7.18
```

![image-20231218203042557](images\image-20231218203042557.png)

打包之后的结果在`dist`目录

![image-20231218203142281](images\image-20231218203142281.png)



此时，打包的APP还无法在手机安装使用，没有涉及签名。

**生成keystore**

`keytool`、`jarsigner` 工具是JAVA JDK自带的，配置好JAVA环境即可！

输入命令：`keytool -genkey -alias new.keystore -keyalg RSA -validity 20000 -keystore new.keystore`，然后在输入两次最低六位数的密钥口令，下面的信息直接`Enter`，最后`y`即可！

![image-20231218203650472](images\image-20231218203650472.png)

**签名APK**

未签名APK不能在安卓手机上安装，想要安装则想要对齐签名。

输入命令：`jarsigner -verbose -keystore new.keystore -signedjar news.apk old.apk new.keystore`然后再输入密钥库的密码短语即你之前设置的密钥口令，即可签名！

```
jarsigner -verbose -keystore new.keystore -signedjar 签名后.apk 未签名.apk new.keystore
```

![image-20231218204126042](images\image-20231218204126042.png)

打包之后的文件路径

![image-20231218204225660](images\image-20231218204225660.png)

至此，apk反编译、重打包、签名全部完成。可以用命令`adb install news-douban718.apk`将此apk安装到手机即可！

![image-20231218204507478](images\image-20231218204507478.png)



#### 8.2 工具打包

工具见课件明细

1、执行指令进行编译

```
java -jar apktool_2.9.0.jar d -f douban7.18.apk  -only-main-classes
```

2、打开工具，选中`“提取dex”`操作，然后将`apk`拖动到源文件中；点击“操作”，得到`dex`文件

<img src="images\image-20231218214250617.png" alt="image-20231218214250617" style="zoom: 67%;" />



3、选中`“dex转jar”`操作，然后将得到的`dex`文件拖动到源文件中；点击“操作”，得到`jar`文件，`jd`工具会自动打开`jar`文件，这样就看到`java`源码了（如果应用进行了混淆，看到的源码类和方法都是abc等）



<img src="images\image-20231218214524979.png" alt="image-20231218214524979" style="zoom:67%;" />





### 9 frida 检测

1、**D-Bus**

`Frida`使用`D-Bus`协议通信，可以遍历`/proc/net/tcp`文件，或者直接从`0-65535`
向每个开放的端口发送`D-Bus`认证消息，哪个端口回复了REJECT，就是`frida-server`

2、**端口检测**

通过检测默认端口`27042`是否开放来检测frida是否开启，只需要启动时指定端口即可绕过

3、**进程名检测**

1. 进程名检测，遍历运行的进程列表，检测`frida-server`是否运行

4、**默认路径**

`frida`默认会在`/data/local/tmp/re.frida.server/frida-agent-64.so`中存放`frida-agent`，可以查找此路径下是否存在对应文件

5、**扫描maps文件**

`maps`文件用于显示当前app中加载的依赖库
Frida在运行时会先确定路径下是否有`re.frida.server`文件夹
若没有则创建该文件夹并存放`frida-agent.so`等文件，该so会出现在`maps`文件中

注：`frida`在注入`App`后会在`maps`中显示`frida`的`frida-agent.so`的内存信息，可以通过搜索特征字符串来检测frida



**Frida-Server 为什么要放在这个目录？**

将 Frida-Server 放在 `/data/local/tmp` 目录下是为了确保它具有足够的权限运行并且可以被设备上的其他应用程序访问。

`Android `设备中的 `/data/local/tmp` 目录是一个特殊的目录，它具有较高的读写权限，并且对于应用程序来说是可访问的。因此，将 Frida-Server 放置在该目录下可以确保它能够在` Android` 设备上正常运行，并且其他应用程序可以通过执行 Frida-Server 来与目标应用程序进行通信。

此外，`/data/local/tmp` 目录下的文件在设备重启后不会被清除，这意味着 `Frida-Server `在设备重启后仍然可用，而不需要重新安装。

总之，将 `Frida-Server` 放置在 `/data/local/tmp` 目录下是为了方便其在 `Android` 设备上的部署和使用，并确保它具有足够的权限和持久性。

除了 `/data/local/tmp` 目录，您还可以将 `Frida-Server` 放置在其他路径中，只要该路径对应的目录具有足够的权限供 `Frida-Server` 运行和其他应用程序访问即可。

**修改存放路径**

以下是一些常见的可用路径：

- `/data/data/package_name`: 这是每个` Android` 应用程序的私有数据目录，其中 `package_name` 是应用程序的包名。您可以在该目录下创建一个子目录，并将 `Frida-Server `放入其中。这样做可以确保` Frida-Server `只能由该应用程序访问。
- `/sdcard`: 这是设备上的外部存储目录，通常对于大多数应用程序来说可读写的权限是允许的。您可以在该目录下创建一个子目录，并将` Frida-Server` 放入其中。但请注意，如果设备没有外部存储或者用户没有授予存储权限，那么该路径可能不适用。
- `/system/bin` 或 `/system/xbin`: 这些是 `Android` 系统的可执行文件路径，通常只有` root `权限才能访问。如果您的设备已经 `root`，您可以将 `Frida-Server` 放置在这些目录中以获得更高的权限。但请注意，在设备上修改系统文件需要小心谨慎，并且需要对系统有足够的了解。

请注意，将 `Frida-Server `放置在非标准路径下可能需要特殊的权限或额外的设置。因此，在选择其他路径之前，请确保您对设备和文件系统有足够的了解，并进行适当的测试和验证。

总而言之，除了 `/data/local/tmp` 目录，您可以根据需求将 `Frida-Server` 放置在其他具有适当权限和访问性的路径中。



**测试版本APP**

正常`Frida`调试、会提示各种检测

<img src="images\image-20231113171237647.png" alt="image-20231113171237647" style="zoom:80%;" />



下载编译得frida服务端

> 地址：https://github.com/hzzheyang/strongR-frida-android/releases?page=5

```
./frida-server-16.0.2-android-arm64  -l 0.0.0.0:8881
adb forward tcp:8881 tcp:8881
```

<img src="images\image-20231113171555407.png" alt="image-20231113171555407" style="zoom:80%;" />



