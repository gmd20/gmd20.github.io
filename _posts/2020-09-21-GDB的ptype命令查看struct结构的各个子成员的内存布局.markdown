比较新的gdb的ptype命令支持/o选项，打印结构的内存布局了，类似pahole命令吧
```text
(gdb) help ptype
Print definition of type TYPE.
Usage: ptype[/FLAGS] TYPE | EXPRESSION
Argument may be any type (for example a type name defined by typedef,
or "struct STRUCT-TAG" or "class CLASS-NAME" or "union UNION-TAG"
or "enum ENUM-TAG") or an expression.
The selected stack frame's lexical context is used to look up the name.
Contrary to "whatis", "ptype" always unrolls any typedefs.

Available FLAGS are:
  /r    print in "raw" form; do not substitute typedefs
  /m    do not print methods defined in a class
  /M    print methods defined in a class
  /t    do not print typedefs defined in a class
  /T    print typedefs defined in a class
  /o    print offsets and sizes of fields in a struct (like pahole)
```

cenos8携带的gdb版本支持了，不过centos7还是不支持的

