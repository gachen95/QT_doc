https://www.toptal.com/qt/qt-multithreading-c-plus-plus

# QML Signal and Handler Event System





QML utilizes Qt's [meta-object](https://doc.qt.io/archives/qt-4.8/metaobjects.html) and [signals](https://doc.qt.io/archives/qt-4.8/signalsandslots.html) systems. Signals and slots created using Qt in C++ are inheritely valid in QML.



## Signals and Handlers

Signals provide a way to notify other objects when an event has occurred. For example, the [MouseArea](https://doc.qt.io/archives/qt-4.8/qml-mousearea.html) `clicked` signal notifies other objects that the mouse has been clicked within the area.

The syntax for defining a new signal is:

```
signal [([ [, ...]])]
```

Attempting to declare two signals or methods with the same name in the same type block generates an error. However, a new signal may reuse the name of an existing signal on the type. (This should be done with caution, as the existing signal may be hidden and become inaccessible.)

Here are various examples of signal declarations:

```
Rectangle {
    signal trigger
    signal send (string notice)
    signal perform (string task, variant object)
}
```

If the signal has no parameters, the "`()`" brackets are optional. If parameters are used, the parameter types must be declared, as for the `string` and `variant` arguments of the `perform` signal.

Adding a signal to an item automatically adds a *signal handler* as well. The signal hander is named `on`, with the first letter of the signal in uppercase. The previous signals have the following signal handlers:

```
onTrigger: console.log("trigger signal emitted")

onSend: {
    console.log("send signal emitted with notice: " + notice)
}

onPerform: console.log("perform signal emitted")
```

Further, each QML properties have a `Changed` signal and its corresponding `onChanged` signal handler. As a result, property changes may notify other components for any changes.

```
Rectangle {
    id: sprite
    width: 25; height: 25
    x: 50; y: 15

    onXChanged: console.log("x property changed, emitted xChanged signal")
    onYChanged: console.log("y property changed, emitted yChanged signal")
}
```

To emit a signal, invoke it as a method. The signal handler binding is similar to a property binding and it is invoked when the signal is emitted. Use the defined argument names to access the respective arguments.

```c
Rectangle {
    id: messenger

    signal send( string person, string notice)

    onSend: {
        console.log("For " + person + ", the notice is: " + notice)
    }

    Component.onCompleted: messenger.send("Tom", "the door is ajar.")
}
```

Note that the `Component.onCompleted` is an [attached signal handler](https://doc.qt.io/archives/qt-4.8/propertybinding.html#attached-signalhandlers); it is invoked when the [Component](https://doc.qt.io/archives/qt-4.8/qml-component.html) initialization is complete.



### Connecting Signals to Methods and Signals

Signal objects have a `connect()` method to a connect a signal either to a method or another signal. When a signal is connected to a method, the method is automatically invoked whenever the signal is emitted. (In Qt terminology, the method is a *slot* that is connected to the *signal*; all methods defined in QML are created as [Qt slots](https://doc.qt.io/archives/qt-4.8/signalsandslots.html).) This enables a signal to be received by a method instead of a [signal handler](https://doc.qt.io/archives/qt-4.8/qdeclarativeintroduction.html#signal-handlers).

```
Rectangle {
    id: relay

    signal send( string person, string notice)
    onSend: console.log("Send signal to: " + person + ", " + notice)

    Component.onCompleted: {
        relay.send.connect(sendToPost)
        relay.send.connect(sendToTelegraph)
        relay.send.connect(sendToEmail)
        relay.send("Tom", "Happy Birthday")
    }

    function sendToPost(person, notice) {
        console.log("Sending to post: " + person + ", " + notice)
    }
    function sendToTelegraph(person, notice) {
        console.log("Sending to telegraph: " + person + ", " + notice)
    }
    function sendToEmail(person, notice) {
        console.log("Sending to email: " + person + ", " + notice)
    }
}
```

The `connect()` method is appropriate when connecting a JavaScript method to a signal.

There is a corresponding `disconnect()` method for removing connected signals.



#### Signal to Signal Connect

By connecting signals to other signals, the `connect()` method can form different signal chains.

```
Rectangle {
    id: forwarder
    width: 100; height: 100

    signal send()
    onSend: console.log("Send clicked")

    MouseArea {
        id: mousearea
        anchors.fill: parent
        onClicked: console.log("MouseArea clicked")
    }
    Component.onCompleted: {
        mousearea.clicked.connect(send)
    }
}
```

Whenever the [MouseArea](https://doc.qt.io/archives/qt-4.8/qml-mousearea.html) `clicked` signal is emitted, the `send` signal will automatically be emitted as well.

```
output:
    MouseArea clicked
    Send clicked
```



## C++ Additions

Because QML uses Qt, a signal defined in C++ also works as a QML signal. The signal may be emitted in QML code or called as a method. In addition, the QML runtime automatically creates signal handlers for the C++ signals. For more signal control, the `connect()` method and the [Connections](https://doc.qt.io/archives/qt-4.8/qml-connections.html) element may connect a C++ signal to another signal or method.

For complete information on how to call C++ functions in QML, read the [Extending QML - Signal Support Example](https://doc.qt.io/archives/qt-4.8/qt-declarative-cppextensions-referenceexamples-signal-example.html).