The C++ standard library does not provide a proper date type. C++ inherits the structs and functions for date and time manipulation from C. To access date and time related functions and structures, you would need to include <ctime> header file in your C++ program.

There are four time-related types: **clock_t, time_t, size_t**, and **tm**. The types - clock_t, size_t and time_t are capable of representing the system time and date as some sort of integer.

The structure type **tm** holds the date and time in the form of a C structure having the following elements −

```
struct tm {
   int tm_sec;   // seconds of minutes from 0 to 61
   int tm_min;   // minutes of hour from 0 to 59
   int tm_hour;  // hours of day from 0 to 24
   int tm_mday;  // day of month from 1 to 31
   int tm_mon;   // month of year from 0 to 11
   int tm_year;  // year since 1900
   int tm_wday;  // days since sunday
   int tm_yday;  // days since January 1st
   int tm_isdst; // hours of daylight savings time
}
```

Following are the important functions, which we use while working with date and time in C or C++. All these functions are part of standard C and C++ library and you can check their detail using reference to C++ standard library given below.

| Sr.No |                      Function & Purpose                      |
| ----- | :----------------------------------------------------------: |
| 1     | **time_t time(time_t \*time);**This returns the current calendar time of the system in number of seconds elapsed since January 1, 1970. If the system has no time, .1 is returned. |
| 2     | **char \*ctime(const time_t \*time);**This returns a pointer to a string of the form *day month year hours:minutes:seconds year\n\0*. |
| 3     | **struct tm \*localtime(const time_t \*time);**This returns a pointer to the **tm** structure representing local time. |
| 4     | **clock_t clock(void);**This returns a value that approximates the amount of time the calling program has been running. A value of .1 is returned if the time is not available. |
| 5     | **char \* asctime ( const struct tm \* time );**This returns a pointer to a string that contains the information stored in the structure pointed to by time converted into the form: day month date hours:minutes:seconds year\n\0 |
| 6     | **struct tm \*gmtime(const time_t \*time);**This returns a pointer to the time in the form of a tm structure. The time is represented in Coordinated Universal Time (UTC), which is essentially Greenwich Mean Time (GMT). |
| 7     | **time_t mktime(struct tm \*time);**This returns the calendar-time equivalent of the time found in the structure pointed to by time. |
| 8     | **double difftime ( time_t time2, time_t time1 );**This function calculates the difference in seconds between time1 and time2. |
| 9     | **size_t strftime();**This function can be used to format date and time in a specific format. |

## Current Date and Time

Suppose you want to retrieve the current system date and time, either as a local time or as a Coordinated Universal Time (UTC). Following is the example to achieve the same −

[Live Demo](http://tpcg.io/DHKMA9)

```
#include <iostream>
#include <ctime>

using namespace std;

int main() {
   // current date/time based on current system
   time_t now = time(0);
   
   // convert now to string form
   char* dt = ctime(&now);

   cout << "The local date and time is: " << dt << endl;

   // convert now to tm struct for UTC
   tm *gmtm = gmtime(&now);
   dt = asctime(gmtm);
   cout << "The UTC date and time is:"<< dt << endl;
}
```

When the above code is compiled and executed, it produces the following result −

```
The local date and time is: Sat Jan  8 20:07:41 2011

The UTC date and time is:Sun Jan  9 03:07:41 2011
```

## Format Time using struct tm

The **tm** structure is very important while working with date and time in either C or C++. This structure holds the date and time in the form of a C structure as mentioned above. Most of the time related functions makes use of tm structure. Following is an example which makes use of various date and time related functions and tm structure −

While using structure in this chapter, I'm making an assumption that you have basic understanding on C structure and how to access structure members using arrow -> operator.

[Live Demo](http://tpcg.io/SMnO0N)

```
#include <iostream>
#include <ctime>

using namespace std;

int main() {
   // current date/time based on current system
   time_t now = time(0);

   cout << "Number of sec since January 1,1970:" << now << endl;

   tm *ltm = localtime(&now);

   // print various components of tm structure.
   cout << "Year:" << 1900 + ltm->tm_year << endl;
   cout << "Month: "<< 1 + ltm->tm_mon<< endl;
   cout << "Day: "<<  ltm->tm_mday << endl;
   cout << "Time: "<< 1 + ltm->tm_hour << ":";
   cout << 1 + ltm->tm_min << ":";
   cout << 1 + ltm->tm_sec << endl;
}
```

When the above code is compiled and executed, it produces the following result −

```
Number of sec since January 1,1970:1563027637
Year2019
Month: 7
Day: 13
Time: 15:21:38
```

You can try the following cross-platform code to get current date/time:

```cpp
#include <iostream>
#include <string>
#include <stdio.h>
#include <time.h>

// Get current date/time, format is YYYY-MM-DD.HH:mm:ss
const std::string currentDateTime() {
    time_t     now = time(0);
    struct tm  tstruct;
    char       buf[80];
    tstruct = *localtime(&now);
    // Visit http://en.cppreference.com/w/cpp/chrono/c/strftime
    // for more information about date/time format
    strftime(buf, sizeof(buf), "%Y-%m-%d.%X", &tstruct);

    return buf;
}

int main() {
    std::cout << "currentDateTime()=" << currentDateTime() << std::endl;
    getchar();  // wait for keyboard input
}
```

Output:

```cpp
currentDateTime()=2012-05-06.21:47:59
```

Please visit [here](http://en.cppreference.com/w/cpp/chrono/c/strftime) for more information about date/time format