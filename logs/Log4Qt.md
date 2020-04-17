## Log4Qt



### 为什么选择 Log4Qt

在《第01课：C++ 日志框架》一文中，我们介绍了一些主流的 C++ 日志框架。至于选择哪一个，可以参考该文中的“日志选择标准”小节内容。

对于很多人来说，第一选择会是 Log4cpp、log4cplus、log4cxx、Log4Qt 中的一个，因为它们均移植自 Java 中著名的日志框架——Log4j，并且保持了 API 上的一致。

![image](11e8-936a-bd63663e31c4)

通过使用 Log4j，我们可以：

- 控制日志的输出格式；
- 通过定义日志信息的级别，来更加细致地控制日志的生成过程；
- 控制日志信息的输出位置，例如：文件、控制台、数据库等；
- ……

最不可思议的是，这些都可以通过配置文件来灵活地控制，而无需修改源代码。

使用 Log4j 系列的最大好处在于，只要熟练掌握其中一个，其它的几个都可以信手拈来。即使是不同的语言环境，配置也不会有太大区别，并且使用方式也相当简单。

衍生品这么多，最终我选择了 Log4Qt，原因很简单：我的第一身份是 Qter。其实选择哪个都无所谓，因为它们是一家，而我们主要是学习 Log4j 的思想。

### Log4Qt 简介

Log4Qt 是 Apache Log4j 的 Qt 移植版，主要用于记录日志。

由于 Log4Qt 是基于 Qt 编写的，所以它也继承了 Qt 的跨平台特性。也就是说，可以将其用于 Windows、Linux、以及 MacOS 平台上。

- 主页：请访问[这里](http://log4qt.sourceforge.net/)。
- 文档：请访问[这里](http://log4qt.sourceforge.net/html/index.html)。

由于 Log4Qt 的开发在 2009 年就终止了，所以其官网提供的源码仅支持 Qt4：

- for Qt4：点击[这里下载](https://sourceforge.net/projects/log4qt/)。

但值得庆祝的是，有人提供了一个兼容 Qt5 的版本：

- for Qt5：点击[这里下载](https://github.com/MEONMedical/Log4Qt)。

这个升级版很棒，不但可以将 Log4Qt 源码添加至项目中，而且还可以将其编译为库，并且它还同时支持 CMake 和 qmake。

最重要的是，它还在持续升级，并且在老版本（for Qt4）的基础上添加了很多新 Feature。

### Log4Qt 分层架构

Log4Qt API 设计为分层结构，其中每一层提供了执行不同任务的不同对象，这种设计为未来的发展提供了很好的可扩展性。

Log4Qt 对象分为：

- 核心对象：使用框架必不可少的强制性对象。
- 支持对象：帮助核心对象完成重要的任务。

![image](9def-35cb75b6091c)

**核心对象：**

- Logger 对象：处于最上层，负责捕获日志信息。
- Appender 对象：负责将日志信息输出到各种目的地，例如：控制台、文件、数据库等。
- Layout 对象：用于控制日志输出格式，该层有助于以可读形式记录信息。

**支持对象：**

- Level 对象：定义日志信息的优先级：TRACE < DEBUG < INFO < WARN < ERROR < FATAL。
- LogManager：负责从配置文件或配置类中读取初始配置参数。
- Filter 对象：用于分析日志信息，并进一步决定是否需要记录信息。
- ObjectRenderer：用于向传递的各种 Logger 对象提供字符串表示（在 Log4Qt 中，尚未用到）。

### 编译 Log4Qt

下载 Log4Qt（for Qt5），进行解压缩，目录结构如下所示：

![image](43197c8c6da2)

可以看到，同时支持 CMake 和 qmake。其中，src 是需要特别关注的目录，里面包含了 Log4Qt 的所有源码。

由于代码结构已经组织好了，所以编译 Log4Qt（以 Windows 为例）非常简单：

- 使用 Qt Creator 打开：`log4qt.pro`。
- 执行 qmake -> 构建。

成功之后，在构建目录下会生成 log4qt.lib、log4qt.dll 以及相应的示例程序。

### 配置 Log4Qt

要使用 Log4Qt，有两种方式：将 Log4Qt 源码添加至项目中，或者将其当做第三方库来使用。

#### 将 Log4Qt 源码添加至项目中

在 `Log4Qt-master/src/log4qt` 目录中，有一个很重要的文件——log4qt.pri，通过它可以很容易地将 Log4Qt 的源码添加至项目中。

仅需在 `.pro`（自己的工程文件）添加如下配置：

```Qt
# 定义所需的宏
DEFINES += LOG4QT_LIBRARY

# 定义 Log4Qt 源码根目录
LOG4QT_ROOT_PATH = $$PWD/../Log4Qt-master

# 指定编译项目时应该被搜索的 #include 目录
INCLUDEPATH += $$LOG4QT_ROOT_PATH/src \
               $$LOG4QT_ROOT_PATH/src/log4qt \
               $$LOG4QT_ROOT_PATH/include \
               $$LOG4QT_ROOT_PATH/include/log4qt

# 将 Log4Qt 源代码添加至项目中
include($$LOG4QT_ROOT_PATH/src/log4qt/log4qt.pri)
include($$LOG4QT_ROOT_PATH/build.pri)
include($$LOG4QT_ROOT_PATH/g++.pri)
```

#### 将 Log4Qt 作为第三方库使用

上面说过，编译 Log4Qt 后，会生成相应的库。所以如果想将其作为第三方库来使用的话，也很方便。

仅需在 `.pro`（自己的工程文件）添加如下配置：

```Qt
# 定义 Log4Qt 源码根目录
LOG4QT_ROOT_PATH = $$PWD/../Log4Qt-master

# 指定链接到项目中的库列表
LIBS += -L$$PWD/../Libs -llog4qt

# 指定编译项目时应该被搜索的 #include 目录
INCLUDEPATH += $$LOG4QT_ROOT_PATH/src \
               $$LOG4QT_ROOT_PATH/src/log4qt \
               $$LOG4QT_ROOT_PATH/include \
               $$LOG4QT_ROOT_PATH/include/log4qt
```

