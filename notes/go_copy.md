# 深浅拷贝 

>   深浅拷贝是针对指针或者引用类型而言。

## 浅拷贝

浅拷贝只是内存中重新开辟了一块`名称空间`，但是被拷贝对象中的数据 是公用的。拷贝对象被修改，会同时影响拷贝前后的两个变量的值。

常见的赋值 

Java 

```java
        Student s1 = new Student();
        // 这里是浅拷贝  s3的地址 == s1的地址
        Student s3 = s1;

        System.out.println("s1 = " + s1);
        System.out.println("s3 = " + s3);

        // s1、s3 的值都会被修改
        s3.setName("张三");
        System.out.println("s1:" + s1.getName());
        System.out.println("s2:" + s3.getName());
```

golang 

```go
	s1 := &Student{
		Id:   "1",
		Name: "张三",
	}

	s2 := s1

	log.Printf("s1 %p s2 %p\n", s1, s2)

	s2.Name = "李四"

	log.Printf("s1 %+v  s2 %+v\n", s1, s2)
```

## Java 中的`clone`

1. 当前类中没有引用其他引用类型时，此时默认的`clone`是深拷贝

```java
public class Student implements Cloneable {

    @Getter @Setter private String id;
    @Getter @Setter private String name;

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}
```

2.  当类中引用了其他的引用类型时，默认的clone无法实现深拷贝

```java
public class Student implements Cloneable {

    @Getter @Setter private String id;
    @Getter @Setter private String name;

    @Getter private Map<String, String> data = Maps.newHashMap();

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}
		// test
    @Test
    public void test() throws CloneNotSupportedException {
        // 创建Student对象
        Student s1 = new Student();

        s1.setId("1");
        s1.setName("Rye");
        s1.getData().put("Hello", "world");

        // 通过clone 拷贝一个对象
        Student s2 = (Student) s1.clone();

        System.out.println("s1:" + s1.getData());
        System.out.println("s2:" + s2.getData());

        System.out.println("-------------");

        s1.setId("2");
        s1.setName("zhangsan");
        s1.getData().remove("Hello");
        // 此时 s1 s2 中的map都会被修改
        System.out.println("s1:" + s1.getData());
        System.out.println("s2:" + s2.getData());
        System.out.println("s1 == s2 ? ==> " + (s1 == s2));
    }
```

## 如何实现深拷贝 

1.  Java 可以在clone方法中实现。

    > 但是当类中引用多个 外部引用时，每个引用对象都需要new 一个新的对象。这样改动比较麻烦，同时也不方便扩展，每次增删字段都需要同步。

2.  利用序列化。

## go中如何实现

使用序列化 `gob` 或者 `json`

**只能拷贝 对象中的public 字段，非导出字段不能复制。**

```go
type Student struct {
	Id   string
	Name string
	Info *Info
}

type Info struct {
	No uint64
}

func (s *Student) DeepCopy() (*Student, error) {
	var buf bytes.Buffer
	if err := gob.NewEncoder(&buf).Encode(s); err != nil {
		return nil, err
	}
	dst := &Student{}
	err := gob.NewDecoder(bytes.NewBuffer(buf.Bytes())).Decode(dst)
	return dst, err
}

// 通用型函数
func DeepCopy(dst, src interface{}) error {
	var buf bytes.Buffer
	if err := gob.NewEncoder(&buf).Encode(src); err != nil {
		return err
	}
	return gob.NewDecoder(bytes.NewBuffer(buf.Bytes())).Decode(dst)
}
```



