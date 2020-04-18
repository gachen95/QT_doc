# QT多线程编程详解

## 一、**线程基础**

### **1、GUI线程与工作线程**

每个程序启动后拥有的第一个线程称为主线程，即GUI线程。QT中所有的组件类和几个相关的类只能工作在GUI线程，不能工作在次线程，次线程即工作线程，主要负责处理GUI线程卸下的工作。

### 2、**数据的同步访问**

每个线程都有自己的栈，因此每个线程都要自己的调用历史和本地变量。线程共享相同的地址空间。

## **二、QT多线程简介**

  QT通过三种形式提供了对线程的支持，分别是平台无关的线程类、线程安全的事件投递、跨线程的信号-槽连接。

  QT中线程类包含如下：

 QThread 提供了跨平台的多线程解决方案

 QThreadStorage 提供逐线程数据存储
  QMutex 提供相互排斥的锁，或互斥量
  QMutexLocker 是一个辅助类，自动对 QMutex 加锁与解锁
  QReadWriterLock 提供了一个可以同时读操作的锁
  QReadLocker与QWriteLocker 自动对QReadWriteLock 加锁与解锁
  QSemaphore 提供了一个整型信号量，是互斥量的泛化
  QWaitCondition 提供了一种方法，使得线程可以在被另外线程唤醒之前一直休眠。

## **三、QThread线程**

### **1、****QThread****线程基础**

  QThread是Qt线程中有一个公共的抽象类，所有的线程类都是从QThread抽象类中派生的，需要实现QThread中的虚函数run(),通过start()函数来调用run函数。

  void run（）函数是线程体函数，用于定义线程的功能。

  void start（）函数是启动函数，用于将线程入口地址设置为run函数。

  void terminate（）函数用于强制结束线程，不保证数据完整性和资源释放。

  QCoreApplication::exec()总是在主线程(执行main()的线程)中被调用，不能从一个QThread中调用。在GUI程序中，主线程也称为GUI线程，是唯一允许执行GUI相关操作的线程。另外，必须在创建一个QThread前创建QApplication(or QCoreApplication)对象。

  当线程启动和结束时，QThread会发送信号started()和finished()，可以使用isFinished()和isRunning()来查询线程的状态。

  从Qt4.8起，可以释放运行刚刚结束的线程对象，通过连接finished()信号到QObject::deleteLater()槽。 
  使用wait()来阻塞调用的线程，直到其它线程执行完毕（或者直到指定的时间过去）。

  静态函数currentThreadId()和currentThread()返回标识当前正在执行的线程。前者返回线程的ID，后者返回一个线程指针。

  要设置线程的名称，可以在启动线程之前调用setObjectName()。如果不调用setObjectName()，线程的名称将是线程对象的运行时类型（QThread子类的类名）。

### **2、线程的优先级**

  QThread线程总共有8个优先级

  QThread::IdlePriority  0 scheduled only when no other threads are running.

  QThread::LowestPriority 1 scheduled less often than LowPriority.

  QThread::LowPriority  2 scheduled less often than NormalPriority.

  QThread::NormalPriority 3 the default priority of the operating system.

  QThread::HighPriority  4 scheduled more often than NormalPriority.

  QThread::HighestPriority 5 scheduled more often than HighPriority.

  QThread::TimeCriticalPriority 6 scheduled as often as possible.

  QThread::InheritPriority  7 use the same priority as the creating thread. This is the default.

  void setPriority(Priority priority) 
  设置正在运行线程的优先级。如果线程没有运行，此函数不执行任何操作并立即返回。使用的start()来启动一个线程具有特定的优先级。优先级参数可以是QThread::Priority枚举除InheritPriortyd的任何值。

### **3、线程的创建**

  **void** start ( Priority *priority* = InheritPriority )

  启动线程执行，启动后会发出started ()信号

### **4、线程的执行**

int exec() [protected] 
  进入事件循环并等待直到调用exit()，返回值是通过调用exit()来获得，如果调用成功则返回0。

void run() [virtual protected] 
  线程的起点，在调用start()之后，新创建的线程就会调用run函数，默认实现调用exec()，大多数需要重新实现run函数，便于管理自己的线程。run函数返回时，线程的执行将结束。

### **5、线程的退出**

void quit();

通知线程事件循环退出，返回0表示成功，相当于调用了QThread::exit(0)。

void exit ( int returnCode = 0 );

调用exit后，thread将退出event loop，并从exec返回，exec的返回值就是returnCode。通常returnCode=0表示成功，其他值表示失败。

void terminate ();

  结束线程，线程是否立即终止取决于操作系统。

  线程被终止时，所有等待该线程Finished的线程都将被唤醒。

  terminate是否调用取决于setTerminationEnabled ( bool enabled = true )开关。

  void requestInterruption() 
  请求线程的中断。请求是咨询意见并且取决于线程上运行的代码，来决定是否及如何执行这样的请求。此函数不停止线程上运行的任何事件循环，并且在任何情况下都不会终止它。

  工程中线程退出的解决方案如下：

  通过在线程类中增加标识变量volatile bool m_stop,通过m_stop变量的值判断run函数是否执行结束返回。

 

