#  `*args` vs `**kwargs`

##  `*` vs `**`

经常可以看到参数中 `*args` 和 `**kwargs` 区别如下 

### 函数定义的参数中(形参)

1.  `*args` 表示任何且多个无名参数，一般用于不定长参数定义 ，本质是一个`tuple` 
2.  `**kwargs` 表示关键字参数,本质是一个字典 

```python
def foo(*args):
    print(type(args))
    print(args)

def bar(**kwargs):
    print(type(kwargs))
    print(kwargs)

if __name__ == '__main__':
    # 输出 <class 'tuple'>
    # 输出 ('1', 2, 3.141592653)
    foo("1", 2, 3.141592653)

    # 输出 <class 'dict'>
    # {'a': '1', 'b': '2', 'c': 3.14}
    bar(a="1", b="2", c=3.14)
```

### 函数定义的实参中

1.  `func(*args)` 将args中的每个元素当做位置参数传入 
2.  `func(**kwargs)`  字典 kwargs 变成关键字参数传递

```python
    l = [1, 2, 3]
    # 输出 ([1, 2, 3])
    # foo是一个不定长参数的函数
    # 函数的参数个数是一个 参数是一个数组 [1,2,3]
    foo(l)

    # 输出 (1,2,3)
    # *l 是将l中的每个元素当做位置参数传进去
    # 函数的参数是3个，分别是 1,2,3
    foo(*l)
    
    d = {'a': 'b'}
    # 输出 {'a': 'b'}
    bar(**d)
```

### 组合使用

```python
def foo_bar(*args, **kwargs) -> str:
    """
    :param args: 表示不定长参数 (位置参数)
    :param kwargs: 表示关键字参数
    :return:
    """
    print(args)
    print(kwargs)

    return "args is {args}, argv is {kwargs}\n".format(args=args, kwargs=kwargs)
  
if __name__ == '__main__':
    # args is ('1', 2, 3.14), argv is {'name': 'Jone', 'age': 20}
    print(foo_bar("1", 2, 3.14, name="Jone", age=20))
    # args is (), argv is {'name': 'Jone', 'age': 20}
    print(foo_bar(name="Jone", age=20))
    # args is ('1', 2, 3.14), argv is {}
    print(foo_bar("1", 2, 3.14))
```