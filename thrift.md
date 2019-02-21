# thrift

## 名词解释

- IDL:`interface definition language` 接口定义语言

## 数据类型

### 基本数据类型

**thrift不支持无符号整型**

| 类型   | 大小   | 说明              | Python中的类型 | Golang中的类型 | Java中的类型 |
| ------ | ------ | ----------------- | -------------- | -------------- | ------------ |
| bool   | 1 byte | 8-bit             |                |                |              |
| byte   | 1 byte | 8-bit             |                |                |              |
| i16    | 2 byte | 16-bit整型 有符号 |                |                |              |
| i32    | 4 byte | 32-bit整型 有符号 |                |                |              |
| i64    | 8 byte | 64-bit整型 有符号 |                |                |              |
| double | 8 byte | 64-bit 浮点型     |                |                |              |
| string |        | string 类型       |                |                |              |
| binary |        | 字节数组          |                |                |              |

### 容器类型

**注意：容器中的元素类型可以是除了service 以外的任何合法thrift类型（包括结构体和异常）**

| 类型 | 说明       |
| ---- | ---------- |
| map  | map        |
| list | 动态数组   |
| set  | 去重的list |

### struct类型

结构体中，每个字段包含一个整数ID，数据类型、字段名，和一个可选的默认值。

```idl
struct Request {
    1:required i64 id ,
    2:string name,
    3:string ua,
    4:double amount,
    5:bool status,
    6:set<string> slots,
    7:optional i16 age
}

```

### 异常类型

类似struct类型，使用exception关键字声明

```idl
exception BadRequestError{
    1:i16 ec,
    2:string em
}
```

### service 类型

```idl
service AdService {
    string ping();
    Response searchAd(1:Request request);
    map<string,string> info();
}
```

## 生成对应的代码

```idl
struct Request {
    1:required i64 id ,
    2:string name,
    3:string ua,
    4:double amount,
    5:bool status,
    6:set<string> slots,
    7:optional i16 age
}

struct Response {
    1:i16 id,
    2:map<string,string> ads
}

exception BadRequestError {
    1:i16 ec,
    2:string em
}


service AdService {
    string ping();
    Response searchAd(1:Request request);
    map<string,string> info();
}
```

### 生成golang

```shell
thrift --gen go demo.thrift
```

### 生成java

```shell
thrift --gen java demo.thrift
```

### 生成python

```shell
thrift --gen py demo.thrift
```

## thrift服务端逻辑

生成的代码主要包含5个方面

- handler 服务端业务逻辑的实现
- processor 从Thrift框架 转移到 业务处理逻辑。因此是RPC调用，客户端要把 参数发送给服务端，而这一切由Thrift封装起来了，由Processor将收到的“数据”转交给业务逻辑去处理
- protocol 序列化合反序列化协议
- transport 传输层
- server 服务端的类型
