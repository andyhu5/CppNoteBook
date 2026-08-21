go类型系统

| 格式 | 作用           |
| ---- | -------------- |
| `%T` | 打印变量的类型 |
| `%v` | 打印变量的值   |
| `%s` | 字符串         |
| `%d` | 整数           |

```
# 打印类型
fmt.Printf("Type of param: %s\n", reflect.TypeOf(param))

```

