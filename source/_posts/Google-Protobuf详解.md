---
title: Google Protobuf详解
date: 2023-01-14 21:23:34
categories: 工具
tags: [Java, protobuf]
---

# 1 简介

`Protobuf`是由`Google`设计的一种高效、轻量级的信息描述格式, 起初是在`Google`内部使用，后面开源，它具有语言中立、平台中立、高效、可扩展等特性，它非常适合用来做数据存储、`RPC`数据交换等。与`json`、`xml`相比，` Protobuf`的编码长度更短、传输效率更高，其实严格意义上讲，`json`、`xml`、并非是一种「**编码**」，而只能称之为「**格式**」，`json`、`xml` 的内容本身都是字符形式，它们的编码采用的是`ASCII`编码。

| -            | xml          | json     | protobuf     |
| ------------ | ------------ | -------- | ------------ |
| 数据结构     | 结构一般复杂 | 结构简单 | 结构比较复杂 |
| 数据存储方式 | 文本         | 文本     | 二进制       |
| 数据存储大小 | 大           | 一般     | 小           |
| 解析效率     | 慢           | 一般     | 快           |
| 跨语言支持   | 非常多       | 多       | 一般         |
| 开发成本     | 比较繁琐     | 非常简单 | 一般         |
| 学习成本     | 一般         | 低       | 一般         |

一旦定义了要处理的数据的数据结构之后，就可以利用`Protobuf`的代码生成工具生成相关的代码。只需使用`Protobuf`对数据结构进行一次描述，即可利用各种不同语言(`proto3`支持`C++、Java、Python、Go、Ruby、Objective-C、C#`)或从各种不同流中对你的结构化数据轻松读写。

本文讲述`Protobuf`的底层编码原理, 以便于了解`Protobuf`为什么编码长度短并且扩展性强, 与此同时我们也将了解到它有哪些不足？

<!-- more -->
<!-- markdownlint-disable MD041 MD002--> 

# 2 用法

## 2.1 关于版本

`Protobuf`有两个大版本，`proto2`和`proto3`，同比于`python2.x`和`python3.x`版本。初学者建议直接学习`proto3`版本。

`proto3`相对于`proto2`而言，简而言之是支持了更多的语言（`Ruby、C#`等）、删除了一些复杂的语法和特性、引入了更多的约定等。

> 与json开箱即用不一样的是，protobuf需要依赖于工具包编译成`java`文件或者`go`文件等，所以需要关注protobuf的版本

## 2.2 使用

### 2.2.1 安装

