# QML与C++混合编程

https://blog.csdn.net/aming090/article/details/83514871



## 一、**QML与C++混合编程简介**

  QML与C++混合编程就是使用QML高效便捷地构建UI，而C++则用来实现业务逻辑和复杂算法。

## 二、QML访问C++

  Qt集成了QML引擎和Qt元对象系统，使得QML很容易从C++中得到扩展，在一定的条件下，QML就可以访问QObject派生类的成员，例如信号、槽函数、枚举类型、属性、成员函数等。

  QML访问C++有两个方法：一是在Qt元对象系统中注册C++类，在QML中实例化、访问；二是在C++中实例化并设置为QML上下文属性，在QML中直接使用。第一种方法可以使C++类在QML中作为一个数据类型，例如函数参数类型或属性类型，也可以使用其枚举类型、单例等，功能更强大。

## 三、**C++类的实现**

  C++类要想被QML访问，首先必须满足两个条件：一是派生自QObject类或QObject类的子类，二是使用Q_OBJECT宏。QObject类是所有Qt对象的基类，作为Qt对象模型的核心，提供了信号与槽机制等很多重要特性。Q_OBJECT宏必须在private区（C++默认为private）声明，用来声明信号与槽，使用Qt元对象系统提供的内容，位置一般在语句块首行。Projects选择Qt Quick Application，工程名为Hello。

### 1、**信号与槽实现**

A、C++类实现

```c
#ifndef HELLO_H
#define HELLO_H
#include <QObject>
#include <QDebug>
 
class Hello: public QObject
{
Q_OBJECT
public slots:
  void doSomething()
  {
    qDebug() << "Hello::dosomething() is called.";
  }
signals:
  void begin();
};
 
#endif // HELLO_H
```

 

  Hello类中的信号begin()和槽doSomething()都可以被QML访问。

**槽必须声明为public或protected，**

**信号在C++中使用时要用到emit关键字，但在QML中就是个普通的函数，用法同函数一样，信号处理器形式为on，Signal首字母大写。**

**信号不支持重载，多个信号的名字相同而参数不同时，能够被识别的只是最后一个信号，与信号的参数无关。**



B、注册C++类型

```c
#include <QGuiApplication>
#include <QQmlApplicationEngine>
#include <QtQml>
#include "hello.h"
 
int main(int argc, char *argv[])
{
  QGuiApplication app(argc, argv);
  //注册C++类型Hello
  qmlRegisterType<Hello>("Hello.module",1,0,"Hello");
 
  QQmlApplicationEngine engine;
  engine.load(QUrl(QStringLiteral("qrc:/main.qml")));
 
  return app.exec();
}
```

 

将C++类注册到Qt元对象系统。

C、在QML文件中导入C++类并使用

```c
import QtQuick 2.5
import QtQuick.Window 2.2
//导入注册的C++类
import Hello.module 1.0
 
Window {
    visible: true
    width: 640
    height: 480
    title: qsTr("Hello QML")
    MouseArea {
        anchors.fill: parent
        onClicked: {
            hello.begin();//单击鼠标调用begin信号函数
        }
    }
    Hello{
        id:hello   //Hello类的实例
        onBegin:doSomething()
    }
}
```

 

  在QML文件中导入注册的C++类（import），关键字Hello就可以在当前QML文件中当作一种QML类型来用。MouseArea铺满界面，单击鼠标时会发送begin()信号，进而调用doSomething()槽函数。

### 2、**枚举类型实现**

A、C++类中枚举的定义

```c

#ifndef HELLO_H
#define HELLO_H
#include <QObject>
#include <QDebug>
 
class Hello: public QObject
{
Q_OBJECT
Q_ENUMS(Color)
public:
  Hello():m_color(RED)
  {
    qDebug() << "Hello() is called.";
  }
  //枚举
  enum Color
  {
    RED,
    BLUE,
    BLACK
  };
public slots:
  void doSomething(Color color)
  {
    qDebug() << "Hello::dosomething() is called " << color;
  }
 
signals:
  void begin();
private:
  Color m_color;
};
 
#endif // HELLO_H
```

 

  C++类中添加了public的Color枚举类型，枚举类型要想在QML中使用，需要使用Q_ENUMS()宏。

