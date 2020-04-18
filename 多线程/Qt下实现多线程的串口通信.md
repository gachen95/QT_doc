# Qt下实现多线程的串口通信

简述
Qt下无论是RS232、RS422、RS485的串口通信都可以使用统一的编码实现。本文把每路串口的通信各放在一个线程中，使用movetoThread的方式实现。

代码之路
用SerialPort类实现串口功能，Widget类调用串口。
serialport.h如下

```c
include <QObject>

include <QSerialPort>

include <QString>

include <QByteArray>

include <QObject>

include <QDebug>

include <QObject>

include <QThread>

class SerialPort : public QObject
{
  Q_OBJECT
public:
  explicit SerialPort(QObject *parent = NULL);
  ~SerialPort();

  void init_port();  //初始化串口

public slots:
  void handle_data();  //处理接收到的数据
  void write_data();     //发送数据

signals:
  //接收数据
  void receive_data(QByteArray tmp);

private:
  QThread *my_thread;
  QSerialPort *port;
};

```




serailport.cpp如下

```c
include "serialport.h"

SerialPort::SerialPort(QObject *parent) : QObject(parent)
{
    my_thread = new QThread();

    port = new QSerialPort();
    init_port();
    this->moveToThread(my_thread);
    port->moveToThread(my_thread);
    my_thread->start();  //启动线程

}

SerialPort::~SerialPort()
{
    port->close();
    port->deleteLater();
    my_thread->quit();
    my_thread->wait();
    my_thread->deleteLater();
}

void SerialPort::init_port()
{
    port->setPortName("/dev/ttyS1");                   //串口名 windows下写作COM1
    port->setBaudRate(38400);                           //波特率
    port->setDataBits(QSerialPort::Data8);             //数据位
    port->setStopBits(QSerialPort::OneStop);           //停止位
    port->setParity(QSerialPort::NoParity);            //奇偶校验
    port->setFlowControl(QSerialPort::NoFlowControl);  //流控制
    if (port->open(QIODevice::ReadWrite))
    {
        qDebug() << "Port have been opened";
    }
    else
    {
        qDebug() << "open it failed";
    }
    connect(port, SIGNAL(readyRead()), this, SLOT(handle_data()), Qt::QueuedConnection); //Qt::DirectConnection
}

void SerialPort::handle_data()
{
    QByteArray data = port->readAll();
    qDebug() << QStringLiteral("data received(收到的数据):") << data;
    qDebug() << "handing thread is:" << QThread::currentThreadId();
    emit receive_data(data);
}

void SerialPort::write_data()
{
    qDebug() << "write_id is:" << QThread::currentThreadId();
    port->write("data", 4);   //发送“data”字符
}

```


widget.h的调用代码

```
include "serialport.h"

public slots:
  void on_receive(QByteArray tmpdata);
private:
  SerialPort *local_serial;

```


widget.cpp调用代码

```
local_serial = new SerialPort();
connect(ui->pushButton, SIGNAL(clicked()), local_serial, SLOT(write_data())，Qt::QueuedConnection);
connect(local_serial, SIGNAL(receive_data(QByteArray)), this, SLOT(on_receive(QByteArray)), Qt::QueuedConnection);
//on_receive槽函数
void Widget::on_receive(QByteArray tmpdata)
{
 ui->textEdit->append(tmpdata);
}
```




写在最后
本文例子实现的串口号是 /dev/ttyS1（对应windows系统是COM1口），波特率38400，数据位8，停止位1，无校验位的串口通信。当然，使用串口程序前，需要在.pro文件中添加 QT += serialport，把串口模块加入程序。

点赞 5
————————————————
版权声明：本文为CSDN博主「lusanshui」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/lusanshui/article/details/84869423



mythread.h

```c

#ifndef MYTHREAD_H
 
#define MYTHREAD_H
 
#include <QObject>
 
#include <QDebug>
 
#include <QThread>
 
#include "qextserialbase.h"
 
#include "win_qextserialport.h"
 
#include <QPushButton>
 
class MyThread : public QThread
 
{
 
        Q_OBJECT
 
public:
 
    MyThread(QString port1);
 
    void run();
 
signals:
 
    void setsenddata(float data);//向主线程发送接收到的串口数据
 
public slots:
 
    void read_serial_data();//读取串口数据
 
    void close_mthread_serial(char *st);//关闭
 
private:
 
    Win_QextSerialPort *mycom;
 
    QString port;
 
    float data;
 
};
 
#endif // MYTHREAD_H
```

mythread.cpp

```c
#include "mythread.h"
 
 
MyThread::MyThread(QString port1)
 
{
 
    port = port1;
 
}
 
void MyThread::run()
 
{
 
    //重写run()函数初始化串口
 
    mycom = new Win_QextSerialPort(port,QextSerialBase::EventDriven);//读取串口采用事件驱动模式
 
    mycom->open(QIODevice::ReadWrite);//读写方式打开
 
    mycom->setBaudRate(BAUD115200);//波特率
 
    mycom->setDataBits(DATA_8);//数据位
 
    mycom->setParity(PAR_NONE);//奇偶校验
 
    mycom->setStopBits(STOP_1);//停止位
 
    mycom->setFlowControl(FLOW_OFF);//控制位
 
    mycom->write("1");//向下位机发送1告诉它开始发送数据
 
    connect(mycom,SIGNAL(readyRead()),this,SLOT(read_serial_data()));//有数据就读
 
}
 
void MyThread::read_serial_data()
 
{
 
    if(mycom->bytesAvailable()>= 9)
 
    {
 
        qDebug()<<mycom->bytesAvailable() << endl;
 
        QByteArray temp;
 
        temp = mycom->read(9);//每串数据为9个字节
 
          union dat{
 
            char a[4];
 
            int x;
 
        };
 
        dat data_serial;
 
        data_serial.a[0] = 0;
 
        data_serial.a[1] = temp[8];
 
        data_serial.a[2] = temp[7];
 
        data_serial.a[3] = temp[6];
 
        data = data_serial.x*2.4/256/8288607*10000;//提取有效数据位，根据需要的数据设计的算法
 
        emit setsenddata(data);//发送数据给主线程
 
        qDebug() <<data ;
 
    //TB->insertPlainText(temp.toHex()+"  ");
 
      }
 
}
 
void MyThread::close_mthread_serial(char *st)
 
{
 
    mycom->write(st);//告知下位机读端已关闭
 
    qDebug() <<"close ok"<<st<<endl;
 
    mycom->close();//关闭子线程
 
}

 
```