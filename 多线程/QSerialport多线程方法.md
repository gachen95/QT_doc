# QSerialport多线程方法



   使用Qt也已经有一段时间了，虽然使用过继承QThread重写run函数，以及继承QObject然后使用MoveToThread两种方法实现多线程，但是在QSerialPort的使用过程中，两种方法都存在一定的问题。

典型的问题：QObject: Cannot create children for a parent that is in a different thread.

    原因：在主线程中创建了QSerialPort对象在子线程中调用，或者在子线程中创建然后在主线程中调用了。

对于继承QThread重写run函数的情况，往往容易在run外部定义QSerialport *port = new QSerialport()对象，然后在run中调用port->readAll()等函数，然而根据QThread的特性，只有run函数才运行在新的子线程中，所以这里就跨线程调用了 QSerialport对象，会出现上述报错

已有解决办法：
    要解决这样的问题，可以借鉴https://blog.csdn.net/u010842812/article/details/28898761这篇博文使用的方法：派生类外部定义QSerialPort指针，在run函数中再定义对象，并且数据的读、写都在run函数中进行。

    上述方法可以解决报错，但是针对不仅仅具有读串口数据需求、还需要向串口写数据的情况，显然这样的方法会让run函数变得十分臃肿，代码的维护十分麻烦。

推荐办法：
    运用继承QObject，结合MoveToThread()的方式：

    自定义my_serial类，继承自QObject，将一个QThread local_thread 对象作为派生类的成员，使用QObject的MoveToThread()将派生类自己以及QSerialPort对象都移动到local_thread线程中，这样派生类中的槽函数以及QSerialPort中的信号与槽函数都将在local_thread线程执行。头文件：

```
include <QObject>

include <QSerialPort>

include <QThread>

include <QString>

include <QByteArray>

include <QDebug>

class my_qserial : public QObject
{
    Q_OBJECT
public:
    explicit my_qserial(QObject *parent = nullptr);
    void show_func_id();
    void init_port();
    QSerialPort *port;
signals:

public slots:
    void show_slots_id();
    void handle_data();
    void write_data();
signals:
    void thread_sig();
private:
    QThread *my_thread;
};

```





其中，show_func_id()是定义在public中的函数，用于显示当前线程编号；show_slots_id()是定义在slots中函数，也是用于显示当前函数的运行线程编号，handle_data()用于响应数据的读，write_data用于响应数据的写，thread_sig()用于测试多线程信号的发射。

具体实现如下.cpp文件：

```
include "my_qserial.h"

my_qserial::my_qserial(QObject *parent) : QObject(parent)
{
    my_thread = new QThread();
    show_func_id();
    show_slots_id();
    port = new QSerialPort();
    init_port();
    this->moveToThread(my_thread);
    port->moveToThread(my_thread);
    my_thread->start();//开启多线程
    qDebug() << "in main thread";
}

void my_qserial::show_func_id()
{
    qDebug() << "func_id is:" << QThread::currentThreadId();
}

void my_qserial::show_slots_id()
{
    qDebug() << "slots_id is:" << QThread::currentThreadId();
    show_func_id();
}

void my_qserial::init_port()
{
    port->setPortName("COM1");
    port->setBaudRate(921600);
    port->setDataBits(QSerialPort::Data8);
    port->setParity(QSerialPort::NoParity);
    port->setStopBits(QSerialPort::OneStop);
    port->setFlowControl(QSerialPort::NoFlowControl);
    if(port->open(QIODevice::ReadWrite))
    {
        qDebug() << "Port have been opened";
    }
    else
    {
       qDebug() << "open it failed";
    }
    connect(this->port,SIGNAL(readyRead()),this,SLOT(handle_data()),Qt::DirectConnection);
}

void my_qserial::handle_data()
{
    QByteArray data = port->readAll();
    qDebug() << "data received:" << data;
    qDebug() << "handling thread is:" << QThread::currentThreadId();
}

void my_qserial::write_data()
{
     qDebug() << "write_id is:" << QThread::currentThreadId();
    port->write("data",4);
    //emit(thread_sig());
}

```





main函数定义如下：

```
include "mainwindow.h"

include "ui_mainwindow.h"

MainWindow::MainWindow(QWidget *parent) :
    QMainWindow(parent),
    ui(new Ui::MainWindow)
{
    ui->setupUi(this);
    local_serial = new my_qserial();
    connect(ui->pushButton,SIGNAL(clicked()),local_serial,SLOT(show_slots_id()),Qt::QueuedConnection);
    qDebug() << "main thread:" << QThread::currentThreadId();
    connect(ui->pushButton_2,SIGNAL(clicked()),this,SLOT(handle_msg()));
    connect(ui->pushButton_2,SIGNAL(clicked()),this->local_serial,SLOT(write_data()));
    connect(this->local_serial,SIGNAL(thread_sig()),this,SLOT(sig_slot()),Qt::DirectConnection);
 }

MainWindow::~MainWindow()
{
    delete ui;
}

void MainWindow::handle_msg()
{
    qDebug() << "handle2 thread" << QThread::currentThreadId();
}

void MainWindow::sig_slot()
{
    qDebug() << "thread_sig thread" << QThread::currentThreadId();
}

```





启动程序，可以看到，my_serial中的函数现在全部运行在主线程中，连slots中函数也不例外！！这是因为函数都是在主线程中调用的，并没有用到Qt的事件循环机制，下面我点击signal按钮，也就是ui->pushbutton：



这个时候，函数执行的线程发生了变化，进入到子线程中了！再试试发送一下数据，点击write也就是ui->pushbutton2：



可以看到具体写数据发生自0x1518线程中，主线程为0x2fec。

再试试数据的接收：向串口发送数据十六进制“24 00 02 00 00 02 55”，可以看出数据大接收也发生在子线程中！



okay.Over~~希望对大家有启发。

另外，注意connect()函数的第五个参数，Qt::Direcconnection,Qt::Queuenconnection以及其他的，不同的连接方式，会影响槽数执行所在的线程，https://blog.csdn.net/qq_16952303/article/details/51585577，这篇博文描述很细致。撒花~
————————————————
版权声明：本文为CSDN博主「RaoJohn」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/RaoJohn/article/details/82969848