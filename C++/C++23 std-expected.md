# C++23 std::expected

The *std::expected* (C++23) comes with a monadic interface. 

Relying on the monadic interface prevents the typical if-then-else verbose error checking.

The *and_then* and *or_else* methods expect a callable that accepts a value/error and returns a *std::expected*.

The *transform* and *transform_error* methods expect a callable that accepts a value/error and returns a value/error.

```C++
#include <expected>
#include <system_error>
#include <string>

std::expected<std::string, std::error_condition> read_input();
std::expected<int, std::error_condition> to_int(const std::string& s);
int add_ten(int v);
std::expected<int, std::error_condition> log_error(
  const std::error_condition& err);

auto result = read_input()
    .and_then(to_int) // invoked if the expected contains a value
    // the callable has to return a std::expected, but can change
    // the type: std::expected<T,Err> -> std::expected<U,Err>
    .transform(add_ten) // invoked if the expected contains a value
    // the callable can return any type
    // std::expected<T,Err> -> std::expected<U,Err>
    .or_else(log_error); // invoked if the expected contains an error
    // the callable has to return a std::expected, but can change
    // the type: std::expected<V,T> -> std::expected<V,U>
```

[Open the example in Compiler Explorer.](https://compiler-explorer.com/z/WWsd5Y3W3)