```
#ifndef WORKTHREAD_H



#define WORKTHREAD_H



#include <QThread>



#include <QDebug>



 



class WorkThread : public QThread



{



protected:



  //线程退出的标识量



  volatile bool m_stop;



  void run()



  {



    qDebug() << "run begin";



    while(!m_stop)



    {



        //task handling



        int* p = new int[1000];



        for(int i = 0; i < 1000; i++)



        {



            p[i] = i * i;



        }



        sleep(2);



        delete [] p;



    }



    qDebug() << "run end";



  }



public:



  WorkThread()



  {



    m_stop = false;



  }



  //线程退出的接口函数，用户使用



  void stop()



  {



    m_stop = true;



  }



};



 



#endif // WORKTHREAD_H
```

 

### **6、线程的等待**

bool wait ( unsigned long time = ULONG_MAX )

  线程将会被阻塞，等待time毫秒,如果线程退出，则wait会返回。Wait函数解决多线程在执行时序上的依赖。

void msleep ( unsigned long msecs )

void sleep ( unsigned long secs )

void usleep ( unsigned long usecs )

  sleep()、msleep()、usleep()允许秒，毫秒和微秒来区分，但在Qt5.0中被设为public。

  一般情况下，wait()和sleep()函数应该不需要，因为Qt是一个事件驱动型框架。考虑监听finished()信号来取代wait()，使用QTimer来取代sleep()。

### **7、线程的状态**

bool isFinished () const 线程是否已经退出

bool isRunning () const  线程是否处于运行状态

### **8、线程的属性**

Priority priority () const

void setPriority ( Priority priority )

uint stackSize () const

void setStackSize ( uint stackSize )

void setTerminationEnabled ( bool enabled = true )

设置是否响应terminate()函数

### **9、线程与事件循环**

  QThread中run()的默认实现调用了exec()，从而创建一个QEventLoop对象，由QEventLoop对象处理线程中事件队列（每一个线程都有一个属于自己的事件队列）中的事件。exec()在其内部不断做着循环遍历事件队列的工作，调用QThread的quit()或exit()方法使退出线程，尽量不要使用terminate()退出线程，terminate()退出线程过于粗暴，造成资源不能释放，甚至互斥锁还处于加锁状态。

  线程中的事件循环，使得线程可以使用那些需要事件循环的非GUI 类(如，QTimer,QTcpSocket,QProcess)。

  在QApplication前创建的对象，QObject::thread()返回NULL,意味着主线程仅为这些对象处理投递事件，不会为没有所属线程的对象处理另外的事件。可以用QObject::moveToThread()来改变对象及其子对象的线程亲缘关系，假如对象有父亲，不能移动这种关系。在另一个线程(而不是创建它的线程)中delete QObject对象是不安全的。除非可以保证在同一时刻对象不在处理事件。可以用QObject::deleteLater(),它会投递一个DeferredDelete事件，这会被对象线程的事件循环最终选取到。假如没有事件循环运行，事件不会分发给对象。假如在一个线程中创建了一个QTimer对象，但从没有调用过exec(),那么QTimer就不会发射它的timeout()信号，deleteLater()也不会工作。可以手工使用线程安全的函数QCoreApplication::postEvent()，在任何时候，给任何线程中的任何对象投递一个事件，事件会在那个创建了对象的线程中通过事件循环派发。事件过滤器在所有线程中也被支持，不过它限定被监视对象与监视对象生存在同一线程中。QCoreApplication::sendEvent(不是postEvent()),仅用于在调用此函数的线程中向目标对象投递事件。

## **四、线程的同步**

### **1、线程同步基础**

  临界资源：每次只允许一个线程进行访问的资源

  线程间互斥：多个线程在同一时刻都需要访问临界资源

  线程锁能够保证临界资源的安全性，通常，每个临界资源需要一个线程锁进行保护。

  线程死锁：线程间相互等待临界资源而造成彼此无法继续执行。

  产生死锁的条件：

  A、系统中存在多个临界资源且临界资源不可抢占

  B、线程需要多个临界资源才能继续执行

  死锁的避免：

  A、对使用的每个临界资源都分配一个唯一的序号

  B、对每个临界资源对应的线程锁分配相应的序号

  C、系统中的每个线程按照严格递增的次序请求临界资源

  QMutex, QReadWriteLock, QSemaphore, QWaitCondition 提供了线程同步的手段。使用线程的主要想法是希望它们可以尽可能并发执行，而一些关键点上线程之间需要停止或等待。例如，假如两个线程试图同时访问同一个全局变量，结果可能不如所愿。

### **2、互斥量****QMutex**

  QMutex 提供相互排斥的锁，或互斥量。在一个时刻至多一个线程拥有mutex,假如一个线程试图访问已经被锁定的mutex,那么线程将休眠，直到拥有mutex的线程对此mutex解锁。QMutex常用来保护共享数据访问。QMutex类所以成员函数是线程安全的。

  头文件声明：  #include <QMutex>

互斥量声明：  QMutex m_Mutex;

互斥量加锁：  m_Mutex.lock();

  互斥量解锁：  m_Mutex.unlock();

  如果对没有加锁的互斥量进行解锁，结果是未定义的。互斥量的加锁和解锁必须在同一线程中成对出现。

  QMutex ( RecursionMode mode = NonRecursive )

  QMutex有两种模式：Recursive, NonRecursive

A、Recursive

  一个线程可以对mutex多次lock，直到相应次数的unlock调用后，mutex才真正被解锁。

