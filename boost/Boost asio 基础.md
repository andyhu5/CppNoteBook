Boost asio 基础

```
async_read_some() 是向 Asio 注册一个异步读取操作，操作系统负责监听 socket 状态；当 socket 有数据可读时，Asio 将数据复制到用户提供的 buffer，并把 handler 投递到 io_context 执行。
```



## post和网络IO有什么区别？

这里容易混淆。

### post

主动投递任务：

```
你的线程
    |
    |
    post()
    |
    |
io_context
```

例如：

```c#
asio::post(ctx, task);
```





### async_read

注册一个IO事件：

```
socket.async_read_some(
    buffer,
    handler
);
```

流程：

```
你的程序

注册read

    |
    |
操作系统等待数据

    |
    |
数据到达

    |
    |
io_context执行handler
```

------

区别：

|            | post     | async_read |
| ---------- | -------- | ---------- |
| 谁产生事件 | 用户代码 | 操作系统   |
| 是否等待IO | 否       | 是         |
| 作用       | 执行任务 | 等待IO完成 |
| 类似       | 消息投递 | IO回调     |

------

## 5. io_context里面有什么？

可以简单理解：

```
io_context
{
    ready queue
    {
        handler1
        handler2
        handler3
    }


    waiting operations
    {
        timer
        socket read
        socket write
    }
}
```

其中：

### post

直接放入：

```
ready queue
```

例如：

```
[
 print(),
 update(),
 callback()
]
```

### async_wait

放入：

```
waiting operations
```

例如：

```
timer
 |
 |
1秒后
 |
 |
ready queue
```