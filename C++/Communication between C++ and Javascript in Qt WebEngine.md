# Communication between C++ and Javascript in Qt WebEngine

As Qt WebKit is replaced by Qt WebEngine(you can refer to this [post](https://wiki.qt.io/QtWebEngine/Porting_from_QtWebKit) about porting issues), accessing html elements from C++ directly becomes impossible. Many works originally done by QWebKit classes are now transferred to javascript. Javascript is used to [manipulate web content](https://doc.qt.io/qt-5.10/qtwebengine-webenginewidgets-contentmanipulation-example.html). And you need to call runJavaScript from C++ code to run javascript on the web page loaded by QWebEngineView.To get web elements, a communication mechanism is invented to bridge the C++ world and the javascript world. The bridging mechanism is more than obtaining the values of web page elements. It provides the ways in which C++ code can call javascript functions, and javascript can invoke C++ functions as well.The values of variables can also be passed from C++ to javascript, and vice versa. Let’s consider the following application scenarios:

## javascript code calls C++ function

C++ code should provide a class which contains the to-be-called function(as a slot), then register an object of this class to a [QWebChannel](http://doc.qt.io/qt-5/qwebchannel.html) object, and set the web channel object to the web page object in the QWebEngineView

```
class WebClass : public QObject
{
    Q_OBJECT
    public slots:
    void jscallme()
    {
        QMessageBox::information(NULL,"jscallme","I'm called by js!");
    }
};

WebClass *webobj = new WebClass();
QWebChannel *channel = new QWebChannel(this);
channel->registerObject("webobj", webobj);
view->page()->setWebChannel(channel);
```



To invoke the C++ function jscallme, javascript should new an instance of QWebChannel object.

```
new QWebChannel(qt.webChannelTransport,
 function(channel){
 var webobj = channel.objects.webobj;
 window.foo = webobj;
 });
```

QWebChannel is defined in [qwebchannel.js](http://doc.qt.io/qt-5/qtwebchannel-javascript.html)(you can find this file in the example folder of Qt installation directory) so the script should be loaded first. In the function passed as the second parameter of function QWebChannel, the exposed object from C++ world(webobj in C++) channel.objects.webobj is assigned to the javascript variable webobj, and then assigned to window.foo so you can use foo elsewhere to refer to the object. After the function is executed, javascript can call the C++ slot function jscallme as follows:

```
foo.jscallme();
```

## Pass data from javascript to C++

We’ve known how to call C++ function from javascript. You should be able to figure out a way to pass data from javascript to C++, i.e., as parameter(s) of function. We re-implement jscallme as follows:

```
void jscallme(const QString &datafromjs)
{
    QMessageBox::information(NULL,"jscallme","I'm called by js!");
    m_data=datafromjs;
}
```

, and invoking of the function from js would be:

```
foo.jscallme(somedata);
```

Note that the “const” before the parameter can not be omitted, otherwise, you will get the following error:
Could not convert argument QJsonValue(string, “sd”) to target type .

Although data can be passed as parameters of function, it would be more convenient if we can pass data by setting an attribute of an object like:

```
foo.someattribute="somedata";
```

We expect after the code is executed, “somedata” will be stored in a member variable of the exposed object (webobj) in C++. This is done by delaring a [qt property](http://doc.qt.io/qt-5/properties.html) in C++ class:

```
class WebClass : public QObject
{
    Q_OBJECT
    Q_PROPERTY(QString someattribute MEMBER m_someattribute)
public slots:
    void jscallme()
    {
        QMessageBox::information(NULL,"jscallme","I'm called by js!");
    }
private:
    QString m_someattribute;
};
```

 

Now if you execute foo.someattribute=”somedata” in javascript, m_someattribute in C++ will be “somedata”.

## Pass data from C++ to javascript

We can send data from C++ to javascript using signals. We emit a signal with the data to send as the parameter of the signal. Javascript must connect the signal to a function to receive the data.

```
class WebClass : public QObject
{
    Q_OBJECT
    Q_PROPERTY(QString someattribute MEMBER m_someattribute)
public slots:
    void jscallme()
    {
        QMessageBox::information(NULL,"jscallme","I'm called by js!");
    }
    void setsomeattribute(QString attr)
    {
        m_someattribute=attr;
       emit someattributeChanged(m_someattribute);
    }
signals:
    void someattributeChanged(QString & attr);
private:
    QString m_someattribute;
};
```

 

```
var updateattribute=function(text)
{
    $("#attrid").val(text);
}
new QWebChannel(qt.webChannelTransport,
function(channel){
var webobj = channel.objects.webobj;
window.foo = webobj;
webobj.someattributeChanged.connect(updateattribute);
});
```

 

The line “webobj.someattributeChanged.connect(updateattribute)” connects the C++ signal someattributeChanged to the javascript function updateattribute. Note that although updateattribute takes one parameter “text”, we did not provide the parameter value in connect. In fact, we do not know the parameter value passed to updateattribute until the signal is received. The signal is accompanied by a parameter “attr” which is passed as the “text” parameter of updateattribute. Now, if you call webobj->setsomeattribute(“hello”), you will see the value of the html element with id “#attrid” is set to “hello”. Note that although we connect the member m_someattribute to the qt property someattribute in the above example, it is not a required step. The signal mechanism alone can realize the delivery of data.
We can make things even simpler by adding the NOTIFY parameter when declaring the someattribute property.
```
class WebClass : public QObject
{
Q_OBJECT
Q_PROPERTY(QString someattribute MEMBER m_someattribute NOTIFY someattributeChanged)
public slots:
    void jscallme()
    {
        QMessageBox::information(NULL,"jscallme","I'm called by js!");
    }

signals:
    void someattributeChanged(QString & attr);
private:
    QString m_someattribute;
};class WebClass : public QObject
{
Q_OBJECT
Q_PROPERTY(QString someattribute MEMBER m_someattribute NOTIFY someattributeChanged)
public slots:
    void jscallme()
    {
        QMessageBox::information(NULL,"jscallme","I'm called by js!");
    }

signals:
    void someattributeChanged(QString & attr);
private:
    QString m_someattribute;
};
```

 

Now, if you call webobj->setProperty(“someattribute”,”hello”) somewhere in C++, the signal “someattributeChanged” is **automatically** emitted and our web page gets updated.

## C++ invokes javascript function

This is much straightforward compared with invoking C++ function from js. Just use [runJavaScript](http://doc.qt.io/qt-5/qwebenginepage.html#runJavaScript-1) passing the function as the parameter as follows:

```
view->page()->runJavaScript("jsfun();",this { qDebug()<<v.toString();});
```



It assumes the jsfun is already defined on your web page, otherwise, you have to define it in the string parameter. The return value is asynchronously passed to the lambda expression as the parameter v.

Now, back to the question raised at the beginning of the post: *How to get the value of an html element in C++?* It can be [done as follows](https://stackoverflow.com/questions/34693830/qwebengine-how-to-get-attribute-value):

```
view->page()->runJavaScript("function getelement(){return $('#elementid').val();} getelement();",this { qDebug()<<v.toString();});
```





It uses jQuery functions so make sure jQuery lib is running on your web page.

Posted in [qt](https://myprogrammingnotes.com/category/qt)

Comments are closed, but [trackbacks](https://myprogrammingnotes.com/communication-c-javascript-qt-webengine.html/trackback) and pingbacks are open.