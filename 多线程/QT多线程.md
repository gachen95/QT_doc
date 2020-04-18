# QT多线程

为何要使用多线程
任何收发两端速度不一致的通讯，都需要在它们之间使用一个足够大的FIFO缓冲区。
对任何FIFO缓冲区的使用，都需要仔细考虑接收端接收时超时无数据和发送端发送时FIFO缓冲区已满这两种情况下该如何做。
这些
经典代码还包括以下必须考虑的因素：
◆跨Windows和Linux平台
◆多线程锁
◆多线程日志
◆日志文件占用的磁盘空间的可控性。
◆日志中的时间包括毫秒
◆传输的数据对应的每个字节到底的英文几
◆如何退出多线程程序

什么情况下不用线程
启动其它线程很容易，但很难保证所有共享的数据保持一致。问题通常很难找到，因为它们可能在某个时候仅显示一次或仅在某种硬件配置下出现。在创建线程解决某些问题之前，应予以考虑可能的替代方案。

线程基础
线程与并行处理任务息息相关，就像进程一样。当前CPU设计的趋势是拥有多核。一个典型的单线程应用程序只能利用一个核。但是，一个多线程程序可被分配给多个核，使得程序以一种完全并行的方式运行。这样，将一个任务分配给多个线程使得程序在多核CPU计算机上的运行速度比传统的单核CPU计算机上的运行速度快很多。

每个程序启动后就会拥有一个线程。该线程称为“主线程”（在Qt应用程序中也叫“GUI线程”）。Qt GUI必须运行在此线程上。所有的部件和几个相关的类，例如：QPixmap，不能工作于次线程中。次线程通常称为“工作者线程”，因为它主要处理从主线程中卸下的一些工作。

QThread继承自QObject
每个线程都有它自己的事件循环。初始线程通过QCoreApplication::exec()来启动它的事件循环, 或者对于单对话框的GUI应用程序，有些时候用QDialog::exec()，其它线程可以用QThread::exec()来启动事件循环。就像QCoreApplication，QThread提供一个exit(int)函数和quit()槽函数.


一个QObject实例被称为存活于它所被创建的线程中。关于这个对象的事件被分发到该线程的事件循环中。可以用QObject::thread()方法获取一个QObject所处的线程。

QObject::moveToThread()函数改变一个对象和它的孩子的线程所属性。（如果该对象有父亲的话，它不能被移动到其它线程中）。

从另一个线程（不是该QObject对象所属的线程）对该QObject对象调用delete方法是不安全的，除非你能保证该对象在那个时刻不处理事件，使用QObejct::deleteLater()更好。一个DeferredDelete类型的事件将被提交（posted），而该对象的线程的事件循环最终会处理这个事件。默认情况下，拥有一个QObject的线程就是创建QObject的那个线程，而不是QObject::moveToThread()被调用后的。

快速使用
在使用 worker-object 时，最主要的事情是要记住 QThread 不是一个线程，而是一个线程对象的包装器。这个包装器提供了信号、槽和方法，来轻松地使用 Qt 中的线程对象。

具体的使用，分为以下几步：

利用继承QObject方法创建多线程，主要的步骤有一下几点：（注意：退出线程循环后，还要调用QThread::quit()函数，该线程才会触发QThread::finished()信号）
1)：首先创建一个类MyThread，基类为QObject。

2)：在类MyThread中创建一个槽函数，用于运行多线程里面的代码。所有耗时代码，全部在这个槽函数里面运行。

3)：实例一个QThread线程对象（容器），将类MyThread的实例对象转到该容器中，用函数void QObject::moveToThread(QThread *thread);

myObjectThread->moveToThread(firstThread);
1
4)：用一个信号触发该多线程槽函数，比如用QThread::started()信号。

connect(firstThread,SIGNAL(started()),myObjectThread,SLOT(startThreadSlot()));
1
5)：用信号QThread::finished绑定槽函数QThread::deleteLatater()，在线程退出时，销毁该线程和相关资源。

connect(firstThread,SIGNAL(finished()),firstThread,SLOT(deleteLater()));
1
6)： 所有线程初始化完成后 ，启动函数QThread::start()开启多线程，然后自动触发多线程启动信号QThread::started()。

