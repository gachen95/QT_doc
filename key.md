Note: For more complex interaction, [Qt Quick Input Handlers](https://doc-snapshots.qt.io/qt5-dev/qtquickhandlers-index.html) where introduced with Qt 5.12. They are intended to be used instead of elements such as MouseArea and Flickable and offer greater control and flexibility.
The idea is to handle one interaction aspect in each handler instance instead of centralizing the handling of all events from a given source in a single element, which was the case before.

