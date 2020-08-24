# How to Use a Custom Class in C++ Model and QML View

https://wiki.qt.io/How_to_Use_a_Custom_Class_in_C%2B%2B_Model_and_QML_View

### The Problem

Sometimes you might want to use a custom class as data of a model that you'll then display in QML. While the process is pretty straightforward, it might be difficult to put together all the infos you need so we'll summarise the process using a small example.

### Letting QML use your custom class

Say we have a pretty simple custom class, 3 members with getter and setter functions plus a const method that does a simple calculation

```
#include <QString>
#include <QDate>
class Person
{
public:
    Person(const QString& name ,const QString& surname,const QDate& dob)
        :m_birthDate(dob),m_name(name),m_surname(surname){}
    Person() = default;
    Person(const Person& other)=default;
    Person& operator=(const Person& other)=default;
    const QDate& birthDate() const { return m_birthDate; }
    void setBirthDate(const QDate& birthDate) { m_birthDate = birthDate; }
    const QString& name() const { return m_name; }
    void setName(const QString &name) { m_name = name; }
    const QString& surname() const { return m_surname; }
    void setSurname(const QString &surname) { m_surname = surname; }
    int age() const {
        if(m_birthDate.isNull())
            return -1;
        const QDate today = QDate::currentDate();
        return 
            (((12*(today.year()-m_birthDate.year()))+today.month()-m_birthDate.month())/12)
            - (today.month()==m_birthDate.month() && today.day()<m_birthDate.day()) ? 1:0
        ;
    }
private:
    QDate m_birthDate;
    QString m_name;
    QString m_surname;
};
```

The first thing we need to do is allow QML to interact with this class. To achieve this we'll make it a `Q_GADGET`. A `Q_GADGET` is a lighter version of `Q_OBJECT`: it doesn't allow you to use signals, slots and the parent/child ownership system but allows you to use `Q_PROPERTY` and `Q_INVOKABLE`.

```
#include <QString>
#include <QDate>
#include <QObject>
class Person
{
    Q_GADGET
    Q_PROPERTY(QString name READ name WRITE setName)
    Q_PROPERTY(QString surname READ surname WRITE setSurname)
    Q_PROPERTY(QDate birthDate READ birthDate WRITE setBirthDate)
    Q_PROPERTY(int age READ age)
public:
    Person(const QString& name ,const QString& surname,const QDate& dob)
        :m_birthDate(dob),m_name(name),m_surname(surname){}
    Person() = default;
    Person(const Person& other)=default;
    Person& operator=(const Person& other)=default;
    const QDate& birthDate() const { return m_birthDate; }
    void setBirthDate(const QDate& birthDate) { m_birthDate = birthDate; }
    const QString& name() const { return m_name; }
    void setName(const QString &name) { m_name = name; }
    const QString& surname() const { return m_surname; }
    void setSurname(const QString &surname) { m_surname = surname; }
    int age() const {
        if(m_birthDate.isNull())
            return -1;
        const QDate today = QDate::currentDate();
        return 
            (((12*(today.year()-m_birthDate.year()))+today.month()-m_birthDate.month())/12)
            - (today.month()==m_birthDate.month() && today.day()<m_birthDate.day()) ? 1:0
        ;
    }
private:
    QDate m_birthDate;
    QString m_name;
    QString m_surname;
};
```

Here we declared 4 properties of the object, 3 for the members and one for the age calculation.

### Using your class in a C++ model

This part is straightforward, we'll build a `QObject` that fills a list model with dummy data and exposes it in a way QML can manage

