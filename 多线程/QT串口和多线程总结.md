# QT串口和多线程总结

Qt的串口个人感觉不是很好用。

大体使用步骤如下：

1.在.pro文件中加入 

```
QT+=serialport
```

2.添加头文件

```
include<QtSerialPort/qserialport>

include<QtSerialPort/QSerialPortInfo>

```



3.编写收发程序

write和readAll 方法实现读写程序



**QT 读写可分为阻塞方式和非阻塞方式**

#### 阻塞方式：通过查询缓冲区是否有数据

waitForReadyRead(int timeout) 当缓冲区有一个字节数据的时候此函数就会返回 true;timeout为最大等待超时时间,若timeout为-1 则阻塞到有数据为止。

waitForBytesWriten(int timenout);当数据全部写完时返回1;

bytesAvailable(); 返回缓冲区已有的字节数;

使用这三个方法可实现阻塞方式读写串口;

Example:

if(Mpor->waitForReadyRead(300))

{

QByteArray readbuff;

readbuff.append(Mport->readAll());

while(Mpor->waitForReadyRead(20))

readbuff.append(Mport->readAll());

}

###非阻塞式：分两种一、在子线程中读写串口；二、使用qt的信号槽机制

##### 在子线程中读写：

QThread 类的run方法中实现读写即可；

Example：

class MyThread:public QThread

{

Qserialport *Mport;

...

public void run();

}

重写run()方法

在run方法中new Qserialport对象，初始化Mport，然后读写串口。

#### 信号槽机制：

如果是在主线程中完成则只需要

connect(sender,signal,receiver,slot);即可; sender 和receiver 一般是本对象（this）

如果在子线程中则：

Qserialport对象必须在子线程中创建，而且最好自己手动 emit signal;

void run()

{

new Qserialport；

Init();

connect(sender,signal,receiver,slot);

Mwrite()

{

Mport->write(bytesarray);

emit msignal();

}

}



注意：

子类方法中使用connect时 需要在子类中 声明 Q_OBJECT  然后重新运行qmake(构建->执行qmake) 

写的不够完善，大体意思如此，不完善的地方有待学习.....
————————————————
版权声明：本文为CSDN博主「随便djy」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/u012232736/article/details/78969005







使用movetothread方式。
```C

	ySerialPort = new YSerialPort();
  	serialPortThread = new QThread();

    ySerialPort->moveToThread(serialPortThread);
    serialPortThread->start();
    connect(serialPortThread,SIGNAL(started()),ySerialPort,SLOT(init()));
    connect(serialPortThread, SIGNAL(finished()), ySerialPort, SLOT(deleteLater()));
```
创建基于QObject的类用来操作串口。
在你构造这个类的地方，也构造一个QThread，将类对象moveToThread(thread)
调用thread->start()开启线程。
信号槽，start信号触发串口类的init槽，用来操作串口。

使用信号槽，信号在一个线程，槽在另一个线程执行。
也就是说start在主线程执行，init在串口类行程执行。
（使用信号槽的lambda表达式，都在信号所做线程执行）

串口类中的所有操作都通过信号槽和外部交互。
————————————————
版权声明：本文为CSDN博主「Y_Hanxiao」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/Y_Hanxiao/article/details/90437725