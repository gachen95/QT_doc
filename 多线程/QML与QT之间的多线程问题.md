# QML与QT之间的多线程问题

最新在运用QML与QT结合时，发现线程是个头痛的问题，如果在QML做了大部分的UI事情时，如何把后台运行在线程中呢？ 官方提供的方法大概以下几种：

1.逻辑处理类继承 QThread，通过通过一系列的signals，然后在run()方法使用，以下官方代码摘抄：
```
class WorkerThread : public QThread
 {
   Q_OBJECT
   void run() Q_DECL_OVERRIDE {
     QString result;
     /* ... here is the expensive or blocking operation ... */
     emit resultReady(result);
   }
 signals:
   void resultReady(const QString &s);
 };
```
很显然让用户在使用时再次用connect连接一些新的槽实现。

 

2.很厉害的moveToThread方法，简单便捷，不用新建一个类然后整一堆的重复的线程代码，官方示例如下：
```
Worker *worker = new Worker;
     worker->moveToThread(&workerThread);
     connect(&workerThread, &QThread::finished, worker, &QObject::deleteLater);
     connect(this, &Controller::operate, worker, &Worker::doWork);
     connect(worker, &Worker::resultReady, this, &Controller::handleResults);
     workerThread.start();
```
这里的特别注意一点，connect的使用特别关键，举个例子，如果此句代码去掉：

 connect(this, &Controller::operate, worker, &Worker::doWork);

而改成在Controller类中直接调用：
worker->doWork()；

**那坑爹的事情可能来了，worker此时并非运行在workerThread上了，而是跟Controller一个线程，如果此时是UI线程，你的界面肯定会卡住。**

 

3.单纯在QML中也可以使用一种WorkScript的块，限制特别多，能处理的情况很有限。 步骤如下：

Step1. 准备一个js文件，写上方法如下格式：
```
WorkerScript.onMessage = function(message) {
  // ... long-running operations and calculations are done here

  WorkerScript.sendMessage({ 'keyA': 'message:'+message.keyB })

}
```
Step2.QML文件中定义一类似这样的块，这是用来接收js的回调的。
```
  WorkerScript{
    id:WorkerScriptObj
    source:"myworkscript.js"
    onMessage:{

​      console.log(messageObject.keyA);
​    }
  }
```
Step3. 如果需要主动调用就用：
```
WorkerScriptObj.sendMessage({'keyB':100);
```
keyA是从JS中传出来的，keyB是可以从QML中传过去的。

 

**注意了： 悲剧的事情来了，首先不可以传对象值到JS中，而且在JS也不能访问任何的QML的对象，所以，如果想处理基本数据操作之外的东西，那这种方式就将让你失望了~~ 可能做个小计算器是没问题的**

 

 

总结：最近一直纠结一个线程问题，因此MARK在这里，留着以后查阅需要，暂时只是用简单的处理，线程操作还有很多，用到继续更新~！！

转载于:https://my.oschina.net/u/138823/blog/824533