B、QML文件中使用C++枚举类型

```c

import QtQuick 2.5
import QtQuick.Window 2.2
//导入注册的C++类
import Hello.module 1.0
 
Window {
    visible: true
    width: 640
    height: 480
    title: qsTr("Hello QML")
    MouseArea {
        anchors.fill: parent
        onClicked: {
            hello.begin();//单击鼠标调用begin信号函数
        }
    }
    Hello{
        id:hello   //Hello类的实例
        onBegin:doSomething(Hello.RED)
    }
}
```

  QML中使用枚举类型的方式是通过C++类型名使用“.”操作符直接访问枚举成员，如Hello.RED。

### 3、**成员函数实现**

A、成员函数定义

```c

#ifndef HELLO_H
#define HELLO_H
#include <QObject>
#include <QDebug>
 
class Hello: public QObject
{
Q_OBJECT
Q_ENUMS(Color)
public:
  Hello():m_color(RED)
  {
    qDebug() << "Hello() is called.";
  }
  //枚举
  enum Color
  {
    RED,
    BLUE,
    BLACK
  };
  Q_INVOKABLE void show()
  {
    qDebug() << "show() is called.";
  }
public slots:
  void doSomething(Color color)
  {
    qDebug() << "Hello::dosomething() is called " << color;
  }
 
signals:
  void begin();
private:
  Color m_color;
};
 
#endif // HELLO_H
```

 

  如果QML中访问C++成员函数，则C++成员函数必须是public或protected成员函数，且使用Q_INVOKABLE宏，位置在函数返回类型的前面。

B、QML中调用C++类成员函数

```c
import QtQuick 2.5
import QtQuick.Window 2.2
//导入注册的C++类
import Hello.module 1.0
 
Window {
    visible: true
    width: 640
    height: 480
    title: qsTr("Hello QML")
    MouseArea {
        anchors.fill: parent
        onClicked: {
            hello.begin();//单击鼠标调用begin信号函数
            hello.show();
        }
    }
    Hello{
        id:hello   //Hello类的实例
        onBegin:doSomething(Hello.RED)
    }
}
```

 

  在QML中访问C++的成员函数的形式是“.”,如*hello*.show()，支持函数重载。

### 4、**C++类的属性**

A、C++类中属性的定义

```c

#ifndef HELLO_H
#define HELLO_H
#include <QObject>
#include <QDebug>
 
class Hello: public QObject
{
Q_OBJECT
Q_ENUMS(Color)
//属性声明
Q_PROPERTY(Color color READ color WRITE setColor NOTIFY colorChanged)
public:
  Hello():m_color(RED)
  {
    qDebug() << "Hello() is called.";
  }
  //枚举
  enum Color
  {
    RED,
    BLUE,
    BLACK
  };
  Q_INVOKABLE void show()
  {
    qDebug() << "show() is called.";
  }
  Color color() const
  {
    return m_color;
  }
  void setColor(const Color& color)
  {
    if(color != m_color)
    {
        m_color = color;
        emit colorChanged();
    }
  }
public slots:
  void doSomething(Color color)
  {
    qDebug() << "Hello::dosomething() is called " << color;
  }
 
signals:
  void begin();
  void colorChanged();
private:
  Color m_color;//属性
};
 
#endif // HELLO_H
```

 

  C++类中添加了Q_PROPERTY()宏，用来在QObject派生类中声明属性，属性同类的数据成员一样，但又有一些额外的特性可通过Qt元对象系统来访问。