B、NonRecursive

  默认模式，mutex只能被lock一次。

  如果使用了Mutex.lock()而没有对应的使用Mutex.unlcok()的话就会造成死锁，其他的线程将永远也得不到接触Mutex锁住的共享资源的机会。尽管可以不使用lock()而使用tryLock(timeout)来避免因为死等而造成的死锁( tryLock(负值)==lock()),但是还是很有可能造成错误。

  bool tryLock();

  如果当前其他线程已对该mutex加锁，则该调用会立即返回，而不被阻塞。

  bool tryLock(int timeout);

  如果当前其他线程已对该mutex加锁，则该调用会等待一段时间，直到超时

 

```
QMutex mutex;



int complexFunction(int flag)



 {



     mutex.lock();



     int retVal = 0;



     switch (flag) {



     case 0:



     case 1:



         mutex.unlock();



         return moreComplexFunction(flag);



     case 2:



         {



             int status = anotherFunction();



             if (status < 0) {



                 mutex.unlock();



                 return -2;



             }



             retVal = status + flag;



         }



         break;



     default:



         if (flag > 10) {



             mutex.unlock();



             return -1;



         }



         break;



     }



 



     mutex.unlock();



     return retVal;



 }
```

 

### **3、互斥锁****QMutexLocker**

  在较复杂的函数和异常处理中对QMutex类mutex对象进行lock()和unlock()操作将会很复杂，进入点要lock()，在所有跳出点都要unlock()，很容易出现在某些跳出点未调用unlock()，所以Qt引进了QMutex的辅助类QMutexLocker来避免lock()和unlock()操作。在函数需要的地方建立QMutexLocker对象，并把mutex指针传给QMutexLocker对象，此时mutex已经加锁，等到退出函数后，QMutexLocker对象局部变量会自己销毁，此时mutex解锁。

头文件声明：  #include<QMutexLocker>

互斥锁声明：  QMutexLocker mutexLocker(&m_Mutex);

互斥锁加锁：  从声明处开始（在构造函数中加锁）

互斥锁解锁：  出了作用域自动解锁（在析构函数中解锁）

 

```
QMutex mutex;



 int complexFunction(int flag)



 {



     QMutexLocker locker(&mutex);



     int retVal = 0;



     switch (flag) {



     case 0:



     case 1:



         return moreComplexFunction(flag);



     case 2:



         {



             int status = anotherFunction();



             if (status < 0)



                 return -2;



             retVal = status + flag;



         }



         break;



     default:



         if (flag > 10)



             return -1;



         break;



     }



     return retVal;



 }
```

 

