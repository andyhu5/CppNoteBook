不是。这里涉及两个完全不同的概念：

```go
v := x.(int)
```

是**类型断言（type assertion）**。

而：

```go
var x any
```

是**声明一个 interface 类型变量**。

------

例如：

```go
package main

import "fmt"

func main() {
	var x any

	x = 123

	v := x.(int)

	fmt.Println(v)
}
```

执行过程：

第一步：

```go
var x any
```

创建一个空接口变量：

```
x
+----------------+
| type: nil      |
| value: nil     |
+----------------+
```

第二步：

```go
x = 123
```

现在 interface 内部保存：

```
x
+----------------+
| type: int      |
| value: 123     |
+----------------+
```

第三步：

```go
v := x.(int)
```

意思：

> 我确定 x 里面保存的是 int，把它取出来，并转换成 int 类型变量。

结果：

```go
v == 123
```

此时：

```go
v
```

的类型是：

```go
int
```

不是 `any`。

------

如果类型不匹配：

```go
var x any = "hello"

v := x.(int)
```

运行时 panic：

```
panic: interface conversion: interface {} is string, not int
```

因为：

```
x
+----------------+
| type: string   |
| value:"hello" |
+----------------+
```

不是 int。

------

安全写法：

```go
v, ok := x.(int)

if ok {
	fmt.Println(v)
}
```

这里：

```go
v
```

类型：

```go
int
```

而：

```go
ok
```

类型：

```go
bool
```

------

和 C++ 类比：

Go:

```go
var x any = 123

v := x.(int)
```

类似 C++：

```cpp
std::any x = 123;

int v = std::any_cast<int>(x);
```

而：

```go
var x any
```

类似：

```cpp
std::any x;
```

------

所以：

| 代码           | 含义                               |
| -------------- | ---------------------------------- |
| `var x any`    | 声明一个可以保存任意类型的接口变量 |
| `x = 123`      | 把 int 放入 interface              |
| `x.(int)`      | 从 interface 中取出 int            |
| `v := x.(int)` | 得到一个真正的 int 变量            |

简单记：

> `any` 是盒子，`x.(int)` 是从盒子里拆出 int。