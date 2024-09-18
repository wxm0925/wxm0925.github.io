---
title: Spring循环依赖分析
category: JVM
tag: class结构
date: 2024/9/11 20:46:25
---

# Class文件分析

```java
package jvm.classstruct;

/**
 * @author wenxiangmin
 * @ClassName TestClass.java
 * @Description TODO
 * @createTime 2024年08月31日 13:51:00
 */
public class TestClass {

    private int m;

    public int inc() {
        return m + 1;
    }
    
}

```

使用winhex打开class文件的十六进制表示如下图所示

![image-20240904224941918](C:/Users/wen_xm/AppData/Roaming/Typora/typora-user-images/image-20240904224941918.png)

## 前10个字节

`CAFEBABE000000340016` 是一个 `.class` 文件的前 10 字节的十六进制表示，表示的是类文件的头部信息。我们可以分段来分析这些字节的含义：

### 1. **前 4 字节（CAFEBABE）**：魔数（Magic Number）

- **十六进制**：`CAFEBABE`
- **描述**：`CAFEBABE` 是 Java 类文件的 **魔数**。魔数是每个 `.class` 文件开头的标志，用于标识该文件是一个有效的 Java 类文件。JVM 通过检查文件的前 4 个字节是否是 `CAFEBABE` 来判断该文件是否为一个有效的 `.class` 文件。
- **原因**：这个魔数是硬编码的，目的是为了确保 JVM 能够快速识别和验证一个 `.class` 文件的格式正确性。

### 2. **接下来的 4 字节（00000034）**：版本号

- **十六进制**：`00000034`

- 描述

  ：

  ```
  00000034
  ```

   分为两个部分：

  - **次版本号（Minor Version）**：`0000`（十六进制），表示次版本号，转换为十进制是 `0`。
  - **主版本号（Major Version）**：`0034`（十六进制），表示主版本号，转换为十进制是 `52`。

- 解释

  ：

  - **版本号**：`00000034` 中的 `0000` 是次版本号，`0034` 是主版本号。主版本号 `52` 对应于 **Java 8** 的 `.class` 文件格式。

  - Java 版本和主版本号对应关系

    ：

    - Java 1.1 -> 45
    - Java 1.2 -> 46
    - Java 1.3 -> 47
    - Java 1.4 -> 48
    - Java 5 -> 49
    - Java 6 -> 50
    - Java 7 -> 51
    - **Java 8 -> 52**
    - Java 9 -> 53
    - Java 10 -> 54
    - Java 11 -> 55



###  3.后2个字节

- **十六进制值**：`0016`

- 解释：

  ```
  0016
  ```

   是一个 2 字节（u2）的值，表示常量池中的条目数量，单位是 16 进制。

  - 将 `0016` 转换为十进制：`0016` (16 进制) = `22` (十进制)。
  - 这意味着 **常量池中有 22 个条目**。

 **常量池（Constant Pool）**

常量池是 `.class` 文件的一部分，它存储了类中用到的所有常量、方法、字段名、类型信息等。常量池是一个数组，数组的第一个条目（即索引 0）是保留的，因此常量池的大小实际上是 `constant_pool_count - 1`，也就是 `22 - 1 = 21` 个有效条目。

在 `.class` 文件中，常量池的作用包括：

- 存储类、方法、字段的符号引用。
- 存储字面值常量（如字符串、整数、浮点数等）。
- 存储方法和字段的描述符。

 **常量池计数器的作用**

`0016` 表示常量池的计数器，它告诉 JVM 在接下来的部分中将会有 22 个常量池条目（从 1 到 21）。这些条目包含了类文件中的所有符号引用、字面量等信息，供 JVM 在加载类和执行代码时使用。



## 常量池

到目前为止，我们知道了有21个有效的常量item，每一个常量item都有一个通用格式

```c
cp_info {
    u1 tag;  #标识符，表示每一个常量属于哪个常量类型
    u1 info[]; #表的数组，一个或者多个表，具体有多少个，需要根据tag来确定
}
```

在上面的16进制图中，常量池数量后面跟着的是`0A`，十进制表示10，也就是tag=10，tag=10常量名称为`CONSTANT_Methodref  `，它的格式如下

```c
CONSTANT_Methodref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}
```

在java8中，常量类型对应的tag如下表所示

| Constant Type               | Tag  |
| :-------------------------- | ---- |
| CONSTANT_Class              | 7    |
| CONSTANT_Fieldref           | 9    |
| CONSTANT_Methodref          | 10   |
| CONSTANT_InterfaceMethodref | 11   |
| CONSTANT_String             | 8    |
| CONSTANT_Integer            | 3    |
| CONSTANT_Float              | 4    |
| CONSTANT_Long               | 5    |
| CONSTANT_Double             | 6    |
| CONSTANT_NameAndType        | 12   |
| CONSTANT_Utf8               | 1    |
| CONSTANT_MethodHandle       | 15   |
| CONSTANT_MethodType         | 16   |
| CONSTANT_InvokeDynamic      | 19   |

...