Q_PROPERTY()(type name

   (READ getFunction [WRITE setFunction] |

​       MEMBER memberName [(READ getFunction | WRITE setFunction)])

​      [RESET resetFunction]

​      [NOTIFY notifySignal]

​      [REVISION int]

​      [DESIGNABLE bool]

​      [SCRIPTABLE bool]

​      [STORED bool]

​      [USER bool]

​      [CONSTANT]

​      [FINAL])

  属性的type、name是必需的，其它是可选项，常用的有READ、WRITE、NOTIFY。属性的type可以是QVariant支持的任何类型，也可以是自定义类型，包括自定义类、列表类型、组属性等。另外，属性的READ、WRITE、RESET是可以被继承的，也可以是虚函数，不常用。

  READ：读取属性值，如果没有设置MEMBER的话，是必需的。一般情况下，函数是个const函数，返回值类型必须是属性本身的类型或这个类型的const引用，没有参数。

  WRITE：设置属性值，可选项。函数必须返回void，有且仅有一个参数，参数类型必须是属性本身的类型或类型的指针或引用。

  NOTIFY：与属性关联的可选信号，信号必须在类中声明过，当属性值改变时，就可触发信号，可以没有参数，有参数的话只能是一个类型同属性本身类型的参数，用来记录属性改变后的值。

B、QML中修改属性

```c
import QtQuick 2.5
import QtQuick.Window 2.2
//导入注册的C++类
import Hello.module 1.0
 
Window {
    visible: true
    width: 640
    height: 480
    title: qsTr("Hello QML")
    MouseArea {
        anchors.fill: parent
        onClicked: {
            hello.begin()//单击鼠标调用begin信号函数
            hello.show()
            hello.color = 2  //修改属性
        }
    }
    Hello{
        id:hello   //Hello类的实例
        onBegin:doSomething(Hello.RED)
        onColorChanged:console.log("color changed.")
    }
}
```

  C++类中的m_color属性可以在QML中访问、修改，访问时调用了color()函数，修改时调用setColor()函数，同时还发送了一个信号来自动更新color属性值。

## 四、**注册C++类为QML类型**

  QObject派生类可以注册到Qt元对象系统，使得类在QML中同其它内建类型一样，可以作为一个数据类型来使用。QML引擎允许注册可实例化的类型，也可以是不可实例化的类型，常见的注册函数有：

qmlRegisterInterface()

qmlRegisterRevision()

qmlRegisterSingletonType()

qmlRegisterType()

qmlRegisterTypeNotAvailable()

qmlRegisterUncreatableType()

 

template<typename T>

int qmlRegisterType(const char *uri,int versionMajor,

​    int versionMinor, const char *qmlName);

  模板函数注册C++类到Qt元对象系统中，uri是需要导入到QML中的库名，versionMajor和versionMinor是其版本数字，qmlName是在QML中可以使用的类型名。

  qmlRegisterType<Hello>("Hello.module",1,0,"Hello");

  main.cpp中将Hello类注册为在QML中可以使用的GHello类型，主版本为1，次版本为0，库的名字是Hello.module。main.qml中导入了C++库，使用Hello构造了一个对象，id为hello，可以借助id来访问C++。

  注册动作必须在QML上下文创建前，否则无效。

  QQuickView为QtQuickUI提供了一个窗口，可以方便地加载QML文件并显示其界面。QApplication派生自QGuiApplication，而QGuiApplication又派生自QCoreApplication，这三个类是常见的管理Qt应用程序的类。QQmlApplicationEngine可以方便地从一个单一的QML文件中加载应用程序，派生自QQmlEngine，QQmlEngine则提供了加载QML组件的环境，可以与QQmlComponent、QQmlContext等一起使用。

## 五、**QML上下文属性设置**

  在C++应用程序加载QML对象时，可以直接嵌入一些C++数据来给QML使用，需要用到QQmlContext::setContextProperty()设置QML上下文属性，上下文属性可以是一个简单的类型，也可以是任何自定义的类对象。

A、C++设置上下文属性

```c
#include <QGuiApplication>
#include <QQuickView>
#include <QQmlContext>
#include "hello.h"
 
int main(int argc, char *argv[])
{
  QGuiApplication app(argc, argv);
  
  QQuickView view;
  Hello hello;
  view.rootContext()->setContextProperty("hello", &hello);
  view.setSource(QUrl(QStringLiteral("qrc:///main.qml")));
  view.show();
 
  return app.exec();
}
```

 

  Hello类先实例化为hello对象，然后注册为QML上下文属性。

B、QML使用