使用`proto`之前需要先安装编译器，[官方下载地址](https://developers.google.com/protocol-buffers/docs/downloads)，具体安装步骤可自行搜索。可在控制台使用以下命令检查是否安装成功：

> mark.hct@ ~ % protoc --version
> libprotoc 3.21.12

>  也可以通过在ideal安装**Protobuf Support**，之后通过maven来编译proto文件

### 2.2.2 proto文件

在`proto`文件中，需要定义程序中需要处理的结构化数据。其中结构化数据被称为`Message`。`proto`文件非常类似于java中的bean。

> `proto`文件对应序列化理论中的`IDL(Interface description language)`接口描述语言

定义一个Test.proto文件

```protobuf
syntax = "proto3"; // PB协议版本

import "google/protobuf/any.proto"; // 引用外部的message，可以是本地的，也可以是此处比较特殊的 Any

package test; // 包名，其他 proto 在引用此 proto 的时候，就可以使用 test.protobuf.PersonTest 来使用，
// 注意：和下面的 java_package 是两种易混淆概念，同时定义的时候，java_package 具有较高的优先级

option java_package = "test"; // 生成类的包名，注意：会在指定路径下按照该包名的定义来生成文件夹
option java_outer_classname = "PersonTestProtos"; // 生成类的类名，注意：下划线的命名会在编译的时候被自动改为驼峰命名

message PersonTest {
  int32 id = 1; // int 类型
  string name = 2; // string 类型
  string email = 3;
  Sex sex = 4; // 枚举类型
  repeated PhoneNumber phone = 5; // 引用下面定义的 PhoneNumber 类型的 message
  map<string, string> tags = 6; // map 类型
  repeated google.protobuf.Any details = 7; // 使用 google 的 any 类型

  // 定义一个枚举
  enum Sex {
    DEFAULT = 0;
    MALE = 1;
    Female = 2;
  }

  // 定义一个 message
  message PhoneNumber {
    string number = 1;
    PhoneType type = 2;

    enum PhoneType {
      MOBILE = 0;
      HOME = 1;
      WORK = 2;
    }
  }
}
```

#### message语法说明（部分）

1、 `proto3`中，枚举的第一个常量名的编号必须为`0`；

> 由于proto3的默认值规则进行了调整，枚举的默认值为第一个，所以必须将第一个常量的标号设置为0，但是这和业务有时是冲突的，此时，将第一个常量设置为`xx_UNSPECIFIED=0`，如：`ENUM_TYPE_UNSPECIFIED = 0`

2、同一个proto文件中，多个枚举之间不允许定义相同的常量名

```protobuf
enum IDE1 {
    IDEA = 0;
    ECLIPSE = 1;
}

enum IDE2 {
    IDEA = 7;
    ECLIPSE = 8;
}
```

​		此时会报错，`IDEA is already defined in "xxx"`

3、 Any理解

`google.protobuf.Any`可以理解为`java`中的`object`，但又和`object`不同。`Any`不是所有的`Message`的父类，而`object`是所有类的父类。如以下示例代码：

`Java Api`的代码为：

```java
@Data
public class ApiResult {
    private int code;
    private String error;
    private Object data;
}
```

对应的`proto`定义为:

```protobuf
message ApiResult {
    int32 code = 1;
    string error = 2;
    google.protobuf.Any data = 3;
}
```

`protobuf`提供了更多选项和数据类型，本文不做详细介绍，感兴趣可以参考[这里](https://developers.google.com/protocol-buffers/docs/proto3)

### 2.2.3 编译

从控制台进入`proto`文件所在路径，通过`protoc`进行编译得到对应的`java`文件拷贝到项目中使用。

```shel
protoc -I=$path --java_out=$path $path/$file
```

> 参数说明：
>
> - -I / -proto_path：指定.proto文件所在的路径
> - --java_out：编译成java文件时，标明输出目标路径
> - $path/$file：指定需要编译的.proto文件

### 2.2.4 项目使用

- 对于`protoc`编译生成的`java`代码实现序列化和反序列化，需要在工程中添加`protobuf-java`的依赖

  ```prot
          <dependency>
              <groupId>com.google.protobuf</groupId>
              <artifactId>protobuf-java</artifactId>
              <version>3.21.12</version>
          </dependency>
  ```

- 可以使用多种方式进行序列化和反序列化

  ```java
  import com.google.protobuf.ByteString;
  import com.google.protobuf.InvalidProtocolBufferException;
  import test.PersonTestProtos.PersonTest.PhoneNumber.PhoneType;
  import test.PersonTestProtos.PersonTest.Sex;
  
  import java.io.ByteArrayInputStream;
  import java.io.ByteArrayOutputStream;
  import java.io.IOException;
  
  /**
   * @author mark.hct
   * @version ProtoTest.java v 0.1 2023/1/15 21:53 Exp $
   * @description
   */
  public class ProtoTest {
      public static void main(String[] args) {
          try {
              /** Step1：生成 personTest 对象 */
              PersonTestProtos.PersonTest.Builder personBuilder = PersonTestProtos.PersonTest.newBuilder();
              // personTest 赋值
              personBuilder.setName("cxk");
              personBuilder.setEmail("cxk@gmail.com");
              personBuilder.setSex(Sex.MALE);
  
              // 内部的 PhoneNumber 构造器
              PersonTestProtos.PersonTest.PhoneNumber.Builder phoneNumberBuilder = PersonTestProtos.PersonTest.PhoneNumber.newBuilder();
              // PhoneNumber 赋值
              phoneNumberBuilder.setType(PhoneType.MOBILE);
              phoneNumberBuilder.setNumber("138xxxx");
  
              // personTest 设置 PhoneNumber
              personBuilder.addPhone(phoneNumberBuilder);
  
              // 生成 personTest 对象
              PersonTestProtos.PersonTest personTest = personBuilder.build();
  
              /** Step2：序列化和反序列化 */
              // 方式一 byte[]：
              // 序列化
              byte[] bytes = personTest.toByteArray();
              PersonTestProtos.PersonTest personTestResult1 = PersonTestProtos.PersonTest.parseFrom(bytes);
              System.out.printf("反序列化得到的信息，姓名：%s，性别：%d，手机号：%s%n", personTestResult1.getName(), personTest.getSexValue(),
                      personTest.getPhone(0).getNumber());
  
              // 方式二 ByteString：
              // 序列化
              ByteString byteString = personTest.toByteString();
              System.out.println(byteString.toString());
              // 反序列化
              PersonTestProtos.PersonTest personTestResult2 = PersonTestProtos.PersonTest.parseFrom(byteString);
              System.out.printf("反序列化得到的信息，姓名：%s，性别：%d，手机号：%s%n", personTestResult2.getName(), personTest.getSexValue(),
                      personTest.getPhone(0).getNumber());
  
              // 方式三 InputStream
              // 粘包,将一个或者多个protobuf 对象字节写入 stream
              // 序列化
              ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
              personTest.writeDelimitedTo(byteArrayOutputStream);
              // 反序列化，从 steam 中读取一个或者多个 protobuf 字节对象
              ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(byteArrayOutputStream.toByteArray());
              PersonTestProtos.PersonTest personTestResult3 = PersonTestProtos.PersonTest.parseDelimitedFrom(byteArrayInputStream);
              System.out.printf("反序列化得到的信息，姓名：%s，性别：%d，手机号：%s%n", personTestResult3.getName(), personTest.getSexValue(),
                      personTest.getPhone(0).getNumber());
  
          } catch (IOException e) {
              throw new RuntimeException(e);
          }
      }
  }
  
  ```

# 3 原理

## 3.1 protobuf数据结构

1、采用`TLV`存储方式，即`Tag-Length-Value`（标识-长度-字段值）；
2、不需要分隔符就能分开字段，较少了分隔符的使用；
3、各字段存储得非常紧凑，存储空间利用率非常高；
4、若字段没有被设置字段值，那么该字段在序列化时的数据是完全不存在的，即不需要要编码；

![image-20230115235320120](Google-Protobuf%E8%AF%A6%E8%A7%A3/image-20230115235320120.png)

> - Tag：字段标识号，用户表示字段；
> - Length：Value的字段长度；
> - Value：消息字段经过编码后的值；

## 3.2 protobuf数据组织

首先来看一个例子，假设客户端和服务端使用`protobuf`作为数据交换格式，`proto`的具体定义为：

```protobuf
syntax = "proto3";
package pbTest;

message Request {
    int32 age = 1;
}
```

`Request`中包含了一个名称为`age`的字段，客户端和服务端双方都用同一份相同的`proto`文件是没有任何问题的，假设客户端自己将`proto`文件做了修改，修改后的`proto`文件如下：

```protobuf
syntax = "proto3";
package pbTest;

message Request {
    int32 age_test = 1;
}
```

在这种情形下, 服务端不修改应用程序仍能够正确地解码，原因在于序列化后的`Protobuf`没有使用字段名称，而仅仅采用了字段编号，与`json xml`等相比，`protobuf`不是一种完全自描述的协议格式, 即接收端在没有`proto`文件定义的前提下是无法解码一个`protobuf`消息体。与此相对的，`json xml`等协议格式是完全自描述的，拿到了`json`消息体，便可以知道这段消息体中有哪些字段，每个字段的值分别是什么，其实对于客户端和服务端通信双方来说, 约定好了消息格式之后完全没有必要在每一条消息中都携带字段名称，`protobuf`在通信数据中移除字段名称, 这可以大大降低消息的长度, 提高通信效率。对于不同数据类型采用不同的序列化方式（编码方式&数据存储方式）如下表：

| wire_type | 编码方式                        | 编码长度         | 存储方式  | 代表的数据类型                                               |
| --------- | ------------------------------- | ---------------- | --------- | ------------------------------------------------------------ |
| 0         | Vaint（负数时以Zigzag辅助编码） | 变长(1-10个字节) | T - V     | int32, int64, unit32, unit64, bool, enum, sint32, sint64(负数时使用) |
| 1         | 64-bit                          | 固定8个字节      | T - V     | fixed64, sfixed64, double                                    |
| 2         | Length-delimi                   | 变长             | T - L - V | string, bytes, embedded messages, packed repeated fields     |
| 3         | Start group                     | 已弃用           | 已弃用    | Groups（已弃用）                                             |
| 4         | End group                       | 已弃用           | 已弃用    | Groups（已弃用）                                             |
| 5         | 32-bit                          | 固定4个字节      | T - V     | Fixed32, sfixed32, float                                     |

对于`int32, int64, uint32`等数据类型在序列化之后都会转为`Varint`编码，除去两种已标记为`已废弃(deprecated)` 的类型，目前`Protobuf`在序列化之后的消息类型`(wire_type)`总共有 4 种，`Protobuf `除了存储字段的值之外，还存储了字段的编号以及字段在通信线路上的格式类型`(wire-type)`， 具体的存储方式为:

> **field_number << 3 | wire_type**

即将字段标号逻辑左移 3 位，然后与该字段的`wire type`的编号按位或，在上表中可以看到，`wire type `总共有 6 种类型，因此可以用 3 位二进制来标识，所以低 3 位实际上存储了其后所跟的数据的` wire type`，接收端可以利用这些信息，结合` proto `文件来解码消息结构体。
以上面`proto` 为例来看一段` Protobuf `实际序列化之后的完整二进制数据，假设` age `为 5，由于` age `在` proto `文件中定义的是 int32 类型，因此序列化之后它的` wire_type `为 0，其字段编号为 1，因此按照上面的计算方式，即` 1 << 3 | 0`，所以其类型和字段编号的信息只占 1 个字节，即` 00001000`，后面跟上字段值 5 的` Varint` 编码，所以整个结构体序列化之后为`（T - V格式）`：

<img src="Google-Protobuf%E8%AF%A6%E8%A7%A3/image-20230116002742031.png" alt="image-20230116002742031" style="zoom: 50%;" />

有了字段编号和` wire type`，其后所跟的数据的长度便是确定的，因此` Protobuf `是一种非常紧密的数据组织格式，其不需要特别地加入额外的分隔符来分割一个消息字段，这可大大提升通信的效率，规避冗余的数据传输。

## 3.3 Varint编码

普通的`int`数据类型，无论值的大小，所占用的存储空间都是相等的，从这点这出发考虑根据数值大小来动态地占用存储空间，使得值比较小的数字占用较少的字节数，值相对比较大的数字占用比较多的字节数，这就是**变长整型编码**的基本思想，采用变长整型编码的数字，其占用的字节数不是完全一致的，为了达到这个目的，`Varint`编码使用每个字节的最高有效位作为标志位，而剩余的7位以二进制补码的形式来存储数字值本身：

- 当最高位有效位为1时，代表后面还跟有字节；
- 当最高位有效位为0时，代表该数字式最后的一个字节；

在`protobuf`中，使用的`Base128 Varint`编码，之所以叫这个名字原因及时在这种方式中，使用`7 bit`来存储数字，`Base128 Varint`采用的是小端序，即**数字的低位存放在高位地址**。

### 编码案例1

![image-20230116234912915](Google-Protobuf%E8%AF%A6%E8%A7%A3/image-20230116234912915.png)

### 编码案例2

![image-20230214140405012](Google-Protobuf%E8%AF%A6%E8%A7%A3/image-20230214140405012.png)

### 解码案例

<img src="Google-Protobuf%E8%AF%A6%E8%A7%A3/image-20230117002739059.png" alt="image-20230117002739059" style="zoom: 67%;" />

## 3.4 Zigzag编码

`Varint`编码的实质在于去掉数字开头的`0`，因此可以缩短数字所占的存储字节数，在上一章节中，说明了整数的`Varint`编码，但是如果数字为负数时，使用`Varint`编码会占用恒定的`10个`字节，原因在于负数的符号位`1`。对于负数，其从符号位开始的高位均为`1`，在`protobuf`的具体实现中，会将此视为一个很大的无符号数。

究其原因在于`protobuf`的内部将`int32`类型的负数转换为`uint64`来处理，转换后的`unit64`数值的高位全是`1`，相当于是一个8字节的很大的无符号数，因此采用`Base128 Varint`编码后将恒定占用10个字节的空间，可见`Varint`编码对于负数时毫无优势，甚至比普通的固定`32`为存储还要多占`4`个字。`Varint`编码的实质在于设法移除数字开头为`0`的比特位，由于负数的高位都为1，因此`Varint`编码在此场景下都会失效，`Zigzag`编码便是为了解决这个问题，其大致思想是：**首先对负数做一次变换，将其映射成一个正数，变换后便可以使用Varint编码进行压缩**。这里关键的一点在于变换算法，其算法必须是可逆的，既可以根据变换后的值计算出原始值，否则无法解码，同时要求变换算法尽可能简单，以避免影响`protobuf`编码、解码的性能。

Zigzag编码的计算方式为：

> `(n << 1) ^ (n >> 31)`

### 编码案例

![image-20230121004322599](Google-Protobuf%E8%AF%A6%E8%A7%A3/image-20230121004322599.png)

## 3.5 定长编码

`double、float`等数据结构的长度是确定的，当解析到这种类型的数据时，直接按照对应长度取数即可。

# 4 总结

1. `Protobuf`是一种高效的数据描述格式，具有平台无关、语言无关、可扩展等特点, 适合做数据存储、RPC 的通信协议等场景；
2. `Protobuf`采用`Varint`编码和`Zigzag`编码来编码数据, 其中`Varint`编码的思想是移除数字高位的`0`，用变长的二进制位来描述一个数字，对于小数字， 其编码长度短，可提高数据传输效率，但由于它在每个字节的最高位额外采用了一个标志位来标记其后是否还跟有有效字节，因此对于大的正数，它会比使用普通的定长格式占用更多的空间，另外对于负数，直接采用`Varint`编码将恒定占`10`个字节，`Zigzag`编码可将负数映射为无符号的正数，然后采用`Varint`编码进行数据压缩，在各种语言的`Protobuf`实现中，对于`int32`类型的数据，`Protobuf`都会转为 `uint64`而后使用`Varint`编码来处理, 因此当字段可能为负数时, 我们应使用`sint32`或`sint64`，这样`Protobuf`会按照`Zigzag`编码将数据变换后再采用`Varint`编码进行压缩，从而缩短数据的二进制位数；
3. `Protobuf`不是完全自描述的信息描述格式，接收端需要有相应的解码器(即`proto`定义)才可解析数据格式，序列化后的`Protobuf`数据不携带字段名，只使用字段编号来标识一个字段，因此更改`proto`的字段名不会影响数据解析(但这显然不是一种好的行为)，字段编号会被编码进二进制的消息结构中，因此我们应尽可能地使用小字段编号；
4. `Protobuf`是一种紧密的消息结构，编码后字段之间没有间隔，每个字段头由两部分组成: **字段编号**和`wire_type`，字段头可确定数据段的长度，因此其字段之前无需加入间隔，也无需引入特定的数据来标记字段末尾，因此`Protobuf`的编码长度短，传输效率高；

# 引用

- [Protobuf3 语法指南](https://colobu.com/2017/03/16/Protobuf3-language-guide/)
- [深入理解 ProtoBuf 原理与工程实践（概述）](https://xie.infoq.cn/article/98ed0dbe753b394c04a655ab7)
- [实现自己的Protobuf Any](https://juejin.cn/post/6982167437185663007)
- [Protobuf 使用指南](https://www.jianshu.com/p/cae40f8faf1e)
- [官方编码原理介绍](https://developers.google.com/protocol-buffers/docs/encoding)
- [Google Protobuf 编码原理](https://sunyunqiang.com/blog/protobuf_encode/)

