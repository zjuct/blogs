---
title: 'JVM字节码格式: 其一'
date: 2024-04-20T15:30:36+08:00
draft: false
tags:
    - JVM
categories:
    - 工程笔记
---

`.class`文件格式是8-bit bytes构成的流，对于多字节数据采用**大端**表示。Java源文件中的每个`class`(包括内部类)，`interface`和`module`都对应一个`.class`字节码文件。需要注意的是，并不是JVM加载的每个类都与磁盘上的一个`.class`文件对应，`.class`规范其实是对字节流的规范，不管是物理磁盘上的文件，还是网络流，或者是内存中的数据，只要满足`.class`规范，都能被JVM作为一个类加载，这也是CGLIB等库能在运行时动态构造新的类的原因。

整个`.class`文件用C风格的结构体表示如下

```C
struct ClassFile {
    u4 magic;               // 魔数
    u2 minor_version;       // .class规范的版本号
    u2 major_version;
    u2 constant_pool_count; // 常量池
    cp_info constant_pool[constant_pool_count-1];
    u2 access_flags;        // 类的访问控制
    u2 this_class;          // 当前类的类名的常量池索引
    u2 super_class;         // 父类类名的常量池索引
    u2 interfaces_count;    // 实现的接口
    u2 interfaces[interfaces_count];
    u2 fields_count;        // 字段
    field_info fields[fields_count];
    u2 methods_count;       // 方法
    method_info methods[methods_count];
    u2 attributes_count;    // 属性
    attribute_info attributes[attributes_count];
};
```

## magic

魔数用来标识一个二进制流是`.class`文件，所有的`.class`文件的头四个字节都应该是固定的`0xCAFEBABE`。即第一个字节是`0xCA`，第四个字节是`0xBE`。魔数在很多文件格式中都有使用，例如ELF和JPEG等。


## 版本号

这个版本号指的是`.class`规范的版本号，随着JDK新版本的发布，`.class`文件规范也在不断更新（虽然改动很小）。每个JDK版本都有对应的JVM版本，标准要求`.class`文件是向前兼容的，即新版本的JVM必须能兼容旧版本的`.class`。因此，用JDK8编译出的`.class`文件能够在JDK17自带的JVM中运行，而JDK17编译出的`.class`文件不能在低版本的JVM中运行（禁止低版本JVM运行高版本`.class`是标准的硬性规范，即使`.class`的各个版本没有什么改动）。

JDK版本对应的版本号以及JVM支持的版本号范围可以在手册中查到，这里不再复制。只要注意`.class`文件的前向兼容和后向不兼容即可。


## 常量池

`.class`文件的常量池范围很大，不仅仅是编程语言意义上的“常量”，例如字符串、整型、浮点数的字面值。还包含类名、方法名、字段名、方法参数列表和返回值、字段类型等。规范地说，常量池表项主要存放编译期生成的各种字面量和符号引用，这部分内容将在类加载时被存放到JVM方法区的运行时常量池中。

需要注意的是，常量池表的第0项是空项（这个跟NULL指针以及x86的GDT类似，常量池通常通过索引来间接引用，当常量池索引用于表达“不引用任何一个常量池项时”，可以将索引值设置为0），因此`constant_pool_count`实际上是常量池项数+1，即常量池中有效的项的下标是`[1, constant_pool_count - 1]`。常量池的长度由一个`u2`字段，因此，常量池的最大长度为65534。

> 常量池的第0项只是一个概念，并没有在`.class`文件中专门分配空间用于存放这个第0项

一个常量池项`cp_info`的结构如下

```C
struct cp_info {
    u1 tag;         // 类型
    u1 info[];      // 实际内容
};
```

常量池项的类型由第一个字节决定，在JDK17中，一共有17种类型，不同类型的常量池项几乎是完全独立的，下面对其逐个进行介绍。

### 字面值

这里的字面值范围比较狭隘，不是一般意义上的源文件中的字面值。字面值指字符串字面值，或声明为final的常量值（注意非final的字面值不会被放入常量池中）

对于int, float, long, double字面值，都有对应的常量表表项，结构比较简单，就是tag加上对应的数值的大端表示。

