# Boost

## Boost Corountine 2

What Why and How. Learn Boost Coroutine asynchronous programming by example.

**Video lecture: https://youtu.be/omEWXWUg5FA**

```cpp
#include <boost/coroutine2/all.hpp>
#include <iostream>
#include <vector>

using namespace boost::coroutines2;

void co_push_fun1(coroutine<int>::push_type& yield, int a) {
    for (int i = 0; i < a; i++) {
        // logic before yield
        std::cout << "coroutine pushing: " << i << std::endl;
        yield(i);
        // logic after yield
        std::cout << "coroutine pushed: " << i << std::endl;
    }
}

void co_pull_fun1(coroutine<int>::pull_type& yield) {
    int round = 0;
    while (yield) {
        // logic before yield
        std::cout << "coroutine pulled: " << round << std::endl;
        auto i = yield.get();
        std::cout << "coroutine pulled value: " << i << std::endl;
        yield();
        // logic after yield
        std::cout << "coroutine pulling: " << round << std::endl;
        round++;
    }
}

void test_co_pull_func1() {
    coroutine<int>::pull_type resume{std::bind(co_push_fun1, std::placeholders::_1, 6)};
    while (resume) {
        std::cout << "caller while: " << resume.get() << std::endl;
        resume();
    }
    coroutine<int>::pull_type resume2{std::bind(co_push_fun1, std::placeholders::_1, 3)};
    for (int c : resume2) {
        std::cout << "caller for: " << c << std::endl;
    }
}

void test_co_push_func1() {
    coroutine<int>::push_type sink{co_pull_fun1};
    std::vector<int> v{1, 1, 2, 3};
    for (int i : v) {
        std::cout << "caller sinking: " << i << std::endl;
        sink(i);
        std::cout << "caller sinked: " << i << std::endl;
    }
}

int main() {
    std::cout << "=============" << std::endl;
    test_co_pull_func1();
    std::cout << "=============" << std::endl;
    test_co_push_func1();
}

```