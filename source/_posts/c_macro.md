---
title: C++中的宏定义
---

## \# ##
----
宏定义中，#的功能是将其后面的宏参数进行字符串化操作，可以理解成去掉#号并在该其左右加上双引号。
```c++
#define WARN_IF(EXP) do {if (EXP) fprintf(stderr, "Warning:" #EXP "\n");} while(0)
...
WARN_IF(divider == 0);
```
上面的宏会被替换成
```c++
do { if (divider == 0) fprintf(stderr, "Warning:" "divider == 0" "\n"); } while(0);
```
这种特性带来的一个非常方便的用处就是使用宏LOG时可以输出更加丰富的信息

## \#\# ##
----
\#\#是连接符，用来将多个Token连接成一个Token。
```c++
struct Command {
  char* name;
  void (*function)(void);
};

#define COMMAND(NAME) {NAME, NAME ## _command}
...
Command command[] = { COMMAND(quit), COMMAND(help), ...};
```

## ...
----
...是变参宏（Variadic Macro）。比如
```c++
#define myprintf(templt, ...) fprintf(stderr, templt, __VA_ARGS__)
```
或者
```c++
#define myprintf(tmplt, args...) fprintf(stderr, templt, args)
```
区别在于如果没有提供变参名，则必须使用__VA__ARGS__来代替它。同时要注意的是，如果变参的个数为0，最后一个逗号不能省略。
```c++
myprintf("Error",);
```
