# qtquick 与c++交互总结



交互方式分为四种,qml访问C++,C++调用qml，model/delegate/view机制，使用序列化字符串json，前面两种是基本，后面两个是第一个的特例，如有不严谨的地方，欢迎指正。

###qml访问C++
qml不能直接访问c++的类型，C++类型必须注册到元对象系统才能被qml访问。qml可以访问属性、信号、槽、枚举定义(Q_ENUM)、函数(Q_INVOKABLE)。

下面以Student类为例，演示上面的各个类型，

###第一种方式，注册类型，

第一步，定义类型。

```c
#include <QObject>

class Student : public QObject
{
    Q_OBJECT
public:
    explicit Student(QObject *parent = nullptr)
    {

    }
    enum Grade{
        FirstGrade,
        SecondGrade,
        GradeThree
    };
    Q_ENUM(Grade)
    Q_PROPERTY(int age READ age WRITE setAge NOTIFY ageChanged)
    Q_INVOKABLE void setName(QString value)
    {
     
    }
    int age()const throw()
    {
        return m_age;
    }
    void setAge(int value)
    {
        if(m_age!=value){
            m_age=value;
            emit ageChanged();
        }
    }
signals:
    void add(int a,int b);
    void ageChanged();
public slots:
    void setGrade(Grade value)
    {

    }

private:
    int m_age;
};
```

第二部，在main.cpp中注册类型，

这样在qml中需要import csdn.fish。注册类型必须在QQmlApplicationEngine对象之前。（还有qmlRegisterType()，qmlRegisterTypeNotAvailable,qmlRegisterSingletonType等注册函数，大同小异，这里不展开讨论）
qmlRegisterType<Student>("csdn.fish", 1, 0, "Student");

第三部，在qml中可以定义该类型，

访问age属性，调用setName函数，调用setGrade槽，响应add信号。

```c
import QtQuick.Controls 2.0
import csdn.fish 1.0
ApplicationWindow {
    visible: true
    width: 640
    height: 480
    title: qsTr("Hello World")
    Student{
        id:liming
        age:20
        onAdd: console.info(a,b)
        onAgeChanged: console.info(age)
        Component.onCompleted: {
            liming.setName("liming");
            liming.setGrade(Student.SecondGrade)
        }
    }
}
```



###第二种方式是设为context的属性，

在main.cpp中这样定义:

```c
#include <QGuiApplication>
#include <QQmlApplicationEngine>
#include <QQmlContext>
#include "Student.h"
int main(int argc, char *argv[])
{
    QCoreApplication::setAttribute(Qt::AA_EnableHighDpiScaling);
    QGuiApplication app(argc, argv);
    qmlRegisterType<Student>("csdn.fish", 1, 0, "Student");
    QQmlApplicationEngine engine;
    Student f_Student;
    engine.rootContext()->setContextProperty("superStudent",&f_Student);
    engine.load(QUrl(QLatin1String("qrc:/main.qml")));
    if (engine.rootObjects().isEmpty())
        return -1;

    return app.exec();
}
```


这样在整个工程的任意qml中可以使用superStudent来访问Student的属性、信号、槽等

```c
import QtQuick 2.7
import QtQuick.Controls 2.0
import csdn.fish 1.0
ApplicationWindow {
    visible: true
    width: 640
    height: 480
    title: qsTr("Hello World")
    Student{
        id:liming
        age:20
        onAdd: console.info(a,b)
        onAgeChanged: console.info(age)
        Component.onCompleted: {
            liming.setName("liming");
            liming.setGrade(Student.SecondGrade)
        }
    }
    Connections{
        target: superStudent
        onAgeChanged:{
            var i=0
        }
    }

    Component.onCompleted: {
        superStudent.setName("hanmeimei")
        superStudent.setGrade(Student.FirstGrade)
        superStudent.age=15
        superStudent.age=16
    }
}
```



## C++调用QML
完全从c++调用qml，需要使用objectName属性来遍历树来查找对象，很不符合我胃口，这种情况不举例，仅以下面为例
首先是main.qml，在qml中定义两个函数，

```c
import QtQuick 2.7
import QtQuick.Controls 2.0
import csdn.fish 1.0
ApplicationWindow {
    visible: true
    width: 640
    height: 480
    title: qsTr("Hello World")
    Student{
        id:liming
        Component.onCompleted: {
            liming.setName("liming");
        }
        function callTeacher(number)
        {
            console.info(number)
        }
        function add(a,b)
        {
            return a+b
        }
    }
}
```
c++中通过invokeMethod来调用qml中的函数，可以指定连接方式，如下：

