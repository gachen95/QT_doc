https://martin.rpdev.net/2018/03/30/using-fontawesome-5-in-qml.html

# [How to use FontAwesome in Qt](https://stackoverflow.com/questions/42742551/how-to-use-fontawesome-in-qt)



# How to make a quick custom Qt/QML checkbox using icon fonts

https://medium.com/@eduard.metzger/how-to-make-a-quick-custom-qml-checkbox-using-icon-fonts-b2ffbd651144

![img](https://miro.medium.com/max/432/1*Gs7HVHylINy08e3CZ0Gfmw.png)

Here we will create a custom checkbox with relatively little code by reusing one of the paid or free icon fonts normally used for the web. This way we don’t have to design much of the checkbox. Just plug everything together.

First download your favourite icon font like [Font-Awesome](http://fortawesome.github.io/Font-Awesome/icons/) or [search Google](https://www.google.com/?q=free+icon+font) for alternatives. A prerequisite is that the font must have a checkbox and empty box icon for this tutorial to work. I like Font-Awesome, because it is easy to search icons and pick the unicode directly from the website. Here is how you use it in QML. After downloading, unzipping and adding the .ttf files to your project, load it like this anywhere in your QML file:

```
FontLoader { id: fontAwesome; source: "qrc:///res/fontawesome-webfont.ttf" }
```

Search the unicode for a checkbox and an empty box on the icon font website or by searching the .css files, which are mostly in the same zip file.
For Font-Awesome its f046 (checkmark in square) and f096 (square).

Now we will build the checkbox with a text right beside it:

```
Rectangle
{
    id: checkBox
    property bool isToggled: true
    width: childrenRect.width
    height: childrenRect.height
    color: "transparent"    
    Row
    {
        spacing: 5        // Checkbox icons
        Text {
            font.family: fontAwesome.name
            color: "#0091f8"
            font.pixelSize: 16
            text: checkBox.isToggled ? "\uf046" : "\uf096" 
            width: 15
        }        // Checkbox text
        Text
        {
            color: "#222"
            font.family: "Lato"
            text: "Checkbox text"
            font.pixelSize: 12
        }
    }    
    
    MouseArea {
        anchors.fill: parent
        onClicked: {
            checkBox.isToggled = !checkBox.isToggled
        }
    }
}
```

Adapt colours as required. The checkbox and empty box icons are switched automatically through the isToggle property, which is inverted on every click on the text or on the checkbox. Done! You can pack this in a separate QML file and give it a property to define the text from outside.

Hit the recommend (heart icon) button below, if you like this and want to see more tricks.

------

*Originally published* [*on my blog*](https://eduardmetzger.wordpress.com/)*.*