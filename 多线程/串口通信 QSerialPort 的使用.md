# 串口通信 QSerialPort 的使用

在qt5中，使用QSerialPort进行串口通信。主要用到的QSerialPort

使用步骤

步骤一：在.pro文件中进行声明

```
QT += serialport         //在.pro文件中添加这个声明
```


步骤二：实例化 QSrerialPort

```
QSerialPort *serial=new QSerialPort   //实例化QSerialPort 并且开辟空间
```


步骤三：寻找显示目前的所有串口信息

```
foreach(const QSerialPortInfo &portinfo,QSerialPortInfo::availablePorts())
     {
         serial->setPort(portinfo);       //此处的serial是在构造函数中QSerialPort 的对象；
         qDebug()<<serial->portName();
    }  
//第一步：setPort(portinfo)方法，将serial绑定到了其中一个接口上，
//第二步：利用portname将这个串口信息进行查询
```



步骤四：打开串口

     serial->setPortName("COM7");  //设置串口名称，在第三步，已经能够获取所有的串口信息
     qDebug()<<"打开串口状态"<<serial->open(QIODevice::ReadWrite);//利用open方法，传入参数：打开方式   。并返回打开操作状态，成功打开串口返回true，没有打开串口返回false
步骤五：串口协议设定

```c
void GetData::Init(int baurate, int flow_ctrl, int databits, int stopbits, int parity){
                 //  波特率        数据控制流        数据位          停止位     奇偶校验
  GetData::Open();//如果串口已经打开过，这个可以省略
  //设置波特率
  switch(baurate) //判断波特率，并进行设置。需要注意一点，波特率的值可以直接传入，也可以用qt给的枚举
  {
  case 1200:
      serial->setBaudRate(QSerialPort::Baud1200);
      break;
  case 2400:
      serial->setBaudRate(QSerialPort::Baud2400);
      break;
  case 4800:
      serial->setBaudRate(QSerialPort::Baud4800);
      break;
  case 9600:
      serial->setBaudRate(QSerialPort::Baud9600);
      break;
  case 19200:
      serial->setBaudRate(QSerialPort::Baud19200);
      break;
  case 38400:
      serial->setBaudRate(QSerialPort::Baud38400);
      break;
  case 57600:
      serial->setBaudRate(QSerialPort::Baud57600);
      break;
  case 1382400:
     qDebug()<<"设置波特率状态"<< serial->setBaudRate(1382400);//QSerialPort::Baud115200);
      break;
  default :
      serial->setBaudRate(baurate);
      break;

  }
  //设置数据控制流
  switch (flow_ctrl)   //注意，这里之所以进行判断，是因为QSerialport类中给了枚举，并且设置方法中只能传入枚举参数
      {
      case 0://不使用流控制
          qDebug()<<"流控制设置状态"<<serial->setFlowControl(QSerialPort::NoFlowControl);
          break;
      case 1://使用硬件流控制
          serial->setFlowControl(QSerialPort::HardwareControl);
          break;
      case 2://使用软件流控制
          serial->setFlowControl(QSerialPort::SoftwareControl);
          break;
      default:
          serial->setFlowControl(QSerialPort::UnknownFlowControl);
      }
  //设置数据位
  switch (databits)
  {
  case 5:
     serial->setDataBits(QSerialPort::Data5);
      break;
  case 6:
      serial->setDataBits(QSerialPort::Data6);
      break;
  case 7:
     serial->setDataBits(QSerialPort::Data7);
      break;
  case 8:
      qDebug()<<"设置数据位状态"<<serial->setDataBits(QSerialPort::Data8);
      break;
  default:
      serial->setDataBits(QSerialPort::UnknownDataBits);
     break;
  }
  //设置奇偶效验
  switch (parity)
      {
      case 'n':
      case 'N': //无奇偶校验位。
          serial->setParity(QSerialPort::NoParity);
          break;
      case 'o':
      case 0://设置为奇校验
           qDebug()<<"设置校验位状态"<<serial->setParity(QSerialPort::OddParity);
          break;
      case 'e':
      case 'E'://设置为偶校验
          serial->setParity(QSerialPort::EvenParity);
          break;
      default:
          serial->setParity(QSerialPort::UnknownParity);
          break;
      }
  //设置停止位
  switch(stopbits)
  {
      case 1:  qDebug()<<"设置停止位状态"<<serial->setStopBits(QSerialPort::OneStop); break;
      case 2: serial->setStopBits(QSerialPort::TwoStop); break;
      default : serial->setStopBits(QSerialPort::UnknownStopBits); break;
  }

qDebug()<<"串口初始化执行完成";

}

```



```

```

步骤六：读取数据

```
void getdata(){ //这个方法是负责接收这个数据
  QByteArray buf_data;        //注意，接收数据时，qByteArray是最常用的类
  qDebug()<<"开始接受数据";    
  buf_data=serial->readAll(); //接收数据
  qDebug()<<"数据接受完成";
  qDebug()<<buf_data.size();
}
```





connect(serial,SIGNAL(readyRead()),this,SLOT(getdata()));//这是串口信息判断，如果有这个信号发出来，调用getdata接收数据。这里需要注意的是，readyread信号发出，不代表你可以一次性把你所发的数据都接收过来。例如：你发一个字符串“123456789”,它可能会发出两个readyread信号，调用两次getdata方法来获取数据

步骤七：关闭数据

```
int  GetData::Close(){
  serial->clear(); //清除缓存区
  serial->close();//关闭串口
  return 0;
}
```


总结：这里暂时没有介绍怎样发数据，而且在数据接受处理上也有很大缺陷，需要后续改善，所以未完待续。。。。
————————————————
版权声明：本文为CSDN博主「智慧小人儿」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/JimBraddock/article/details/83345071