```c
import QtQuick 2.5
 
Item {
    width: 640
    height: 480
    MouseArea {
        anchors.fill: parent
        onClicked: {
            hello.begin()//单击鼠标调用begin信号函数
            hello.show()
        }
    }
    Connections{
       target:hello
       onBegin:console.log("hello")
    }
}
```

 

  在main.qml中不能使用Hello类型来实例化，也不能调用doSomething()槽函数，因为doSomething()函数中的枚举类型在QML中是访问不到的，正确的用法是通过已经设置的QML上下文属性“hello”来访问C++，可以访问信号begin()和成员函数show()，此时的信号处理器需要用Connections来处理。

## 六、**C++访问QML**

  在C++中也可以访问QML中的属性、函数和信号。

  在C++中加载QML文件可以用QQmlComponent或QQuickView，然后就可以在C++中访问QML对象。QQuickView提供了一个显示用户界面的窗口，而QQmlComponent没有。

### 1、**C++使用****QQmlComponent**

```c
#include <QGuiApplication>
#include <QQmlApplicationEngine>
#include <QtQml>
#include "hello.h"
 
int main(int argc, char *argv[])
{
  QGuiApplication app(argc, argv);
  //注册C++类型Hello
  qmlRegisterType<Hello>("Hello.module",1,0,"Hello");
 
  QQmlApplicationEngine engine;
  QQmlComponent component(&engine, QUrl(QStringLiteral("qrc:/main.qml")));
  component.create();
 
  return app.exec();
}
```

 

### 2、**C++使用QML的属性**

  在C++中加载了QML文件并进行组件实例化后，就可以在C++中访问、修改实例的属性值，可以是QML内建属性，也可以是自定义属性。

A、QML中定义部分元素的属性

```c
import QtQuick 2.5
import QtQuick.Window 2.0
import Hello.module 1.0
 
Window {
    visible: true
    width: 640
    height: 480
    title: "Hello QML"
    color: "white"
    MouseArea {
        anchors.fill: parent
        onClicked: {
            hello.begin()//单击鼠标调用begin信号函数
            hello.doSomething(2)
        }
    }
    Rectangle{
        objectName: "rect"
        anchors.fill: parent
        color:"red"
        }
    Hello{
       id:hello
       onBegin:console.log("hello")
       onColorChanged:hello.show()
    }
}
```

  QML中添加了一个Rectangle，设置objectName属性值为“rect”，objectName是为了在C++中能够找到Rectangle。

B、C++中使用QML元素的属性

```c
#include <QGuiApplication>
#include <QQmlApplicationEngine>
#include <QtQml>
#include "hello.h"
 
int main(int argc, char *argv[])
{
  QGuiApplication app(argc, argv);
  //注册C++类型Hello
  qmlRegisterType<Hello>("Hello.module",1,0,"Hello");
 
  QQmlApplicationEngine engine;
  QQmlComponent component(&engine, QUrl(QStringLiteral("qrc:/main.qml")));
  QObject* object = component.create();
  qDebug() << "width value is" << object->property("width").toInt();
  object->setProperty("width", 500);//设置window的宽
  qDebug() << "width value is" << object->property("width").toInt();
  qDebug() << "height value is" << QQmlProperty::read(object, "height").toInt();
  QQmlProperty::write(object, "height", 300);//设置window的高
  qDebug() << "height value is" << QQmlProperty::read(object, "height").toInt();
  QObject* rect = object->findChild<QObject*>("rect");//查找名称为“rect”的元素
  if(rect)
  {
      rect->setProperty("color", "blue");//设置元素的color属性值
      qDebug() << "color is " << object->property("color").toString();
  }
 
  return app.exec();
}
```

 

  QObject::property()/setProperty()来读取、修改width属性值。

  QQmlProperty::read()/write()来读取、修改height属性值。

  如果某个对象的类型是QQuickItem，例如QQuickView::rootObject()的返回值，可以使用QQuickItem::width/setWidth()来访问、修改width属性值。

  QML组件是一个复杂的树型结构，包含兄弟组件和孩子组件，可以使用QObject::findchild()/findchildren()来查找。

