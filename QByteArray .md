# Converting a QByteArray into an Unsigned Integer

2018/03/28[Burkhard Stubert](https://www.embeddeduse.com/author/bvs/)

My program received messages from a network as an array of bytes. The sender of the messages encoded integers in Big Endian format. So, the hex number 0x1540 arrived in the sequence [0x15, 0x40]. I wrote the following function, which converts such a byte array of at most four bytes into a 32-bit unsigned integer. The function converts the sequence [0x15, 0x40] into the hex number 0x1540.

```cpp
quint32 byteArrayToUint32(const QByteArray &bytes)
{
    auto count = bytes.size();
    if (count == 0 || count > 4) {
        return 0;
    }
    quint32 number = 0U;
    for (int i = 0; i < count; ++i) {
        auto b = static_cast<quint32>(bytes[count - 1 - i]);
        number += static_cast<quint32>(b << (8 * i));
    }
    return number;
}
```

Of course, I also wrote a unit test.

```cpp
void TestByteArrayToUint32::testByteArrayToUint32()
{
    QCOMPARE(byteArrayToUint32(QByteArray::fromHex("9020562f")), 0x9020562fU);
    QCOMPARE(byteArrayToUint32(QByteArray::fromHex("314922")), 0x314922U);
    QCOMPARE(byteArrayToUint32(QByteArray::fromHex("1540")), 0x1540U);
    QCOMPARE(byteArrayToUint32(QByteArray::fromHex("38")), 0x38U);
    QCOMPARE(byteArrayToUint32(QByteArray::fromHex("")), 0x0U);
    QCOMPARE(byteArrayToUint32(QByteArray::fromHex("559020562f")), 0x0U);
}
```

All QCOMPARE checks passed. All seemed fine. After using my function for a (short) while, a test didn't want to pass. I tracked the problem down to my function. Converting the byte array [0x01, 0x80] yielded the result 0x80.

Can you spot the bug in my code - and in my test?

When I single-stepped through the for-loop, I noticed that `b = 4294967168 = 0xFFFFFF80` for `i = 0`. Huh?

The hex representation 0xFFFFFF80 gives away the problem. It is the two's complement representation of -128 for 32-bit integers. The hex number 0x80 is interpreted as a negative integer, because QByteArray uses (signed) char to store bytes internally. Casting -128 to a quint32 yields the two's complement. The fix is to cast to quint8.

```cpp
        auto b = static_cast<quint8>(bytes[count - 1 - i]);
```

But why didn't my unit test catch the problem?

The last two checks are caught by the if-statement at the beginning of the function byteArrayToUint32(). The checks for "314922", "1540" and "38" only use byte values of less than 128 (0x80). This leaves the check for "9020562f", which starts with 0x90 - the negative integer -112. The two's complement of -112 is 0xFFFFFF90. The line

```cpp
        number += static_cast<quint32>(b << (8 * i));
```

shifts 0xFFFFFF90 24 bits to the left, which yields 0x90000000. The shifting lets all the leading F's disappear but only for the byte taking up the bit positions 24 to 31. For all other bytes, shifting can't do this magic trick. The other bytes are not shifted far enough.

The unit test didn't catch the problem, because I accidentally chose a skewed distribution of the byte values. I added two more checks to the unit test.

```cpp
void TestByteArrayToUint32::testByteArrayToUint32()
{
    // ...
    QCOMPARE(byteArrayToUint32(QByteArray::fromHex("0180")), 0x180U);
    QCOMPARE(byteArrayToUint32(QByteArray::fromHex("95")), 0x95U);
}
```

You can find the example code on [github](https://github.com/bstubert/embeddeduse/tree/master/BlogPosts/ByteArrayToUint32).



1. Tim
   2018/03/31

   Interesting. To my mind this is really the fault of QByteArray::operator[] returning a char instead of an unsigned char; a byte is just 8 bits (I know, I know, it wasn’t always so…) and should be returned as such from a container named “QByteArray”. Returning a char is actually worse than returning a straight “signed char” as it’s ambiguous if the value is signed or not. Urgh.

   If I were to implement the above, I would be inclined to use .data() and cast the result to a const uint8_t* and then use that. No nasty surprises…

   

   Burkhard Stubert
   2018/03/31

   Your idea of using .data() and casting only works if the byte array comes in little endian – as you can see from the first two checks in the unit test below. That’s the reason why I don’t like network protocols using big endian notation. Here is the conversion function with its test.

   ```cpp
       quint32 castToUint32(const QByteArray <bytes) const
       {
           auto number = reinterpret_cast<const quint32 *>(bytes.data());
           return *number;
       }
   
   void TestByteArrayToUint32::testCastToUint32()
   {
       QCOMPARE(castToUint32(QByteArray::fromHex("8001")), 0x180U);
       QCOMPARE(castToUint32(QByteArray::fromHex("0180")), 0x8001U);
       QCOMPARE(castToUint32(QByteArray::fromHex("224931")), 0x314922U);
   }
   ```

2. 

   Burkhard Stubert
   2018/03/31

   You perfectly summarised the problem with QByteArray. We expect the underlying type to be a uint8_t (quint8). If it were a char, the class should be called QCharArray.