实现一个简单的 thread 类：
mythread.h

```c
ifndef MYTHREAD_H

define MYTHREAD_H

include <QObject>

class MyThread : public QObject
{
    Q_OBJECT
public:
    explicit MyThread(QObject *parent = nullptr);

    void closeThread();

signals:

public slots:
    void startThreadSlot();

private:
    volatile bool isStop;
};

endif

```


现在，看看这个类的基本实现：

mythread.cpp

```c
include "mythread.h"

include <QDebug>

include <QThread>

MyThread::MyThread(QObject *parent) : QObject(parent)
{
    isStop = false;
}

void MyThread::closeThread()
{
    isStop = true;
}

void MyThread::startThreadSlot()
{
    while (1)
    {
        if(isStop)
            return;
        qDebug()<<"MyThread::startThreadSlot QThread::currentThreadId()=="<<QThread::currentThreadId();
        QThread::sleep(1);
    }
}

```




虽然 Worker 类没做什么特别的事情，但它包含了所有必要的元素。当主函数（这里是 process()）被调用时，就开始处理数据。一旦处理完成，就会发射 finished() 信号，以触发 QThread 实例的关闭。

这里需要注意一点：永远不要在 QObject 类的构造函数中分配堆对象（使用 new），因为这个分配是在主线程上执行的，而不是在新的 QThread 实例上执行的。也就是说，新创建的对象将由主线程所拥有，而非 QThread 实例。这会使代码无法正常工作。相反，在这种情况下，应该在主函数（process()）中分配资源，因为当被调用时，对象将在新的线程实例上，因此它将拥有资源。

创建 thread 实例
现在，来看看如何创建一个 thread 实例，并把它放到一个 QThread 实例中：

widget.h

```
ifndef WIDGET_H

define WIDGET_H

include <QWidget>

include <QVBoxLayout>

include <QPushButton>

include <QThread>

include "mythread.h"

class Widget : public QWidget
{
    Q_OBJECT

public:
    Widget(QWidget *parent = 0);
    ~Widget();

    void createView();

private slots:
    void openThreadSlot();
    void closeThreadSlot();
    void finishedThreadSlot();

private:
    QVBoxLayout *mainLayout;
    QThread *firstThread;
    MyThread *myObjectThread;
};

endif // WIDGET_H

```




注意： connect() 系列是最为关键的部分。

widget.cpp

```
include <QDebug>

include "widget.h"

Widget::Widget(QWidget *parent)
    : QWidget(parent)
{
    createView();
}

Widget::~Widget()
{
}

void Widget::createView()
{
    /UI界面/
    mainLayout = new QVBoxLayout(this);
    QPushButton *openThreadBtn = new QPushButton(tr("打开线程"));
    QPushButton *closeThreadBtn = new QPushButton(tr("关闭线程"));
    mainLayout->addWidget(openThreadBtn);
    mainLayout->addWidget(closeThreadBtn);
    mainLayout->addStretch();
    connect(openThreadBtn,SIGNAL(clicked(bool)),this,SLOT(openThreadSlot()));
    connect(closeThreadBtn,SIGNAL(clicked(bool)),this,SLOT(closeThreadSlot()));
}

void Widget::openThreadSlot()
{
    /开启一条多线程/
    qDebug()<<tr("开启线程");
    firstThread = new QThread;                                                      //线程容器
    myObjectThread = new MyThread;
    myObjectThread->moveToThread(firstThread);                                      //将创建的对象移到线程容器中
    connect(firstThread,SIGNAL(finished()),myObjectThread,SLOT(deleteLater()));        //终止线程时要调用deleteLater槽函数
    connect(firstThread,SIGNAL(started()),myObjectThread,SLOT(startThreadSlot()));  //开启线程槽函数
    connect(firstThread,SIGNAL(finished()),this,SLOT(finishedThreadSlot()));
    firstThread->start();                                                           //开启多线程槽函数
    qDebug()<<"mainWidget QThread::currentThreadId()=="<<QThread::currentThreadId();
}

void Widget::closeThreadSlot()
{        
    qDebug()<<tr("关闭线程");
    if(firstThread->isRunning())
    {
        myObjectThread->closeThread();  //关闭线程槽函数
        firstThread->quit();            //退出事件循环
        firstThread->wait();            //释放线程槽函数资源
    }
}

void Widget::finishedThreadSlot()
{
    qDebug()<<tr("多线程触发了finished信号123");
}

```





