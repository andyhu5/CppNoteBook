

```c++
#define _WIN32_WINNT 0x0A00
#include "BeastApp.h"
#include <boost/asio.hpp>
#include <boost/asio/awaitable.hpp>
#include <boost/asio/ssl/stream.hpp>
#include <boost/url.hpp>
#include <boost/asio/ssl.hpp>
#include <boost/beast/http.hpp>
#include <boost/beast/core/flat_buffer.hpp>


using namespace std;

namespace asio = boost::asio;
namespace beast = boost::beast;
using tcp = asio::ip::tcp;
namespace http = boost::beast::http;

namespace asio = boost::asio;

namespace beast = boost::beast;
namespace http = beast::http;
namespace urls = boost::urls;
namespace ssl = boost::asio::ssl;


asio::awaitable<void> handle_client(tcp::socket socket)
{
	asio::streambuf buffer;
	for (;;) {
		co_await asio::async_read_until(socket, buffer, '\n');
		co_await asio::async_write(socket, buffer);
	}
}

asio::awaitable<void> async_main_() {
	auto executor = co_await asio::this_coro::executor;
	tcp::acceptor acceptor(executor, tcp::endpoint(tcp::v4(), 12345));
	for (;;) {
		tcp::socket socket = co_await acceptor.async_accept(asio::use_awaitable);
		asio::co_spawn(executor, handle_client(std::move(socket)), asio::detached);
	}
}

int main()
{
    asio::io_context io;

    asio::co_spawn(
        io,
        async_main(),
        asio::detached
    );

    io.run();
}
```

