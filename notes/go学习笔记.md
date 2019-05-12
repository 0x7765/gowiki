
# go语言中的一些注意点

## switch-case 一些用法
- [ ] 局部变量作用范围
  
  ```go
    // if 中的局部变量作用范围是整个if-else块
    func test01(x int) {
        if a, b := x+2, x+10; b%a == 0 {
            fmt.Println("a = ", a)
        } else if b%a == 1 {
            fmt.Println("b = ", b)
        } else {
            fmt.Println("a = ", a, "b = ", b)
        }
    }
  ```
- [ ] `case` 多条件

  ```go
    func test02(x int) {
        // 多个匹配条件 命中其中一个就行
        switch x {
        case 1, 2:
            fmt.Println("x is 1 or 2")
        case 3:
            fmt.Println("x is 3")
        case 4, 5, 6:
            fmt.Println("x is 4 or 5 or 6")
        default:
            fmt.Println(x)
        }
    }
  ```
- [ ] switch 初始化语句
  
  ```go
    func test03(x int) {
        // switch支持初始化
        switch y := x % 3; y {
        case 1:
            fmt.Println("remainder is 1", x)
        case 2:
            fmt.Println("remainder is 2", x)
        case 4:
            fmt.Println("remainder is 4", x)
        default:
            fmt.Println(x)
        }
    }
  ```
- [ ] 相邻的case不构成多条件匹配
  
  ```go
    func test04(x int) {
        switch y := x % 3; y {
        case 1: // 隐式 case1: break 
        case 2:
            fmt.Println("remainder is 2", x)
        case 4:
            fmt.Println("remainder is 4", x)
        default:
            fmt.Println(x)
        }
    }
  ```
- [ ] `fallthrough`
  
  ```go
    func test05(x int) {
        switch y := x % 3; y {
        case 1:
            fmt.Println("remainder is 1", x)
            fallthrough // 继续向下执行 下一个case 但是不再匹配case条件表达式
        case 2:
            fmt.Println("remainder is 2", x)
        case 4:
            fmt.Println("remainder is 4", x)
        default:
            fmt.Println(x)
        }
    }
    // 输入4 输出
    // remainder is 1 4
    // remainder is 2 4
  ```
## for-range的一些用法
- [ ] `for-range` 数据迭代的类型

|数据类型|1st value|2st value|
|----|----|----|
|string|index|s[index]|
|slice|index|s[index]|
|map|key|value|
|channel|element||

- [ ] 局部变量重复使用

  ```go
    func test06() {
        data := []string{"a", "b", "c"}
        for i, v := range data {
            println(&i, &v)
        }
    }
    // 输出
    // 0xc00006fed0 0xc00006fee8
    // 0xc00006fed0 0xc00006fee8
    // 0xc00006fed0 0xc00006fee8
  ```
- [ ] for-range会复制目标数据

  ```go
    func test07() {
        m := make(map[string]*Student)
        stus := []Student{
            {Name: "zhou", Age: 24},
            {Name: "li", Age: 23},
            {Name: "wang", Age: 22},
        }
        // for 遍历时，变量stu的地址不变 始终指向stus slice的最后一个元素
        //  那么 map中的所有元素的value都被替换成了 slice最后一个元素
        for _, stu := range stus {
            m[stu.Name] = &stu
        }

        for k, v := range m {
            fmt.Println("k = ", k, "v = ", v)
        }
    }
    // 输出
    // k =  zhou v =  &{wang 22}
    // k =  li v =  &{wang 22}
    // k =  wang v =  &{wang 22}
  ```
  改成一下就可以解决这个问题
  ```go
  	for _, stu := range stus {
		temp := stu
		m[stu.Name] = &temp
    }
    // 或者
    	for i := range stus {
		temp := stus[i]
		m[temp.Name] = &temp
	}
  ```
