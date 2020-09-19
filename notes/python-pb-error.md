# python 中pb赋值错误 

```protobuf
syntax = "proto3";
package demo;
message AddressBook &#123;
  repeated Person people = 1;
}
~~~~
message Person {
  string name = 1;
  int32 id = 2;
  string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
    }
  message PhoneNumber {
    string number = 1;
    PhoneType type = 2;
  }
}
````

赋值逻辑

```python
from address_book_pb2 import AddressBook, Person

if __name__ == '__main__':
    person = Person()
    person.name = 'adam'
    person.id = 1
    person.email = 'demo@demo.com'

    addr = AddressBook()
    addr.people = person

    print(addr)
```

这样赋值会直接报错

```python
Traceback (most recent call last):
  File "/Users/weixuan/code/pcode/proto_demo/main.py", line 10, in <module>
    addr.people = person
AttributeError: Assignment not allowed to repeated field "people" in protocol message object.
```

# 解决方式
官方文档中 不允许list直接赋值，可以使用下面两种方式解决 

## extend

```python
addr.people.extend([person])
```

## [:]


`python2` 和 `protobuf 2` 的版本下测试正常 `python3`和`protobuf 3` 下测试，无法赋值

```python
addr.people[:] = np.array(ps)
print(addr)
```