### **4、****QReadWriteLock**

  QReadWriterLock 与QMutex相似，但对读写操作访问进行区别对待，可以允许多个读者同时读数据，但只能有一个写，并且写读操作不同同时进行。使用QReadWriteLock而不是QMutex，可以使得多线程程序更具有并发性。 QReadWriterLock默认模式是[NonRecursive](https://blog.51cto.com/9291927/1879757#RecursionMode-enum)。

QReadWriterLock类成员函数如下：

QReadWriteLock ( )

QReadWriteLock ( RecursionMode recursionMode )

void lockForRead ()

void lockForWrite ()

bool tryLockForRead ()

bool tryLockForRead ( int timeout )

bool tryLockForWrite ()

bool tryLockForWrite ( int timeout )

boid unlock ()

使用实例：

 

```
QReadWriteLock lock;



 void ReaderThread::run()



 {



     lock.lockForRead();



     read_file();



     lock.unlock();



 }



 



 void WriterThread::run()



 {



     lock.lockForWrite();



     write_file();



     lock.unlock();



 }
```

 

### **5、****QReadLocker和QWriteLocker**

  在较复杂的函数和异常处理中对QReadWriterLock类lock对象进行lockForRead()/lockForWrite()和unlock()操作将会很复杂，进入点要lockForRead()/lockForWrite()，在所有跳出点都要unlock()，很容易出现在某些跳出点未调用unlock()，所以Qt引进了QReadLocker和QWriteLocker类来简化解锁操作。在函数需要的地方建立QReadLocker或QWriteLocker对象，并把lock指针传给QReadLocker或QWriteLocker对象，此时lock已经加锁，等到退出函数后，QReadLocker或QWriteLocker对象局部变量会自己销毁，此时lock解锁。

 

```
QReadWriteLock lock;



 QByteArray readData()



 {



     lock.lockForRead();



     ...



     lock.unlock();



     return data;



 }
```

 

使用QReadLocker：

 

```
QReadWriteLock lock;



 QByteArray readData()



 {



     QReadLocker locker(&lock);



     ...



     return data;



 }
```

 

### **6、信号量****QSemaphore**

  QSemaphore 是QMutex的一般化，是特殊的线程锁，允许多个线程同时访问临界资源，而一个QMutex只保护一个临界资源。QSemaphore 类的所有成员函数是线程安全的。

  经典的生产者-消费者模型如下：某工厂只有固定仓位，生产人员每天生产的产品数量不一，销售人员每天销售的产品数量也不一致。当生产人员生产P个产品时，就一次需要P个仓位，当销售人员销售C个产品时，就要求仓库中有足够多的产品才能销售。如果剩余仓位没有P个时，该批次的产品都不存入，当当前已有的产品没有C个时，就不能销售C个以上的产品，直到新产品加入后方可销售。

  QSemaphore来控制对环状缓冲的访问，此缓冲区被生产者线程和消费者线程共享。生产者不断向缓冲区写入数据直到缓冲末端，再从头开始。消费者从缓冲不断读取数据。信号量比互斥量有更好的并发性，假如我们用互斥量来控制对缓冲的访问，那么生产者、消费者不能同时访问缓冲区。然而，我们知道在同一时刻，不同线程访问缓冲的不同部分并没有什么危害。

QSemaphore 类成员函数：

QSemaphore ( int n = 0 )

void acquire ( int n = 1 )

int available () const

void release ( int n = 1 )

bool tryAcquire ( int n = 1 )

bool tryAcquire ( int n, int timeout )

实例代码：

 QSemaphore sem(5);    // sem.available() == 5

 sem.acquire(3);     // sem.available() == 2

 sem.acquire(2);     // sem.available() == 0

 sem.release(5);     // sem.available() == 5

 sem.release(5);     // sem.available() == 10

 sem.tryAcquire(1);    // sem.available() == 9, returns true

 sem.tryAcquire(250);   // sem.available() == 9, returns false

生产者-消费者实例：

\#include <QtCore/QCoreApplication>

\#include <QSemaphore>

\#include <QThread>

\#include <cstdlib>

\#include <cstdio>

const int DataSize = 100000;

const int BufferSize = 8192;

char buffer[BufferSize];

QSemaphore production(BufferSize);

QSemaphore consumption;

class Producor:public QThread

{

public:

  void *run*();

};

void Producor::*run*()

{

  for(int i = 0; i < DataSize; i++)

  {

​    production.acquire();

​    buffer[i%BufferSize] = "ACGT"[(int)qrand()%4];

​    consumption.release();

  }

}

class Consumer:public QThread

{

public:

  void *run*();

};

void Consumer::*run*()

{

  for(int i = 0; i < DataSize; i++)

  {

​    consumption.acquire();

​    fprintf(stderr, "%c", buffer[i%BufferSize]);

​    production.release();

  }

  fprintf(stderr, "%c", "\n");

}

int main(int argc, char *argv[])

{

  QCoreApplication a(argc, argv);

  Producor productor;

  Consumer consumer;

  productor.start();

  consumer.start();

  productor.wait();

  consumer.wait();

  return a.exec();

}

Producer::run函数：

  当producer线程执行run函数，如果buffer中已满，而consumer线程没有读，producer不能再往buffer中写字符,在 productor.acquire 处阻塞直到 consumer线程读(consume）数据。一旦producer获取到一个字节(资源）就写入一个随机的字符，并调用 consumer.release 使consumer线程可以获取一个资源(读一个字节的数据）。

  Consumer::run函数：

  当consumer线程执行run函数，如果buffer中没有数据，则consumer线程在consumer.acquire处阻塞，直到producer线程执行写操作写入一个字节，并执行consumer.release 使consumer线程的可用资源数=1时，consumer线程从阻塞状态中退出， 并将consumer 资源数-1，consumer当前资源数=0。

### **7、等待条件****QWaitCondition**

  QWaitCondition 允许线程在某些情况发生时唤醒另外的线程。一个或多个线程可以阻塞等待QWaitCondition ,用wakeOne()或wakeAll()设置一个条件。wakeOne()随机唤醒一个，wakeAll()唤醒所有。

QWaitCondition ()

bool wait ( QMutex * mutex, unsigned long time = ULONG_MAX )

bool wait ( QReadWriteLock * readWriteLock, unsigned long time = ULONG_MAX )

void wakeOne ()

void wakeAll ()

头文件声明：  #include <QWaitCondition>

等待条件声明：  QWaitCondtion m_WaitCondition;

等待条件等待：  m_WaitConditon.wait(&m_muxtex, time);

等待条件唤醒：  m_WaitCondition.wakeAll();

在经典的生产者-消费者场合中，生产者首先必须检查缓冲是否已满(numUsedBytes==BufferSize)，如果缓冲区已满，线程停下来等待 bufferNotFull条件。如果没有满，在缓冲中生产数据，增加numUsedBytes,激活条件 bufferNotEmpty。使用mutex来保护对numUsedBytes的访问。QWaitCondition::wait() 接收一个mutex作为参数，mutex被调用线程初始化为锁定状态。在线程进入休眠状态之前，mutex会被解锁。而当线程被唤醒时，mutex会处于锁定状态,从锁定状态到等待状态的转换是原子操作。当程序开始运行时，只有生产者可以工作，消费者被阻塞等待bufferNotEmpty条件，一旦生产者在缓冲中放入一个字节，bufferNotEmpty条件被激发，消费者线程于是被唤醒。

\#include <QtCore/QCoreApplication>

\#include <QSemaphore>

\#include <QThread>

\#include <cstdlib>

\#include <cstdio>

\#include <QWaitCondition>

\#include <QMutex>

\#include <QTime>

const int DataSize = 32;

const int BufferSize = 16;

char buffer[BufferSize];

QWaitCondition bufferNotEmpty;

QWaitCondition bufferNotFull;

QMutex mutex;

int used = 0;

class Producor:public QThread

{

public:

  void *run*();

};

void Producor::*run*()

{

  qsrand(QTime(0,0,0).secsTo(QTime::currentTime()));

  for(int i = 0; i < DataSize; i++)

  {

​    mutex.lock();

​    if(used == BufferSize)

​      bufferNotFull.wait(&mutex);

​    mutex.unlock();

​    buffer[i%BufferSize] = used;

​    mutex.lock();

​    used++;

​    bufferNotEmpty.wakeAll();

​    mutex.unlock();

  }

}

class Consumer:public QThread

{

public:

  void *run*();

};

void Consumer::*run*()

{

  for(int i = 0; i < DataSize; i++)

  {

​    mutex.lock();

​    if(used == 0)

​      bufferNotEmpty.wait(&mutex);

​    mutex.unlock();

​    fprintf(stderr, "%d\n", buffer[i%BufferSize]);

​    mutex.lock();

​    used--;

​    bufferNotFull.wakeAll();

​    mutex.unlock();

  }

  fprintf(stderr, "%c", "\n");

}

int main(int argc, char *argv[])

{

  QCoreApplication a(argc, argv);

  Producor productor;

  Consumer consumer;

  productor.start();

  consumer.start();

  productor.wait();

  consumer.wait();

  return a.exec();

}

### **8、高级事件队列**

QT事件系统对进程间通信很重要，每个进程可以有自己的事件循环，要在另外一个线程中调用一个槽函数（或任何invokable方法），需要将调用槽函数放置在目标线程的事件循环中，让目标线程在槽函数开始运行之前，先完成自己的当前任务，而原来的线程继续并行运行。

要在一个事件循环中执行调用槽函数，需要一个queued信号槽连接。每当信号发出时，信号的参数将被事件系统记录。信号接收者存活的线程将运行槽函数。另外，不使用信号，调用QMetaObject::invokeMethod()也可以达到相同的效果。在这两种情况下，必须使用queued连接，因为direct连接绕过了事件系统，并且立即在当前线程中运行此方法。

  当线程同步使用事件系统时，没有死锁风险。然而，事件系统不执行互斥。如果调用方法访问共享数据，仍然需要使用QMutex来保护。

如果只使用信号槽，并且线程间没有共享变量，那么，多线程程序可以完全没有低级原语。

## **五、可重入与线程安全**

可重入reentrant与线程安全thread-safe被用来说明一个函数如何用于多线程程序。

一个线程安全的函数可以同时被多个线程调用，甚至调用者会使用共享数据也没有问题，因为对共享数据的访问是串行的。一个可重入函数也可以同时被多个线程调用，但是每个调用者只能使用自己的数据。因此，一个线程安全的函数总是可重入的，但一个可重入的函数并不一定是线程安全的。

  一个可重入的类，指的是类的成员函数可以被多个线程安全地调用，只要每个线程使用类的不同的对象。而一个线程安全的类，指的是类的成员函数能够被多线程安全地调用，即使所有的线程都使用类的同一个实例。

### **1、可重入**

  大多数C++类是可重入的，因为它们典型地仅仅引用成员数据。任何线程可以访问可重入类实例的成员函数，只要同一时间没有其他线程调用这个实例的成员函数。

class Counter
{
 public:
   Counter() {n=0;}
   void increment() {++n;}
   void decrement() {--n;}
   int value() const {return n;}
 private:
   int n;
};

  Counter类是可重入的，但却不是线程安全的。假如多个线程都试图修改数据成员n,结果未定义。

  大多数Qt类是可重入，非线程安全的。有一些类与函数是线程安全的，主要是线程相关的类，如QMutex,QCoreApplication::postEvent()。

### **2、线程安全**

  所有的GUI类（如QWidget及其子类），操作系统核心类（如QProcess）和网络类都不是线程安全的。

class Counter
 {
 public:
   Counter() { n = 0; }

void increment() { QMutexLocker locker(&mutex); ++n; }
   void decrement() { QMutexLocker locker(&mutex); --n; }
   int value() const { QMutexLocker locker(&mutex); return n; }

private:
   mutable QMutex mutex;
   int n;
 };

 Counter类是可重入和线程安全的。QMutexLocker类在构造函数中自动对mutex进行加锁，在析构函数中进行解锁。mutex使用了mutable关键字来修饰，因为在value()函数中对mutex进行加锁与解锁操作，而value()是一个const函数。

## 六、**线程与信号槽**

### **1、线程的依附性**

  线程的依附性是对象与线程的关系。默认情况下，对象依附于自身被创建的线程。

  对象的依附性与槽函数执行的关系，默认情况下，槽函数在其所依附的线程中被调用执行。

  修改对象的依附性的方法：QObject::moveToThread函数用于改变对象的线程依附性，使得对象的槽函数在依附的线程中被调用执行。

### **2、****QObject****与****线程**

QThread类具有发送信号和定义槽函数的能力。QThread主要信号如下：

void started();线程开始运行时发送信号

void finished();线程完成运行时发送信号

void terminated();线程被异常终止时发送信号

  QThread继承自QObject,发射信号以指示线程执行开始与结束，并提供了许多槽函数。QObjects可以用于多线程，发射信号以在其它线程中调用槽函数，并且向“存活”于其它线程中的对象发送事件。
QObject的可重入性

  QObject是可重入的，QObject的大多数非GUI子类如 QTimer、QTcpSocket、QUdpSocket、QHttp、QFtp、QProcess也是可重入的，在多个线程中同时使用这些类是可能的。可重入的类被设计成在一个单线程中创建与使用，在一个线程中创建一个对象而在另一个线程中调用该对象的函数，不保证能行得通。有三种约束需要注意：

  A、一个QObject类型的孩子必须总是被创建在它的父亲所被创建的线程中。这意味着，除了别的以外，永远不要把QThread对象（this）作为该线程中创建的一个对象的父亲（因为QThread对象自身被创建在另外一个线程中）。

  B、事件驱动的对象可能只能被用在一个单线程中。特别适用于计时器机制（timer mechanism）和网络模块。例如：不能在不属于这个对象的线程中启动一个定时器或连接一个socket，必须保证在删除QThread之前删除所有创建在这个线程中的对象。在run()函数的实现中，通过在栈中创建这些对象，可以轻松地做到这一点。

  C、虽然QObject是可重入的，但GUI类，尤其是QWidget及其所有子类都不是可重入的，只能被用在GUI线程中。QCoreApplication::exec()必须也从GUI线程被调用。

  在实践中，只能在主线程而非其它线程中使用GUI的类，可以很轻易地被解决：将耗时操作放在一个单独的工作线程中，当工作线程结束后在GUI线程中由屏幕显示结果。

  一般来说，在QApplication前创建QObject是不行的，会导致奇怪的崩溃或退出，取决于平台。因此，不支持QObject的静态实例。一个单线程或多线程的应用程序应该先创建QApplication，并最后销毁QObject。

### **3、线程的事件循环**

  每个线程都有自己的事件循环。主线程通过QCoreApplication::exec()来启动自己的事件循环,但对话框的GUI应用程序，有些时候用QDialog::exec()，其它线程可以用QThread::exec()来启动事件循环。就像 QCoreApplication，QThread提供一个exit(int)函数和quit()槽函数。

  线程中的事件循环使得线程可以利用一些非GUI的、要求有事件循环存在的Qt类（例如：QTimer、QTcpSocket、和QProcess），使得连接一些线程的信号到一个特定线程的槽函数成为可能。

  

  一个QObject实例被称为存活于它所被创建的线程中。关于这个对象的事件被分发到该线程的事件循环中。可以用QObject::thread()方法获取一个QObject所处的线程。

  QObject::moveToThread()函数改变一个对象和及其子对象的线程所属性。（如果对象有父对象的话，对象不能被移动到其它线程中）。

从另一个线程（不是QObject对象所属的线程）对该QObject对象调用delete方法是不安全的，除非能保证该对象在那个时刻不处理事件，使用QObejct::deleteLater()更好。一个DeferredDelete类型的事件将被提交（posted），而该对象的线程的 件循环最终会处理这个事件。默认情况下，拥有一个QObject的线程就是创建QObject的线程，而不是 QObject::moveToThread()被调用后的。

  如果没有事件循环运行，事件将不会传递给对象。例如：在一个线程中创建了一个QTimer对象，但从没有调用exec()，那么，QTimer就永远不会发射timeout()信号，即使调用deleteLater()也不行。（这些限制也同样适用于主线程）。

  利用线程安全的方法QCoreApplication::postEvent()，可以在任何时刻给任何线程中的任何对象发送事件，事件将自动被分发到该对象所被创建的线程事件循环中。

  所有的线程都支持事件过滤器，而限制是监控对象必须和被监控对象存在于相同的线程中。QCoreApplication::sendEvent()（不同于postEvent()）只能将事件分发到和该函数调用者相同的线程中的对象。

### **4、其他线程访问QObject子类**

  QObject及其所有子类都不是线程安全的。这包含了整个事件交付系统。重要的是，切记事件循环可能正在向你的QObject子类发送事件，当你从另一个线程访问该对象时。

  如果你正在调用一个QObject子类的函数，而该子类对象并不存活于当前线程中，并且该对象是可以接收事件的，那么你必须用一个mutex保护对该QObject子类的内部数据的所有访问，否则，就有可能发生崩溃和非预期的行为。

  同其它对象一样，QThread对象存活于该对象被创建的线程中 – 而并非是在QThread::run()被调用时所在的线程。一般来说，在QThread子类中提供槽函数是不安全的，除非用一个mutex保护成员变量。

  另一方面，可以在QThread::run()的实现中安全地发射信号，因为信号发射是线程安全的。

### **5、跨线程的信号槽**

  线程的信号槽机制需要开启线程的事件循环机制，即调用QThread::exec()函数开启线程的事件循环。

Qt信号-槽连接函数原型如下：

bool QObject::connect ( const QObject * sender, const char * signal, const QObject * receiver, const char *method, Qt::ConnectionType type = Qt::AutoConnection ) 

Qt支持5种连接方式

A、Qt::DirectConnection（直连方式）（信号与槽函数关系类似于函数调用，同步执行）

  当信号发出后，相应的槽函数将立即被调用。emit语句后的代码将在所有槽函数执行完毕后被执行。

  当信号发射时，槽函数将直接被调用。

  无论槽函数所属对象在哪个线程，槽函数都在发射信号的线程内执行。

B、Qt::QueuedConnection（队列方式）（此时信号被塞到事件队列里，信号与槽函数关系类似于消息通信，异步执行）

  当信号发出后，排队到信号队列中，需等到接收对象所属线程的事件循环取得控制权时才取得该信号，调用相应的槽函数。emit语句后的代码将在发出信号后立即被执行，无需等待槽函数执行完毕。

  当控制权回到接收者所依附线程的事件循环时，槽函数被调用。

  槽函数在接收者所依附线程执行。

C、Qt::AutoConnection（自动方式）

   Qt的默认连接方式，如果信号的发出和接收信号的对象同属一个线程，那个工作方式与直连方式相同；否则工作方式与队列方式相同。

如果信号在接收者所依附的线程内发射，则等同于直接连接

如果发射信号的线程和接受者所依附的线程不同，则等同于队列连接

D、Qt::BlockingQueuedConnection(信号和槽必须在不同的线程中，否则就产生死锁)

  槽函数的调用情形和Queued Connection相同，不同的是当前的线程会阻塞住，直到槽函数返回。

E、Qt::UniqueConnection

  与默认工作方式相同，只是不能重复连接相同的信号和槽，因为如果重复连接就会导致一个信号发出，对应槽函数就会执行多次。

  QThread是用来管理线程的，QThread对象所依附的线程和所管理的线程并不是同一个概念。QThread所依附的线程，就是创建QThread对象的线程，QThread 所管理的线程，就是run启动的线程，也就是新建线程。QThread对象依附在主线程中，QThread对象的slot函数会在主线程中执行，而不是次线程。除非QThread对象依附到次线程中(通过movetoThread)。

工程实践中，为了避免冻结主线程的事件循环（即避免因此而冻结了应用的UI），所有的计算工作是在一个单独的工作线程中完成的，工作线程结束时发射一个信号，通过信号的参数将工作线程的状态发送到GUI线程的槽函数中更新GUI组件状态。

## **七、线程的设计**

### 1、**线程的生命周期**

如果线程的正处于执行过程中时，线程对象被销毁时，程序将会出错。

工程实践中线程对象的生命期必须大于线程的生命期。

### 2、**同步线程类设计**

线程对象主动等待线程生命期结束后才销毁，线程对象销毁时确保线程执行结束，支持在栈或堆上创建线程对象。

在线程类的析构函数中先调用wait函数，强制等待线程执行结束。

使用场合：适用于线程生命期较短的场合

\#ifndef SYNCTHREAD_H

\#define SYNCTHREAD_H

 

\#include <QThread>

 

class SyncThread : public QThread

{

 Q_OBJECT

protected:

 void *run*()

 {

  

 }

public:

 explicit SyncThread(QObject* parent = 0):QThread(parent)

 {

  

 }

 ~*SyncThread*()

 {

  wait();

 }

};

 

\#endif // SYNCTHREAD_H

### **3、异步线程类设计**

线程生命期结束时通知线程对象销毁。

只能在堆空间创建线程对象，线程对象不能被外界主动销毁。

在run函数中最后调用deleteLater()函数。

线程函数主动申请销毁线程对象。

使用场合：

线程生命期不可控，需要长时间运行于后台的线程。

\#ifndef ASYNCTHREAD_H

\#define ASYNCTHREAD_H

 

\#include <QThread>

 

class AsyncThread : public QThread

{

 Q_OBJECT

protected:

 void *run*()

 {

 

  deleteLater();

 }

 explicit AsyncThread(QObject* parent = 0):QThread(parent)

 {

 

 }

 ~*AsyncThread*()

 {

 

 }

public:

 static AsyncThread* newThread(QObject* parent = 0)

 {

  return new AsyncThread(parent);

 }

};

 

\#endif // ASYNCTHREAD_H

## **八、线程的使用方式**

### **1、子类化QThread**

QThread的两种使用方法:

（1）不使用事件循环

 A、子类化 QThread

  B、重写run函数，run函数内有一个 while 或 for 的死循环

  C、设置一个标记为来控制死循环的退出。

  适用于后台执行长时间的耗时操作，如文件复制、网络数据读取。

（2）使用事件循环。

  A、子类化 QThread

  B、重写run 使其调用 QThread::exec() ，开启线程的事件循环

C、为子类定义信号和槽，由于槽函数并不会在新开的 Thread 运行，在构造函数中调用 moveToThread(this)。

适用于事务性操作，如文件读写、数据库读写。

### **2、Worker-Object**

  在Qt4.4之前，run 是纯虚函数，必须子类化QThread来实现run函数。
  而从Qt4.4开始，QThread不再支持抽象类，run 默认调用 QThread::exec() ，不需要子类化 QThread，只需要子类化一个 QObject 。

  通过继承的方式实现多线程已经没有任何意义，QThread是操作系统线程的接口或控制点，用于充当线程操作的集合。

  使用Worker-Object通过QObject::moveToThread将它们移动到线程中。

  指定一个线程对象的线程入口函数的方法：

A、在类中定义一个槽函数void tmain()作为线程入口函数

B、在类中定义一个QThread成员对象m_thread

C、改变当前对象的线程依附性到m_thread

D、连接m_thread的started()信号到tmain槽函数。

 

\#ifndef WORKER_H

\#define WORKER_H

 

\#include <QObject>

\#include <QThread>

\#include <QDebug>

 

class Worker : public QObject

{

 Q_OBJECT

 QThread m_thread;

protected slots:

 void tmain()

 {

  qDebug() << "void tmain()";

 }

public:

 explicit Worker(QObject* parent = 0):QObject(parent)

 {

  moveToThread(&m_thread);

  connect(&m_thread, SIGNAL(started()), this, SLOT(tmain()));

 }

 void start()

 {

  m_thread.start();

 }

 void terminate()

 {

  m_thread.terminate();

 }

 

 void exit(int c)

 {

  m_thread.exit(c);

 }

 ~*Worker*()

 {

  m_thread.wait();

 }

};

 

\#endif // WORKER_H

## **九、多线程与GUI组件的通信**

### **1、多线程与GUI组件通信基础**

  GUI系统的设计原则：

  所有界面组件的创建只能在GUI线程（主线程）中完成。子线程与界面组件的通信有两种方式：

  A、信号槽方式

  B、发送自定事件方式

### **2、信号槽方式**

使用信号槽解决多线程与界面组件的通信的方案：

A、在子线程中定义界面组件的更新信号

B、在主窗口类中定义更新界面组件的槽函数

C、使用异步方式连接更新信号到槽函数

子线程通过发送信号的方式更新界面组件，所有的界面组件对象只能依附于GUI线程（主线程）。

子线程更新界面状态的本质是子线程发送信号通知主线程界面更新请求，主线程根据具体信号以及信号参数对界面组件进行修改。

使用信号槽在子线程中更新主界面中进度条的进度显示信息。

工作线程类：

\#ifndef WORKTHREAD_H

\#define WORKTHREAD_H

\#include <QThread>

 

class WorkThread : public QThread

{

 Q_OBJECT

signals:

 void signalProgressValue(int value);

protected:

 void *run*()

 {

  work();

  exec();

 }

 

public:

 WorkThread()

 {

  m_stop = false;

  moveToThread(this);

 }

 void work()

 {

  for(int i = 0; i < 11; i++)

  {

​    emit signalProgressValue(i*10);

​    sleep(1);

  }

 }

};

 

\#endif // WORKTHREAD_H

主界面类：

\#ifndef WIDGET_H

\#define WIDGET_H

 

\#include <QWidget>

\#include <QProgressBar>

\#include "WorkThread.h"

 

class Widget : public QWidget

{

 Q_OBJECT

 QProgressBar* m_progress;//进度条

 WorkThread* m_thread;//工作线程

public:

 Widget(QWidget *parent = 0):QWidget(parent)

 {

  m_progress = new QProgressBar(this);

  m_progress->move(10, 10);

  m_progress->setMinimum(0);

  m_progress->setMaximum(100);

  m_progress->setTextVisible(true);

  m_progress->resize(100, 30);

  m_thread = new WorkThread();

  m_thread->start();

  connect(m_thread, SIGNAL(finished()), m_thread, SLOT(deleteLater()));

  //连接工作线程的信号到界面的槽函数

  connect(m_thread, SIGNAL(signalProgressValue(int)), this, SLOT(onProgress(int)));

 }

 ~*Widget*()

 {

 }

protected slots:

 void onProgress(int value)

 {

  m_progress->setValue(value);

 }

};

 

\#endif // WIDGET_H

Main函数：

\#include "Widget.h"

\#include <QApplication>

 

int main(int argc, char *argv[])

{

 QApplication a(argc, argv);

 Widget w;

 w.show();

 

 return a.exec();

}

### **3、发送自定义事件方式**

  A、自定义事件用于描述界面更新细节

  B、在主窗口类中重写事件处理函数event

  C、使用postEvent函数（异步方式）发送自定义事件类对象

  子线程指定接收消息的对象为主窗口对象，在event事件处理函数更新界面状态

  事件对象在主线程中被处理，event函数在主线程中调用。

  发送的事件对象必须在堆空间创建

  子线程创建时必须附带目标对象的地址信息

自定义事件类：

\#ifndef PROGRESSEVENT_H

\#define PROGRESSEVENT_H

\#include <QEvent>

 

class ProgressEvent : public QEvent

{

 int m_progress;

public:

 const static Type TYPE = static_cast<Type>(QEvent::User + 0xFF);

 ProgressEvent(int progress = 0):QEvent(TYPE)

 {

  m_progress = progress;

 }

 int progress()const

 {

  return m_progress;

 }

};

 

\#endif // PROGRESSEVENT_H

 

自定义线程类：

\#ifndef WORKTHREAD_H

\#define WORKTHREAD_H

\#include <QThread>

\#include <QApplication>

\#include <ProgressEvent.h>

 

class WorkThread : public QThread

{

 Q_OBJECT

protected:

 volatile bool m_stop;

 void *run*()

 {

  work();

  exec();

 }

 

public:

 WorkThread()

 {

  m_stop = false;

 }

 void stop()

 {

  m_stop = true;

 }

 void work()

 {

  for(int i = 0; i < 11; i++)

  {

​    QApplication::postEvent(parent(), new ProgressEvent(i*10));

​    sleep(1);

  }

 }

};

 

\#endif // WORKTHREAD_H

自定义界面类：

\#ifndef WIDGETUI_H

\#define WIDGETUI_H

 

\#include <QWidget>

\#include <QProgressBar>

\#include "WorkThread.h"

\#include "ProgressEvent.h"

 

class WidgetUI : public QWidget

{

 Q_OBJECT

 QProgressBar* m_progress;//进度条

 WorkThread* m_thread;//工作线程

public:

 WidgetUI(QWidget *parent = 0):QWidget(parent)

 {

  m_progress = new QProgressBar(this);

  m_progress->move(10, 10);

  m_progress->setMinimum(0);

  m_progress->setMaximum(100);

  m_progress->setTextVisible(true);

  m_progress->resize(100, 30);

  m_thread = new WorkThread();

  m_thread->setParent(this);

  m_thread->start();

 }

 ~*WidgetUI*()

 {

  m_thread->quit();

 }

protected:

 bool *event*(QEvent *event)

 {

  bool ret = true;

  if(event->type() == ProgressEvent::TYPE)

  {

​    ProgressEvent* evt = dynamic_cast<ProgressEvent*>(event);

​    if(evt != NULL)

​    {

​      //设置进度条的进度为事件参数的值

​      m_progress->setValue(evt->progress());

​    }

  }

  else

  {

​    ret = QWidget::*event*(event);

  }

  return ret;

 }

};

 

\#endif // WIDGETUI_H

Main函数：

\#include "WidgetUI.h"

\#include <QApplication>

 

int main(int argc, char *argv[])

{

 QApplication a(argc, argv);

 WidgetUI w;

 w.show();

 

 return a.exec();

}

- [点赞 1](javascript:;)