```c
#include <QObject>
#include <QVariant>
class Student : public QObject
{
    Q_OBJECT
public:
    explicit Student(QObject *parent = nullptr)
    {

    }
    Q_INVOKABLE void setName(QString value)
    {
        QVariant a=QVariant::fromValue(10);
        QVariant b=QVariant::fromValue(8);
        QVariant retVal;
        QMetaObject::invokeMethod(this, "add", Qt::DirectConnection,
                                   Q_RETURN_ARG(QVariant, retVal),
                                   Q_ARG(QVariant, a),
                                   Q_ARG(QVariant, b));
        QVariant para=QVariant::fromValue(retVal.toInt());
        QMetaObject::invokeMethod(this, "callTeacher", Qt::QueuedConnection,
                                   Q_ARG(QVariant, para));



    }
};
```
这样在调用setName函数中，先直接调用add函数，再调用callteacher函数将加法算出的值（10+8=18）输出到consolve中。
model/delegate/view机制
这个这里不具体举例，ListView,TableView,TreeView等需要model的地方，都可以从c++中定义类型，注册到qml中然后使用，严格来说属于第一种方式。
序列化字符串json
这个还是在第一种方式，但是如果需要传递复杂的数据结构，如果安装第一种方式，代码量太可观，各种属性定义，文件变多，对于实时性要求不高，我觉得json可能是最优选择。
示例把一个json对象序列化为字符串，设置到Student中去，再调用getValue方法返回序列化后的数据，通过JSON.parse得到对象，这里为了比对数据，实际使用时，不再序列化为字符串。
main.qml如下

```c
import QtQuick 2.7
import QtQuick.Controls 2.0
import csdn.fish 1.0
ApplicationWindow {
    visible: true
    width: 640
    height: 480
    title: qsTr("Hello World")

    Student{
        id:liming
        Component.onCompleted: {
            var test= {
                name:"liming",
                age:20,
                weight:70,
                height:180,
                sex:"man",
                score:[
                    {
                        math:95,
                        english:86
                    },
                    {
                        math:92,
                        english:81
                    },
                    {
                        math:99,
                        english:90
                    }
                ]
            }
            var before=JSON.stringify(test,null,4)
            liming.setValue(before)
            var temp=liming.getValue()
            var afterObject=JSON.parse(temp)
            var after=JSON.stringify(afterObject,null,4)
            console.info("开始:\r\n",before,"\r\n结束:\r\n",after,before==after)
        }
    }
}
```

C++中如下：

```c
#include <QObject>
#include <QJsonObject>
#include <QJsonArray>
#include <QJsonDocument>

class Data
{
public:
    QString name;
    int age;
    int weight;
    int height;
    QString sex;
    struct Score{
        int math;
        int english;
    }scores[3];
};

class Student : public QObject
{
    Q_OBJECT
public:
    explicit Student(QObject *parent = nullptr)
    {

    }
    Q_INVOKABLE void setValue(QString value)
    {
        QJsonDocument jsonDocument = QJsonDocument::fromJson(value.toUtf8());
        auto root=jsonDocument.object();
        data.name=root["name"].toString();
        data.age=root["age"].toInt();
        data.weight=root["weight"].toInt();
        data.height=root["height"].toInt();
        data.sex=root["sex"].toString();
        QJsonArray score=root["score"].toArray();
        for(auto loop=0;loop!=score.size();++loop){
            auto source=score[loop].toObject();
            auto& dest=data.scores[loop];
            dest.math=source["math"].toInt();
            dest.english=source["english"].toInt();
        }
    }
    Q_INVOKABLE QString getValue()
    {
        QJsonObject root;
        root["name"]=data.name;
        root["age"]=data.age;
        root["weight"]=data.weight;
        root["height"]=data.height;
        root["sex"]=data.sex;
        QJsonArray score;
        for(auto loop=0;loop!=_countof(data.scores);++loop){
            QJsonObject dest;
            const auto& source=data.scores[loop];
            dest["math"]=source.math;
            dest["english"]=source.english;
            score.append(dest);
        }
        root["score"]=score;
        QJsonDocument jsonDocument(root);
        return jsonDocument.toJson();
    }
private:
    Data data;
};

```

输出的结果如下：
qml: 开始:

```json
 {
    "age": 20,
    "height": 180,
    "name": "liming",
    "score": [
        {
            "english": 86,
            "math": 95
        },
        {
            "english": 81,
            "math": 92
        },
        {
            "english": 90,
            "math": 99
        }
    ],
    "sex": "man",
    "weight": 70
} 
```

结束:

```json
 {
    "age": 20,
    "height": 180,
    "name": "liming",
    "score": [
        {
            "english": 86,
            "math": 95
        },
        {
            "english": 81,
            "math": 92
        },
        {
            "english": 90,
            "math": 99
        }
    ],
    "sex": "man",
    "weight": 70
} true
```


