# [Qt5 With Log4qt](https://endle.github.io/2015/02/20/qt5-with-log4qt/)

layout: post title: “Qt5 中使用 log4qt 输出日志” description: “” category: project tags: [C++, QT] —

最近在写一个 QT 的程序，需要打印日志。搜索了一下，选定了 [Log4Qt](http://log4qt.sourceforge.net/)。不过，原项目已经长期不更新了，而 [DevBean](http://www.devbean.net/) 维护了一个 [Qt5 的版本](https://github.com/devbean/log4qt)。所以，我执行了 `git submodule add https://github.com/devbean/log4qt.git` 而在此同时，另一个团队也在孜孜不倦地 [维护着 Log4qt](https://gitorious.org/log4qt)。感兴趣的朋友不妨也测试一下。

官方提供了一个比较详细的[使用介绍](http://log4qt.sourceforge.net/)，但也隐藏了几个小坑，整理如下：

#### Step 1

将这句话加入 .pro 文件即可 `include(/src/log4qt/log4qt.pri)`

#### Step 2

在预编译头文件里加入

```
#include "log4qt/consoleappender.h"
#include "log4qt/logger.h"
//我添加的内容
#include "log4qt/logmanager.h"
#include "log4qt/patternlayout.h"
//被我移除的头文件
//#include "log4qt/ttcclayout.h"
```

#### Step 3

初始化 Log4Qt，我是在 `QApplication::exec()` 之前执行的这段代码 为了能灵活地处理 Layout，我小幅修改了原有的代码。

```
//Configure a logger to generate output. The example uses the root logger
//Create a layout
Log4Qt::LogManager::rootLogger();

//TTCCLayout *p_layout = new TTCCLayout();
//使用 PatternLayout，自定义输出格式
p_layout = new Log4Qt::PatternLayout("%-5p [%c] %m%n");
p_layout->setName(QLatin1String("My Layout"));
p_layout->activateOptions();

// Create an appender
ConsoleAppender *p_appender = new ConsoleAppender(p_layout, ConsoleAppender::STDOUT_TARGET);
p_appender->setName(QLatin1String("My Appender"));
p_appender->activateOptions();
// Set appender on root logger
Log4Qt::Logger::rootLogger()->setAppender(p_appender);
```

关于 Layout 的详细内容，可以参考 [Log4j 的一篇文档](https://logging.apache.org/log4j/1.2/apidocs/org/apache/log4j/PatternLayout.html)，使用方法与 C 的格式化字符串是一样的。

#### Step 4

然后，就可以直接输出 log 了，例如官方示例
`Log4Qt::Logger::logger(QLatin1String("My Logger"))->info("Hello World!");`

为了简化代码，我在预编译头文件里又加入了这段内容

```
#define INITIATE_LOG4QT(ClassName) LOG4QT_DECLARE_STATIC_LOGGER(LOG4QT_LOGGER, ClassName)

#define ERROR(s) LOG4QT_LOGGER()->error(QString(s))
#define WARN(s) LOG4QT_LOGGER()->warn(QString(s))
#define INFO(s) LOG4QT_LOGGER()->info(QString(s))
```

而在定义类的头文件里，只要初始化一下

```
class MyClass
{
private:
    INITIATE_LOG4QT(MyClass);

public:
    MyClass();
    ~MyClass();
};
```

就可以直接在类的成员函数里用 `ERROR()` 这样的宏了。

#### Step 5

Log4Qt 的功能非常强大。例如，四个 Level 用于输出不同的信息，可以在启动时动态确定输出哪些信息。同样，也可以使用条件编译，在 Release 里将 `INFO` 编译成 `(void)0` 来减少性能损失。不过，Log4Qt 的性能没有测试，希望了解的朋友贴出自己的经验 :-)

#### Last

从我学 Qt 的第一天起就没少看 DevBeans 的博客，也在这里推荐一下他的书籍 [《Qt 5编程入门》](http://www.amazon.cn/gp/product/B00SALSVVG/ref=as_li_ss_tl?ie=UTF8&camp=536&creative=3132&creativeASIN=B00SALSVVG&linkCode=as2&tag=blo-23)

