# 面对对象

## 魔法函数定义

函数名 以(双下划线)`___`开头和结尾的函数 

## 示例

```python
# -*- coding: utf-8 -*-
from logger import logger


class Company(object):

    # 双下划线开头的函数 称为魔法函数
    def __init__(self, employees):
        self.employees = employees

    # 使得当前对象是 可以迭代访问的
    def __getitem__(self, item):
        return self.employees[item]

    def __len__(self):
        return len(self.employees)

    # tostring 方法
    def __str__(self):
        return ",".join(self.employees)


if __name__ == '__main__':
    c = Company(employees=['tank', 'tom', 'jim'])
    # 普通方式遍历
    for employee in c.employees:
        logger.info("name is {name}".format(name=employee))

    # getitem 魔法函数之后
    for employee in c:
        logger.info("name is {name}".format(name=employee))

    # by index
    logger.info("slice {e}".format(e=c[0]))
    logger.info("slice {e}".format(e=c[1]))
    logger.info("slice {e}".format(e=c[2]))

    logger.info("slice {c}".format(c=c))
```

`Java`中实现可迭代对象 

```java
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.util.Iterator;
import java.util.NoSuchElementException;

@NoArgsConstructor
public class IterableObject<T> implements Iterable<T>, Iterator<T> {
	
	// 需要迭代的数据
	@Getter
	@Setter
	private T[] data;
	// 索引
	private int index;
	
	public IterableObject(T[] data) {
		this.data = data;
	}
	
	@Override
	public Iterator<T> iterator() {
		index = 0;
		return this;
	}
	
	@Override
	public boolean hasNext() {
		return this.index < data.length;
	}
	
	@Override
	public T next() {
		if (hasNext()) {
			return data[index++];
		}
		throw new NoSuchElementException("has no next");
	}
}
```

## 抽象基类

