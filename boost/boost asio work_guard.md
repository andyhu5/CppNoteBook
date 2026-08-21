你遇到的问题原因是：**`ctx.run()` 在主线程先执行时，`io_context` 没有任何未完成任务，会立即返回。**

你的流程实际可能是：

```text
main线程:

创建ctx

创建MessageQueue

启动input_thread

ctx.run()

    |
    |
发现没有任务

    |
    |
立即返回


程序继续join

input线程等待输入

push消息

但是:

io_context已经停止运行

handler不会执行
```

所以：

```cpp
handle(msg);
```

永远不会被调用。

------

## 修复方法1：使用 `executor_work_guard`（推荐）

告诉 `io_context`：

> 即使当前没有任务，也保持运行。

增加：

```cpp
asio::executor_work_guard<asio::io_context::executor_type> work_guard(
    ctx.get_executor()
);
```

完整：

```cpp
int main()
{
    asio::io_context ctx;


    auto work_guard =
        asio::make_work_guard(ctx);


    MessageQueue msgqueue(ctx);


    std::thread input_thread(
        [&msgqueue, &ctx]()
        {
            std::string line;

            while(true)
            {
                std::cout << "Input: ";

                std::getline(std::cin, line);


                if(line == "exit")
                {
                    ctx.stop();
                    break;
                }


                msgqueue.push(line);
            }

        }
    );


    ctx.run();


    input_thread.join();

    return 0;
}
```

------

## 为什么 work_guard 有效？

没有：

```text
io_context

任务队列:

[]

ctx.run()

发现没有任务

退出
```

------

有：

```text
io_context

work_guard
    |
    |
保持运行


ctx.run()

等待任务
```

当：

```cpp
msgqueue.push("hello")
```

执行：

```cpp
asio::post(ctx, handler);
```

变成：

```text
io_context

任务队列:

[
 consume()
]


ctx.run()

    |
    v

consume()

    |
    v

handle()
```

------

## 另外你的 MessageQueue 还有一个问题

这里：

```cpp
void consume()
{
    while (!m_queue.empty())
    {
        auto msg = m_queue.front();
        m_queue.pop();

        handle(msg);
    }
}
```

存在竞态：

线程A：

```cpp
if(!queue.empty())
```

线程B：

```cpp
push()
```

可能同时修改queue。

建议：

```cpp
void consume()
{
    while(true)
    {
        std::string msg;


        {
            std::lock_guard<std::mutex> lock(m_mutex);


            if(m_queue.empty())
                break;


            msg = std::move(m_queue.front());

            m_queue.pop();
        }


        handle(msg);
    }
}
```

------

## 修复后的运行效果

输入：

```
Input: Hello
```

流程：

```
input thread

getline()
    |
    |
msgqueue.push("Hello")
    |
    |
asio::post()
    |
    |
io_context queue


main thread

ctx.run()

    |
    |
consume()

    |
    |
handle()

    |
    |
Received: Hello
```

输出：

```
Received: Hello
```

------

这也是 Boost.Asio 服务器程序的标准模式：

```cpp
io_context ctx;

work_guard 保活;

启动worker线程:

    ctx.run();


其他线程:

    post(handler);

停止:

    ctx.stop();

join();
```

你的例子已经接近真实 Asio 服务端内部的消息分发模型了。