```cpp
#include <boost/asio.hpp>

#include <chrono>
#include <iostream>
#include <thread>

namespace asio = boost::asio;

class Service
{
  public:
    explicit Service(asio::io_context &ctx);

    void stop();

    void input(char c);

  private:
    void timerExpired(const boost::system::error_code &ec);

    asio::io_context &m_ctx;
    asio::steady_timer m_timer;
    bool m_stop{};
    int m_charCount{};
};

Service::Service(asio::io_context &ctx) :
    m_ctx(ctx),
    m_timer(ctx, asio::chrono::seconds(1))
{
    post(m_ctx,
         [] {
             std::cout << "Input: ";
         });
    m_timer.async_wait([this](const boost::system::error_code &ec) { timerExpired(ec); });
}

void Service::stop()
{
    m_timer.cancel();
    m_stop = true;
}

void Service::input(char c)
{
    post(m_ctx,
         [c, this]
         {
             std::string output = " > ";
             if (std::isprint(c))
             {
                 output += c;
             }
             std::cout << output;
         });
}

void Service::timerExpired(const boost::system::error_code &ec)
{
    if (ec || m_stop)
    {
        return;
    }

    std::time_t now = std::time(nullptr);
    std::cout << "now: " << std::ctime(&now) << std::flush;

    m_timer.expires_after(asio::chrono::seconds(1));
    m_timer.async_wait([this](const boost::system::error_code &ec) { timerExpired(ec); });
}

void getConsoleInput(Service &svc)
{
    static const int CTRL_C = 3;

    while (true)
    {
        int c = std::cin.get();
        if (c == CTRL_C)
        {
            break;
        }

        svc.input(static_cast<char>(c));
    }
}

int main() {
  asio::io_context ctx;
  Service svc(ctx);

  std::thread thread([&svc] {
    getConsoleInput(svc);
    svc.stop();
  });

  ctx.run();
  thread.join();
  std::cout << "Input: ";
  return 0;
}
```

