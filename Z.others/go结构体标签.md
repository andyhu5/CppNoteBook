你看到的：

```go
type ToCKLog struct {
	Id      string `json:"id" bson:"id"`
	Message string `json:"message"`
}
```

这里 `json:"id" bson:"id"` 叫 **struct tag（结构体标签）**。

它不是 Go 语法的一部分，而是附加给字段的**元数据**，供反射（reflection）机制读取。

------

### 1. `json:"id"` 的作用

Go 标准库 `encoding/json` 使用它来控制 JSON 序列化/反序列化时的字段名称。

例如：

```go
package main

import (
	"encoding/json"
	"fmt"
)

type ToCKLog struct {
	Id      string `json:"id"`
	Message string `json:"message"`
}

func main() {
	log := ToCKLog{
		Id:      "1001",
		Message: "hello",
	}

	data, _ := json.Marshal(log)

	fmt.Println(string(data))
}
```

输出：

```json
{"id":"1001","message":"hello"}
```

------

如果没有 tag：

```go
type ToCKLog struct {
	Id      string
	Message string
}
```

默认：

```json
{"Id":"1001","Message":"hello"}
```

Go 默认使用字段名。

------

### 2. `bson:"id"` 的作用

`bson` 是 MongoDB 使用的格式。

例如：

```go
type ToCKLog struct {
	Id string `bson:"id"`
}
```

MongoDB 保存时：

```json
{
    "id": "1001"
}
```

而不是：

```json
{
    "Id": "1001"
}
```

通常配合：

```go
go.mongodb.org/mongo-driver/bson
```

使用。

------

### 3. 一个字段可以有多个 tag

例如：

```go
type User struct {
	Name string `json:"name" bson:"name"`
	Age  int    `json:"age" bson:"age"`
}
```

表示：

| Go字段 | JSON | BSON |
| ------ | ---- | ---- |
| Name   | name | name |
| Age    | age  | age  |

这样同一个结构体可以同时用于：

- HTTP API JSON
- MongoDB BSON
- 数据传输对象 DTO

------

### 4. tag 本质是什么？

编译后它只是一个字符串：

```go
`json:"id" bson:"id"`
```

通过反射读取：

```go
package main

import (
	"fmt"
	"reflect"
)

type Log struct {
	Id string `json:"id" bson:"id"`
}

func main() {
	t := reflect.TypeOf(Log{})

	field, _ := t.FieldByName("Id")

	fmt.Println(field.Tag.Get("json"))
	fmt.Println(field.Tag.Get("bson"))
}
```

输出：

```
id
id
```

------

### 5. 常见 tag

JSON：

```go
json:"name"
```

忽略字段：

```go
Password string `json:"-"`
```

omitempty：

```go
Name string `json:"name,omitempty"`
```

例如：

```go
User{Name:""}
```

输出：

```json
{}
```

而不是：

```json
{"name":""}
```

------

### 6. 和 C++ 类比

Go 的 struct tag 有点类似 C++ 中：

- attribute
- annotation
- reflection metadata

例如：

C++：

```cpp
[[json("id")]]
std::string Id;
```

Go：

```go
Id string `json:"id"`
```

区别是 Go 原生支持反射读取 tag，生态大量依赖它。

在 Go Web 开发中，这种写法非常常见，例如：

```go
type Request struct {
	UserID int `json:"user_id" validate:"required"`
}
```

同一个字段可以同时被：

- JSON 序列化
- BSON/MongoDB
- 参数校验库
- ORM

读取。