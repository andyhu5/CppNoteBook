`boost::intrusive_ptr` 主要解决的是 **`std::shared_ptr` 的额外内存分配和控制块开销问题**。

简单说：

- `shared_ptr`：引用计数在**对象外部**
- `intrusive_ptr`：引用计数在**对象内部**

------

## 1. shared_ptr 的结构

例如：

```cpp
auto p = std::make_shared<Session>();
```

内存大概：

```
heap
+-------------------+
| Session object    |
+-------------------+
| control block     |
|  - ref count      |
|  - weak count     |
|  - deleter        |
+-------------------+

p
 |
 +-----> object
 |
 +-----> control block
```

`control block` 是额外分配的。

例如：

```cpp
std::shared_ptr<Session> a;
std::shared_ptr<Session> b;
```

两个指针：

```
a ----+
      |
b ----+----> control block(ref=2)
              |
              +----> Session
```

每次：

```cpp
a = b;
```

都会：

```
atomic++ / atomic--
```

引用计数。

------

## 2. intrusive_ptr 的结构

Boost:

```cpp
boost::intrusive_ptr<Session>
```

对象自己保存引用计数：

```cpp
class Session
{
    std::atomic<int> ref_count{0};

public:

    void add_ref()
    {
        ref_count++;
    }

    void release()
    {
        if(--ref_count == 0)
            delete this;
    }
};
```

内存：

```
heap

+----------------+
| Session        |
|                |
| ref_count      |  <--- 在对象里面
| socket         |
| buffer         |
+----------------+
```

没有 control block。

------

## 3. 它解决什么问题？

主要三个：

------

### ① 减少一次内存分配

shared_ptr：

```cpp
make_shared
```

虽然已经优化为一次分配：

```
object + control block
```

但是：

```cpp
shared_ptr(new T)
```

是：

```
malloc object

malloc control block
```

两次。

intrusive_ptr：

```
malloc object
```

只有一次。

对于：

- 网络连接
- 游戏对象
- 消息对象
- 高频创建销毁对象

有价值。

------

### ② 减少内存占用

假设：

```cpp
100万个 Connection
```

每个：

```
control block 32 bytes
```

那么：

```
32MB
```

只是控制块。

如果对象本身：

```
Connection = 64 bytes
```

那么：

```
对象:
64MB

控制块:
32MB

增加50%
```

这时候 intrusive_ptr 有意义。

------

### ③ 更好的缓存局部性

shared_ptr:

```
CPU cache

Session
    |
    |
    v
control block
```

访问引用计数时可能访问另一个 cache line。

intrusive:

```
Session

+----------------+
| ref_count      |
| data           |
+----------------+
```

数据和引用计数一起。

------

## 4. 为什么标准库没有 intrusive_ptr？

因为 intrusive_ptr 要求对象参与生命周期管理。

例如：

```cpp
class Session
{
    int ref_count;
};
```

侵入了类设计。

而：

```cpp
std::shared_ptr<T>
```

可以：

```cpp
std::shared_ptr<int>
std::shared_ptr<MyClass>
std::shared_ptr<第三方库对象>
```

不需要修改对象。

所以：

|          | shared_ptr | intrusive_ptr |
| -------- | ---------- | ------------- |
| 侵入对象 | 否         | 是            |
| 控制块   | 外部       | 内部          |
| 内存开销 | 较高       | 较低          |
| 使用简单 | ★★★★★      | ★★            |
| 性能     | 好         | 更高          |
| 适合     | 普通应用   | 高性能系统    |

------

## 5. Boost.Asio 中什么时候考虑 intrusive_ptr？

普通：

```cpp
1000 connections
```

不用考虑。

```cpp
class Session :
    public enable_shared_from_this<Session>
```

非常合适。

但是：

例如：

- 高性能网关
- 万级百万级 websocket
- 游戏服务器
- 消息中间件
- 高频交易系统

可能：

```
10万~100万个对象
```

这时：

```
shared_ptr:
    control block
    atomic refcount

intrusive_ptr:
    object内部refcount
```

差距明显。

------

## 6. 一个 Asio Session 的 intrusive_ptr 例子

```cpp
class Session
{
public:

    std::atomic<int> refs{0};


    void add_ref()
    {
        refs.fetch_add(1);
    }


    void release()
    {
        if(refs.fetch_sub(1)==1)
            delete this;
    }


    void async_read()
    {
        socket.async_read_some(
            buffer,
            [this](auto ec, auto n)
            {
                handle_read();
                release();
            });
    }


private:

    boost::asio::ip::tcp::socket socket;
};
```

然后：

```cpp
boost::intrusive_ptr<Session> session;
```

生命周期由对象自己控制。

------

对于你前面讨论的 `ticker`：

几十个、几百个 timer：

```cpp
shared_ptr
```

是最佳选择。

如果设计的是：

```
百万级定时任务调度器
```

才会考虑：

- intrusive_ptr
- timer wheel
- object pool
- slab allocator

这些高性能技术。