当 worker 实例发出 finished() 信号时，就会通知线程退出。然后，使用相同的信号来删除 worker 实例。

最后，为了防止 crash（有可能当 thread 被删除时，还没有完全关闭），需要将 thread（非 worker）的 finished() 信号连接到其自身的 deleteLater() 槽函数。这样以来，只有线程在完全退出时，才会被删除.

线程注意
1.
许多Qt的类都是可重入的，但不是线程安全的，因为线程安全意味着为锁定与解锁一个QMutex增加额外的开销。例如：QString是可重入的，但不是线程安全的。你能够同时从多个线程访问不同的QString的实例，但不能同时从多个线程访问QString的同一个实例（除非用QMutex保护访问）。

有些Qt的类和函数是线程安全的。它们主要是线程相关类（例如：QMutex）和一些基本函数（例如： QCoreApplication::postEvent()）。

2.
如果你正在调用一个QObject子类的函数，而该子类对象并不存活于当前线程中，并且该对象是可以接收事件的，那么你必须用一个mutex保护对该QObject子类的内部数据的所有访问，否则，就有可能发生崩溃和非预期的行为。

同其它对象一样，QThread对象存活于该对象被创建的线程中 – 而并非是在QThread::run()被调用时所在的线程。一般来说，在QThread子类中提供槽函数是不安全的，除非用一个mutex保护成员变量。

参考文章-欢迎点击
1.作者：三公子Tjq https://blog.csdn.net/naibozhuan3744/article/details/81201502
2.作者：一去丶二三里 https://blog.csdn.net/liang19890820/article/details/50277095
————————————————
版权声明：本文为CSDN博主「雪山飞狐W」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_41429220/article/details/97631999





QML界面卡顿时也可以采用线程的方法。下面提供二个思路：

1.Qt提供了一个WorkerScript来将脚本中的执行的函数放到一个线程中执行。WorkerScript 用于生成新的线程,并通过消息进行通信。自行查看Qt的帮助文档，有相关例子。

网站https://doc.qt.io/qt-5/qtquick-threading-example.html

2.采用C++中的QThread处理相关操作，完成之后利用信号槽通知QML组件。



# QML中调用C++耗时操作造成阻塞的解决办法



在QML中经常会调用用C++写的比较耗时的操作时，一般会造成界面的卡死。刚开始的时候是想着是不是可以在QML中开辟新线程，一查还真有，WorkerScript。但这玩意儿有点坑的是你不能访问其他对象的属性、方法，官方原文是这样写的：
  Since the WorkerScript.onMessage() function is run in a separate thread, the JavaScript file is evaluated in a context separate from the main QML engine. This means that unlike an ordinary JavaScript file that is imported into QML, the script.js in the above example cannot access the properties, methods or other attributes of the QML item, nor can it access any context properties set on the QML object through QQmlContext.
这样的话很多地方就会受限了。后来看了安晓辉老师的Qt Quick核心编程，里面有一章节讲的C++与QML混合编程，其中有一个例子讲的是图像处理，他其实是使用信号槽的方式将QML中的同步操作改成了异步操作，所以我后来的解决办法是这样的。
  创建一个C++桥接类并注册到QML中，该类用于和QML进行交互。在该类中将比较耗时的操作放入单独的线程进行运行。逻辑顺序是这样的，QML中 先执行桥接类的触发执行信号的函数，并进行必要的传参；桥接类发出执行信号后，相应的处理线程启动，处理线程处理完毕后，会发出信号，桥接类收到信号后也会发出完成信号。此时，在QML中做好连接（Connections），收到完成信号后，将结果传到QML中，这样的话整个耗时的操作就会在C++中执行，同时也不会阻塞主线程。
————————————————
版权声明：本文为CSDN博主「鬼马行天」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/guimaxingtian/java/article/details/84064449