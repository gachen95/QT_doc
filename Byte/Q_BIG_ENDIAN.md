# [Getting most significant byte of qint64](https://stackoverflow.com/questions/13347186/getting-most-significant-byte-of-qint64)

https://doc.qt.io/qt-5/qtglobal.html#Q_BYTE_ORDER

This is more of a C/C++ question than Qt. But anyway:

```cpp
qint64 a = 56747234992934;
union {
    qint64 i64;
    int8_t i8[8];
} u = {a};
#if Q_BYTE_ORDER == Q_BIG_ENDIAN
qDebug() << u.i8[0]; // MSB is the first byte on big endian machines
#else
qDebug() << u.i8[7]; // MSB is the last byte on little endian machines
#endif
```

Edit: To avoid messy endian specific position code:

```cpp
qint64 a = 56747234992934;
union {
    qint64 i64;
    int8_t i8[8];
} u = {qToBigEndian(a)};
qDebug() << u.i8[0]; // MSB is the first byte on big endian machines
```

note that you need to include `qendian.h` for this to work.

This is a destructive solution. It destroys the content of `a` and maybe some brain tissue of the unwary code reviewer. It is also aesthetically pleasing.

```cpp
int8_t a8 = a;
int8_t a7 = a >>= 8;
int8_t a6 = a >>= 8;
int8_t a5 = a >>= 8;
int8_t a4 = a >>= 8;
int8_t a3 = a >>= 8;
int8_t a2 = a >>= 8;
int8_t a1 = a >>= 8;
```

A better solution, probably both in performance and readability and preserves the content of a:

```cpp
int8_t a8 = a;
int8_t a7 = a >> 8;
int8_t a6 = a >> 16;
int8_t a5 = a >> 24;
int8_t a4 = a >> 32;
int8_t a3 = a >> 40;
int8_t a2 = a >> 48;
int8_t a1 = a >> 56;
```