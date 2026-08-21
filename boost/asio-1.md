```c++
#define _WIN32_WINNT 0x0A00
#include <iostream>
#include <mutex>
#include <queue>
#include <string>
#include <thread>
#include <boost/asio.hpp>

namespace asio = boost::asio;


class MessageQueue
{
public:

    explicit MessageQueue(asio::io_context& ctx)
        : m_ctx(ctx)
    {
    }


    void push(std::string msg)
    {
        {
            std::lock_guard<std::mutex> lock(m_mutex);
            m_queue.push(std::move(msg));
        }


        asio::post(
            m_ctx,
            [this]
            {
                consume();
            });
    }


private:

    void consume()
    {
        while (true)
        {
            std::string msg;

            {
                std::lock_guard<std::mutex> lock(m_mutex);

                if (m_queue.empty())
                    break;

                msg = std::move(m_queue.front());
                m_queue.pop();
            }

            handle(msg);
        }
    }


    void handle(const std::string& msg)
    {
        std::cout
            << "Received: "
            << msg
            << std::endl;
    }


private:

    std::queue<std::string> m_queue;
    std::mutex m_mutex;
    asio::io_context& m_ctx;
};



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

            while (true)
            {
                std::cout << "Input: ";

                std::getline(std::cin, line);


                if (line == "exit")
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