```
#include <QStandardItemModel>
#include "person.h"
class People : public QObject{
    Q_OBJECT
    Q_PROPERTY(QAbstractItemModel* model READ model)
    Q_DISABLE_COPY(People)
public:
    explicit People(QObject* parent = Q_NULLPTR)
        :QObject(parent)
    {
        m_model=new QStandardItemModel(this);
        m_model->insertColumn(0);
        fillDummyData();
    }
    Q_SLOT void addPerson(const QString& name ,const QString& surname,const QDate& dob){
        if(name.isEmpty() && surname.isEmpty() && dob.isNull())
            return;
        const int newRow= m_model->rowCount();
        const Person newPerson(name,surname,dob);
        m_model->insertRow(newRow);
        m_model->setData(m_model->index(newRow,0),QVariant::fromValue(newPerson),Qt::EditRole);
    }
    QAbstractItemModel* model() const {return m_model;}
private:
    void fillDummyData(){
        addPerson(QStringLiteral("Albert"),QStringLiteral("Einstein"),QDate(1879,3,14));
        addPerson(QStringLiteral("Robert"),QStringLiteral("Oppenheimer"),QDate(1904,4,22));
        addPerson(QStringLiteral("Enrico"),QStringLiteral("Fermi"),QDate(1901,9,29));
    }
    QAbstractItemModel* m_model;
};
```

We expose the model through a property and we also added a slot to add a new person to the list. We used `QStandardItemModel` here just for convenience.

### Passing to QML

The main function is very simple, the only change we made to QtCreator's default is creating the `People` object and passing it to the root context of QtQuick

```
#include <QGuiApplication>
#include <QQmlContext>
#include <QQmlApplicationEngine>
#include "people.h"
int main(int argc, char *argv[])
{
#if defined(Q_OS_WIN)
    QCoreApplication::setAttribute(Qt::AA_EnableHighDpiScaling);
#endif
    QGuiApplication app(argc, argv);
    QQmlApplicationEngine engine;
    People people;
    engine.rootContext()->setContextProperty("people", &people);
    engine.load(QUrl(QStringLiteral("qrc:/main.qml")));
    if (engine.rootObjects().isEmpty())
        return -1;
    return app.exec();
}
```

### Using the data in QML

We'll display the data in a ListView

```
import QtQml 2.2
import QtQuick 2.7
import QtQuick.Window 2.2
import QtQuick.Layouts 1.3
import QtQuick.Controls 2.3

Window {
    visible: true
    width: 640
    height: 480
    title: qsTr("Custom Class in C++ Model and QML View")

    GridLayout {
        columns: 3
        anchors.fill: parent

        Text{
            text: "Name"
            Layout.row: 0
            Layout.column: 0
        }

        TextField  {
            id: inputName
            Layout.row: 1
            Layout.column: 0
            focus: true

        }

        Text{
            text: "Surname"
            Layout.row: 0
            Layout.column: 1
        }

        TextField  {
            id: inputSurname
            Layout.row: 1
            Layout.column: 1
        }

        Text{
            text: "Date of Birth"
            Layout.row: 0
            Layout.column: 2
        }

        TextField  {
            id: inputDob
            Layout.row: 1
            Layout.column: 2
            validator: RegExpValidator{regExp: /\d{4}-\d{2}-\d{2}/}
        }

        Button {
            text: "Add"
            Layout.row: 2
            Layout.column: 0
            Layout.columnSpan: 3
            onClicked: {
                people.addPerson(inputName.text, inputSurname.text, Date.fromLocaleString(Qt.locale(),inputDob.text,"yyyy-MM-dd"))
            }
        }

        ListView {
            width: 200; height: 250
            model: people.model
            delegate: Text {
                text: edit.name + " " + edit.surname +" "+ edit.birthDate.toLocaleString(Qt.locale(),"yyyy-MM-dd") + " ("+ edit.age +")"
            }
            Layout.row: 3
            Layout.column: 0
            Layout.columnSpan: 3
        }
    }
}
```

The first thing to notice is that we used the model property from People as the `ListView`'s model. Now let's focus on the delegate, as you can see accessing our data is very easy we just use the `roleName.propertyName` syntax. Since we saved our data in `Qt::EditRole` we used `edit` as role name (a list of the default role names is [In `QAbstractItemModel`'s documentation](http://doc.qt.io/qt-5/qabstractitemmodel.html#roleNames)) and then accesses directly all the properties of `Person` directly