### 3、**C++使用QML中信号与函数**

  在C++中，使用QMetaObject::invokeMethod()可以调用QML中的函数，从QML传递过来的函数参数和返回值会被转换为C++中的QVariant类型，成功返回true，参数不正确或被调用函数名错误返回false，invokeMethod()共有四个重载函数。必须使用Q_ARG()宏来声明函数参数，用Q_RETURN_ARG()宏来声明函数返回值，其原型如下：

QGenericArgument      Q_ARG(Type, const Type & value)

QGenericReturnArgument   Q_RETURN_ARG(Type, Type & value)

  使用QObject::connect()可以连接QML中的信号，connect()共有四个重载函数，都是静态函数。必须使用SIGNAL()宏来声明信号，SLOT()宏声明槽函数。

  使用QObject::disconnect()可以解除信号与槽函数的连接。

A、QML中定义信号与函数

```c
import QtQuick 2.5
import QtQuick.Window 2.0
import Hello.module 1.0
 
Window {
    visible: true
    width: 640
    height: 480
    title: "Hello QML"
    color: "white"
    //定义信号
    signal qmlSignal(string message)
    //定义函数
    function qmlFunction(parameter) {
        console.log("qml function parameter is", parameter)
        return "function from qml"
    }
    MouseArea {
        anchors.fill: parent
        onClicked: {
            hello.begin()//单击鼠标调用begin信号函数
            hello.doSomething(2)
            qmlSignal("This is an qml signal.")//发送信号
        }
    }
    Rectangle{
        objectName: "rect"
        anchors.fill: parent
        color:"red"
        }
    Hello{
       id:hello
       onBegin:console.log("hello")
       onColorChanged:hello.show()
    }
}
```

 

  QML中添加了qmlSignal()信号和qmlFunction()函数，信号在QML中发送，函数在C++中调用。

B、C++中调用

```c

#include <QGuiApplication>
#include <QQmlApplicationEngine>
#include <QtQml>
#include "hello.h"
 
int main(int argc, char *argv[])
{
  QGuiApplication app(argc, argv);
  //注册C++类型Hello
  qmlRegisterType<Hello>("Hello.module",1,0,"Hello");
 
  QQmlApplicationEngine engine;
  QQmlComponent component(&engine, QUrl(QStringLiteral("qrc:/main.qml")));
  QObject* object = component.create();
  qDebug() << "width value is" << object->property("width").toInt();
  object->setProperty("width", 500);//设置window的宽
  qDebug() << "width value is" << object->property("width").toInt();
  qDebug() << "height value is" << QQmlProperty::read(object, "height").toInt();
  QQmlProperty::write(object, "height", 300);//设置window的高
  qDebug() << "height value is" << QQmlProperty::read(object, "height").toInt();
  QObject* rect = object->findChild<QObject*>("rect");//查找名称为“rect”的元素
  if(rect)
  {
      rect->setProperty("color", "blue");//设置元素的color属性值
      qDebug() << "color is " << object->property("color").toString();
  }
  //调用QML中函数
  QVariant returnedValue;
  QVariant message = "Hello from C++";
  QMetaObject::invokeMethod(object, "qmlFunction",
                            Q_RETURN_ARG(QVariant, returnedValue),
                            Q_ARG(QVariant, message));
  qDebug() << "returnedValue is" << returnedValue.toString(); // function from qml
  Hello hello;
  //连接QML元素中的信号到C++槽函数
  QObject::connect(object, SIGNAL(qmlSignal(QString)),
                   &hello, SLOT(qmlSlot(QString)));
 
  return app.exec();
}
```

 

C++类中的槽函数：

```c
public slots:
  void doSomething(Color color)
  {
    qDebug() << "Hello::dosomething() is called " << color;
  }
  void qmlSlot(const QString& message)
  {
    qDebug() << "C++ called: " << message;
  }
 
```

 

## 七、**QML与C++混合编程注意事项**

  QML与混合编程注意事项：

  A、自定义C++类一定要派生自QObject类或其子类，并使用Q_OBJECT宏。

  B、注册自定义C++类到Qt元对象系统或设置自定义类对象实例为QML上下文属性是必须的。

  C、两者交互进行数据传递时，要符合QML与C++间数据类型的转换规则。