> byte, short, char, boolean的字面值在编译后都会以Integer字面值的形式进入符号表

```C
struct CONSTANT_Integer_info {
    u1 tag;
    u4 bytes;
};
```

比如下面这个例子，5会进入常量池，但6不会

```
class A {
    final int i = 5;
    int j = 6;
}
```

字符串字面值对应常量池中的`CONSTANT_String_info`项。字符串字面值本身并不存储在`CONSTANT_String_info`中，而是存储在`CONSTANT_Utf8_info`中。通过`string_index`指向对应的`CONSTANT_Utf8_info`表项。

```
struct CONSTANT_String_info {
    u1 tag;
    u2 string_index;
};
```

> 在`.class`文件中，字符串字面值采用改良的UTF-8编码，但是Java String对象采用Unicode(UTF-16)编码。因此在`.class`文件的加载过程中，需要进行编码转换

### CONSTANT_Utf8_info

在常量池中，所有的字符串本体存放在`CONSTANT_Utf8_info`中，其采用改良的UTF-8编码以节省空间。

```C
struct CONSTANT_Utf8_info {
    u1 tag;
    u2 length;
    u1 bytes[length];
};
```

### 符号引用

Java采用动态链接的方式，即JVM在加载.class文件时进行符号链接的解析。

- 符号引用：以一组符号来描述所引用的目标，符号可以是任意形式的字符串字面值，在类加载的过程中，JVM会根据符号引用加载并解析对应的符号，将其转化为直接引用。
- 直接引用：直接引用可以是直接指向目标的指针、相对偏移量或者一个能间接定位到目标的句柄。

`CONSTANT_Class_info`是对类的符号引用。`name_index`是常量池索引，指向`CONSTANT_Utf8_info`项，其中包含这个类的全限定名。

```C
struct CONSTANT_Class_info {
    u1 tag;
    u2 name_index;
};
```

全限定名即将全类名`com.example.Person`中的`.`替换为`/`，并在最后加上`;`用于分隔，即`com/example/Person;`

`CONSTANT_Fieldref_info`是对字段的符号引用。`class_index`指向`CONSTANT_Class_info`项，表示这个字段定义在哪个类中(可以是class，也可以是interface)，`name_and_type_index`指向`CONSTANT_NameAndType_info`项，其中包含字段名和描述符(即字段的类型)。

```C
struct CONSTANT_Fieldref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
};
```

`CONSTANT_Methodref_info`是对方法的符号引用。`class_index`指向的表项只能是class，不能是interface。方法的描述符包含方法的参数和返回值类型。

```C
struct CONSTANT_Methodref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
};
```

`CONSTANT_InterfaceMethodref_info`与Methodref的唯一区别是，`class_index`指向的表项只能是interface，不能是class。

### CONSTANT_NameAndType_info

包含字段/方法的名字和描述符，`name_index`和`descriptor_index`都指向`CONSTANT_Utf8_info`。

```C
struct CONSTANT_NameAndType_info {
    u1 tag;
    u2 name_index;
    u2 descriptor_index;
};
```

- `name_index`指向的字符串包含字段和方法的非限定名(unqualified name)，即不包括字段类型、方法参数和返回值。例如对于方法`void func(int a)`，其非限定名就是`func`。
- `descriptor_index`指向描述符字符串。所有基本类型、对象类型、数组类型和void按照如下规则进行表示
  - byte: B
  - char: C
  - double: D
  - float: F
  - int: I
  - long: L
  - short: S
  - boolean: Z
  - 对象类型: 采用`L类的全限定名`表示，例如String即为`Ljava.lang.String;`
  - 数组：数组的每一维采用一个`[`表示，例如String[][]表示为`[[Ljava.lang.String;`
  
字段描述符即由上述规则描述其类型，方法描述符为`(参数1参数2...)返回值`

例如

```
class Foo {
    int bar;
    void hello(int a, int b, String[] c);
}
```

`bar`对应的字段描述符为`I`；`hello`对应的方法描述符为`(II[Ljava.lang.String)V`。