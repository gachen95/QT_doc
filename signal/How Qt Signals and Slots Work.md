# How Qt Signals and Slots Work

Qt is well known for its signals and slots mechanism. But how does it work? In this blog post, we will explore the internals of QObject and QMetaObject and discover how signals and slot work under the hood.

In this blog article, I show portions of Qt5 code, sometimes edited for formatting and brevity.

## Signals and Slots

First, let us recall how signals and slots look like by showing the [official example](http://qt-project.org/doc/qt-4.8/signalsandslots.html).

Hover over the code to see fancy tool tips powered by the [Woboq Code Browser](https://code.woboq.org/)!

The header looks like this:

```
class Counter : public QObject
{
    Q_OBJECT
    int m_value;
public:
    int value() const { return m_value; }
public slots:
    void setValue(int value);
signals:
    void valueChanged(int newValue);
};
```

Somewhere in the `.cpp` file, we implement `setValue()`

```
void Counter::setValue(int value)
{
    if (value != m_value) {
        m_value = value;
        emit valueChanged(value);
    }
}
```

Then one can use this Counter object like this:

```
  Counter a, b;
  QObject::connect(&a, SIGNAL(valueChanged(int)),
                   &b, SLOT(setValue(int)));

  a.setValue(12);  // a.value() == 12, b.value() == 12
```

This is the original syntax that has almost not changed since the beginning of Qt in 1992.

But even if the basic API has not changed since the beginning, its implementation has been changed several times. New features have been added and a lot happened under the hood. There is no magic involved and this blog post will show you how it works.

## MOC, the Meta Object Compiler

The Qt signals/slots and property system are based on the ability to introspect the objects at runtime. Introspection means being able to list the methods and properties of an object and have all kinds of information about them such as the type of their arguments.
QtScript and QML would have hardly been possible without that ability.

C++ does not offer introspection support natively, so Qt comes with a tool to provide it. That tool is MOC. It is a *code generator* (and NOT a preprocessor like some people call it).

It parses the header files and generates an additional C++ file that is compiled with the rest of the program. That generated C++ file contains all the information required for the introspection.

Qt has sometimes been criticized by language purists because of this extra code generator. I will let the [Qt documentation respond to this criticism](http://qt-project.org/doc/qt-4.8/templates.html). There is nothing wrong with code generators and the MOC is of a great help.

## Magic Macros

Can you spot the *keywords* that are not pure C++ keywords? `signals, slots, Q_OBJECT, emit, SIGNAL, SLOT`. Those are known as the Qt extension to C++. They are in fact simple macros, defined in [qobjectdefs.h](https://code.woboq.org/qt5/qtbase/src/corelib/kernel/qobjectdefs.h.html#66)

```
#define signals public``#define slots /* nothing */
```

That is right, signals and slots are simple functions: the compiler will handle them them like any other functions. The macros still serve a purpose though: the `MOC` will see them.

Signals were protected in Qt4 and before. They are becoming public in Qt5 in order to enable [the new syntax](https://woboq.com/blog/new-signals-slots-syntax-in-qt5.html).

```
#define Q_OBJECT \``public``: \``  ``static` `const` `QMetaObject` `staticMetaObject; \``  ``virtual` `const` `QMetaObject` `*metaObject() ``const``; \``  ``virtual` `void` `*qt_metacast(``const` `char` `*); \``  ``virtual` `int` `qt_metacall(``QMetaObject``::Call, ``int``, ``void` `**); \``  ``QT_TR_FUNCTIONS` `/* translations helper */` `\``private``: \``  ``Q_DECL_HIDDEN ``static` `void` `qt_static_metacall(``QObject` `*, ``QMetaObject``::Call, ``int``, ``void` `**);
```

`Q_OBJECT` defines a bunch of functions and a static `QMetaObject` Those functions are implemented in the file generated by MOC.

```
#define emit /* nothing */
```

`emit` is an empty macro. It is not even parsed by MOC. In other words, emit is just optional and means nothing (except being a hint to the developer).

```
Q_CORE_EXPORT ``const` `char` `*qFlagLocation(``const` `char` `*method);``#ifndef QT_NO_DEBUG``# define QLOCATION "\0" __FILE__ ":" QTOSTRING(__LINE__)``# define SLOT(a)   qFlagLocation("1"#a QLOCATION)``# define SIGNAL(a)  qFlagLocation("2"#a QLOCATION)``#else``# define SLOT(a)   "1"#a``# define SIGNAL(a)  "2"#a``#endif
```

Those macros just use the preprocessor to convert the parameter into a string, and add a code in front.

In debug mode we also annotate the string with the file location for a warning message if the signal connection did not work. This was added in Qt 4.5 in a compatible way. In order to know which strings have the line information, we use `qFlagLocation` which will register the string address in a table with two entries.

## MOC Generated Code

We will now go over portion of the code generated by `moc` in Qt5.

### The QMetaObject

```
const QMetaObject Counter::staticMetaObject = {
    { &QObject::staticMetaObject, qt_meta_stringdata_Counter.data,
      qt_meta_data_Counter,  qt_static_metacall, Q_NULLPTR, Q_NULLPTR}
};


const QMetaObject *Counter::metaObject() const
{
    return QObject::d_ptr->metaObject ? QObject::d_ptr->dynamicMetaObject() : &staticMetaObject;
}
```

We see here the implementation of `Counter::metaObject()` and `Counter::staticMetaObject`. They are declared in the `Q_OBJECT` macro. `QObject::d_ptr->metaObject` is only used for dynamic meta objects (QML Objects), so in general, the virtual function `metaObject()` just returns the `staticMetaObject` of the class.

The `staticMetaObject` is constructed in the read-only data. QMetaObject as defined in [qobjectdefs.h](https://code.woboq.org/qt5/qtbase/src/corelib/kernel/qobjectdefs.h.html#QMetaObject) looks like this:

```
struct QMetaObject
{
    /* ... Skiped all the public functions ... */

    enum Call { InvokeMetaMethod, ReadProperty, WriteProperty, /*...*/ };

    struct { // private data
        const QMetaObject *superdata;
        const QByteArrayData *stringdata;
        const uint *data;
        typedef void (*StaticMetacallFunction)(QObject *, QMetaObject::Call, int, void **);
        StaticMetacallFunction static_metacall;
        const QMetaObject **relatedMetaObjects;
        void *extradata; //reserved for future use
    } d;
};
```

The `d` indirection is there to symbolize that all the member should be private. They are not private in order to keep it a POD and allow static initialization.

The `QMetaObject` is initialized with the meta object of the parent object (`QObject::staticMetaObject` in this case) as `superdata`. `stringdata` and `data` are initialized with some data explained later in this article. `static_metacall` is a function pointer initialized to `Counter::qt_static_metacall`.

### Introspection Tables

First, let us analyze the integer data of QMetaObject.

```
static const uint qt_meta_data_Counter[] = {

 // content:
       7,       // revision
       0,       // classname
       0,    0, // classinfo
       2,   14, // methods
       0,    0, // properties
       0,    0, // enums/sets
       0,    0, // constructors
       0,       // flags
       1,       // signalCount

 // signals: name, argc, parameters, tag, flags
       1,    1,   24,    2, 0x06 /* Public */,

 // slots: name, argc, parameters, tag, flags
       4,    1,   27,    2, 0x0a /* Public */,

 // signals: parameters
    QMetaType::Void, QMetaType::Int,    3,

 // slots: parameters
    QMetaType::Void, QMetaType::Int,    5,

       0        // eod
};
```

The first 13 `int` consists of the header. When there are two columns, the first column is the count and the second column is the index in this array where the description starts.
In this case we have 2 methods, and the methods description starts at index 14.

The method descriptions are composed of 5 `int`. The first one is the name, it is an index in the string table (we will look into the details later). The second integer is the number of parameters, followed by the index at which one can find the parameter description. We will ignore the tag and flags for now. For each function, `moc` also saves the return type of each parameter, their type and index to the name.

### String Table

```
struct qt_meta_stringdata_Counter_t {
    QByteArrayData data[6];
    char stringdata0[46];
};
#define QT_MOC_LITERAL(idx, ofs, len) \
    Q_STATIC_BYTE_ARRAY_DATA_HEADER_INITIALIZER_WITH_OFFSET(len, \
    qptrdiff(offsetof(qt_meta_stringdata_Counter_t, stringdata0) + ofs \
        - idx * sizeof(QByteArrayData)) \
    )
static const qt_meta_stringdata_Counter_t qt_meta_stringdata_Counter = {
    {
QT_MOC_LITERAL(0, 0, 7), // "Counter"
QT_MOC_LITERAL(1, 8, 12), // "valueChanged"
QT_MOC_LITERAL(2, 21, 0), // ""
QT_MOC_LITERAL(3, 22, 8), // "newValue"
QT_MOC_LITERAL(4, 31, 8), // "setValue"
QT_MOC_LITERAL(5, 40, 5) // "value"

    },
    "Counter\0valueChanged\0\0newValue\0setValue\0"
    "value"
};
#undef QT_MOC_LITERAL
```

This is basically a static array of `QByteArray`. the `QT_MOC_LITERAL` macro creates a static `QByteArray` that references a particular index in the string below.

### Signals

The MOC also implements the signals. They are simple functions that just create an array of pointers to the arguments and pass that to `QMetaObject::activate`. The first element of the array is the return value. In our example it is 0 because the return value is `void`.
The 3rd parameter passed to activate is the signal index (0 in that case).

```
// SIGNAL 0
void Counter::valueChanged(int _t1)
{
    void *_a[] = { Q_NULLPTR, const_cast<void*>(reinterpret_cast<const void*>(&_t1)) };
    QMetaObject::activate(this, &staticMetaObject, 0, _a);
}
```

### Calling a Slot

It is also possible to call a slot by its index in the `qt_static_metacall` function:

```
void Counter::qt_static_metacall(QObject *_o, QMetaObject::Call _c, int _id, void **_a)
{
    if (_c == QMetaObject::InvokeMetaMethod) {
        Q_ASSERT(staticMetaObject.cast(_o));
        Counter *_t = static_cast<Counter *>(_o);
        Q_UNUSED(_t)
        switch (_id) {
        case 0: _t->valueChanged((*reinterpret_cast< int(*)>(_a[1]))); break;
        case 1: _t->setValue((*reinterpret_cast< int(*)>(_a[1]))); break;
        default: ;
        }
```

The array pointers to the argument is the same format as the one used for the signal. `_a[0]` is not touched because everything here returns void.

## A Note About Indexes.

In each QMetaObject, the slots, signals and other invokable methods of that object are given an index, starting from 0. They are ordered so that the signals come first, then the slots and then the other methods. This index is called internally the **relative index**. They do not include the indexes of the parents.

But in general, we do not want to know a more global index that is not relative to a particular class, but include all the other methods in the inheritance chain. To that, we just add an offset to that relative index and get the **absolute index**. It is the index used in the public API, returned by functions like `QMetaObject::indexOf{Signal,Slot,Method}`

The connection mechanism uses a vector indexed by signals. But all the slots waste space in the vector and there are usually more slots than signals in an object. So from Qt 4.6, a new internal **signal index** which only includes the signal index is used.

While developing with Qt, you only need to know about the absolute method index. But while browsing the Qt's QObject source code, you must be aware of the difference between those three.

## How Connecting Works.

The first thing Qt does when doing a connection is to find out the index of the signal and the slot. Qt will look up in the string tables of the meta object to find the corresponding indexes.

Then a `QObjectPrivate::Connection` object is created and added in the internal linked lists.

What information needs to be stored for each connection? We need a way to quickly access the connections for a given signal index. Since there can be several slots connected to the same signal, we need for each signal to have a list of the connected slots. Each connection must contain the receiver object, and the index of the slot. We also want the connections to be automatically destroyed when the receiver is destroyed, so each receiver object needs to know who is connected to him so he can clear the connection.

Here is the `QObjectPrivate::Connection` as defined in [qobject_p.h](https://code.woboq.org/qt5/qtbase/src/corelib/kernel/qobject_p.h.html#QObjectPrivate::Connection) :

```
struct QObjectPrivate::Connection
{
    QObject *sender;
    QObject *receiver;
    union {
        StaticMetaCallFunction callFunction;
        QtPrivate::QSlotObjectBase *slotObj;
    };
    // The next pointer for the singly-linked ConnectionList
    Connection *nextConnectionList;
    //senders linked list
    Connection *next;
    Connection **prev;
    QAtomicPointer<const int> argumentTypes;
    QAtomicInt ref_;
    ushort method_offset;
    ushort method_relative;
    uint signal_index : 27; // In signal range (see QObjectPrivate::signalIndex())
    ushort connectionType : 3; // 0 == auto, 1 == direct, 2 == queued, 4 == blocking
    ushort isSlotObject : 1;
    ushort ownArgumentTypes : 1;
    Connection() : nextConnectionList(0), ref_(2), ownArgumentTypes(true) {
        //ref_ is 2 for the use in the internal lists, and for the use in QMetaObject::Connection
    }
    ~Connection();
    int method() const { return method_offset + method_relative; }
    void ref() { ref_.ref(); }
    void deref() {
        if (!ref_.deref()) {
            Q_ASSERT(!receiver);
            delete this;
        }
    }
};
```

Each object has then a connection vector: It is a vector which associates for each of the signals a linked lists of `QObjectPrivate::Connection`.
Each object also has a reversed lists of connections the object is connected to for automatic deletion. It is a doubly linked list.

![img](https://woboq.com/blog/how-qt-signals-slots-work/qobject_connection.png)

Linked lists are used because they allow to quickly add and remove objects. They are implemented by having the pointers to the next/previous nodes within `QObjectPrivate::Connection`
Note that the `prev` pointer of the `senderList` is a pointer to a pointer. That is because we don't really point to the previous node, but rather to the pointer to the next in the previous node. This pointer is only used when the connection is destroyed, and not to iterate backwards. It allows not to have a special case for the first item.

![img](https://woboq.com/blog/how-qt-signals-slots-work/qobject_connection_node.png)

## Signal Emission

When we call a signal, we have seen that it calls the MOC generated code which calls `QMetaObject::activate`.

Here is an annotated version of its implementation from [qobject.cpp](https://code.woboq.org/qt5/qtbase/src/corelib/kernel/qobject.cpp.html#_ZN11QMetaObject8activateEP7QObjectPKS_iPPv)

```
void QMetaObject::activate(QObject *sender, const QMetaObject *m, int local_signal_index,
                           void **argv)
{
    activate(sender, QMetaObjectPrivate::signalOffset(m), local_signal_index, argv);
    /* We just forward to the next function here. We pass the signal offset of
     * the meta object rather than the QMetaObject itself
     * It is split into two functions because QML internals will call the later. */
}

void QMetaObject::activate(QObject *sender, int signalOffset, int local_signal_index, void **argv)
{
    int signal_index = signalOffset + local_signal_index;

    /* The first thing we do is quickly check a bit-mask of 64 bits. If it is 0,
     * we are sure there is nothing connected to this signal, and we can return
     * quickly, which means emitting a signal connected to no slot is extremely
     * fast. */
    if (!sender->d_func()->isSignalConnected(signal_index))
        return; // nothing connected to these signals, and no spy

    /* ... Skipped some debugging and QML hooks, and some sanity check ... */

    /* We lock a mutex because all operations in the connectionLists are thread safe */
    QMutexLocker locker(signalSlotLock(sender));

    /* Get the ConnectionList for this signal.  I simplified a bit here. The real code
     * also refcount the list and do sanity checks */
    QObjectConnectionListVector *connectionLists = sender->d_func()->connectionLists;
    const QObjectPrivate::ConnectionList *list =
        &connectionLists->at(signal_index);

    QObjectPrivate::Connection *c = list->first;
    if (!c) continue;
    // We need to check against last here to ensure that signals added
    // during the signal emission are not emitted in this emission.
    QObjectPrivate::Connection *last = list->last;

    /* Now iterates, for each slot */
    do {
        if (!c->receiver)
            continue;

        QObject * const receiver = c->receiver;
        const bool receiverInSameThread = QThread::currentThreadId() == receiver->d_func()->threadData->threadId;

        // determine if this connection should be sent immediately or
        // put into the event queue
        if ((c->connectionType == Qt::AutoConnection && !receiverInSameThread)
            || (c->connectionType == Qt::QueuedConnection)) {
            /* Will basically copy the argument and post an event */
            queued_activate(sender, signal_index, c, argv);
            continue;
        } else if (c->connectionType == Qt::BlockingQueuedConnection) {
            /* ... Skipped ... */
            continue;
        }

        /* Helper struct that sets the sender() (and reset it backs when it
         * goes out of scope */
        QConnectionSenderSwitcher sw;
        if (receiverInSameThread)
            sw.switchSender(receiver, sender, signal_index);

        const QObjectPrivate::StaticMetaCallFunction callFunction = c->callFunction;
        const int method_relative = c->method_relative;
        if (c->isSlotObject) {
            /* ... Skipped....  Qt5-style connection to function pointer */
        } else if (callFunction && c->method_offset <= receiver->metaObject()->methodOffset()) {
            /* If we have a callFunction (a pointer to the qt_static_metacall
             * generated by moc) we will call it. We also need to check the
             * saved metodOffset is still valid (we could be called from the
             * destructor) */
            locker.unlock(); // We must not keep the lock while calling use code
            callFunction(receiver, QMetaObject::InvokeMetaMethod, method_relative, argv);
            locker.relock();
        } else {
            /* Fallback for dynamic objects */
            const int method = method_relative + c->method_offset;
            locker.unlock();
            metacall(receiver, QMetaObject::InvokeMetaMethod, method, argv);
            locker.relock();
        }

        // Check if the object was not deleted by the slot
        if (connectionLists->orphaned) break;
    } while (c != last && (c = c->nextConnectionList) != 0);
}
```

## Conclusion

We saw how connections are made and how signals slots are emitted. What we have not seen is the implementation of [the new Qt5 syntax](https://woboq.com/blog/new-signals-slots-syntax-in-qt5.html), but that will be for another post.

**Update:** [The part 2 is available](https://woboq.com/blog/how-qt-signals-slots-work-part2-qt5.html).