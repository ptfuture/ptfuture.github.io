# C++ Language basics

## Function pointer, std::function, functor, lambda

C++ 11 to 20: Compare different function definitions as callback functions. What are the pros and cons of these different ways of defining function types, which one we should choose.

**Video lecture: https://youtu.be/5yb9TEoMr2A**

```cpp
#include <iostream>

int regular_function(int arg) {
    return arg;
}

int three_arg_function(std::string arg1, double arg2, int arg3) {
    return arg3;
}

class function_class {
   public:
    int three_arg_function(std::string arg1, double arg2, int arg3) {
        return arg3;
    }
};

typedef int (*func_ptr)(int);
void using_func_ptr(func_ptr callback) {
    callback(1);
}

using func_type = int(int);
void using_func_type(func_type callback) {
    callback(1);
}

using stdfunc_type = std::function<int(int)>;
void using_std_func(stdfunc_type callback) {
    callback(1);
}

struct functor_type {
    int operator()(int arg) {
        return arg;
    }
};
void using_functor(functor_type callback) {
    callback(1);
}

auto global_cap = 15;

int main() {
    auto cap = 12;
    auto ff = std::bind(three_arg_function, "something", 2.0, std::placeholders::_1);
    function_class fclass;
    auto clz_ff = std::bind(&function_class::three_arg_function, &fclass, "something else", 3.0,
                            std::placeholders::_1);
    // using func_ptr
    using_func_ptr(regular_function);
    using_func_ptr([](auto arg) {
        return arg * global_cap;
    });
    // using_func_ptr([&cap](auto arg) { return arg * cap; });  // compile error
    // using_func_ptr([&](auto arg) { return arg * cap; });     // compile error
    // using_func_ptr([=](auto arg) { return arg * cap; });     // compile error
    // using_func_ptr([cap](auto arg) { return arg * cap; });   // compile error
    // using_func_ptr(ff);                                      // compile error
    // using_func_ptr(clz_ff);                                  // compile error
    // using_func_ptr(functor_type());                          // compile error

    // using func_type
    using_func_type(regular_function);
    using_func_type([](auto arg) {
        return arg * global_cap;
    });
    // using_func_type([&cap](auto arg) { return arg * cap; });  // compile error
    // using_func_type([&](auto arg) { return arg * cap; });     // compile error
    // using_func_type([=](auto arg) { return arg * cap; });     // compile error
    // using_func_type([cap](auto arg) { return arg * cap; });   // compile error
    // using_func_type(ff);                                      // compile error
    // using_func_type(clz_ff);                                  // compile error
    // using_func_type(functor_type());                          // compile error

    // using functor_type
    using_functor(functor_type());
    // using_functor(regular_function);                           // compile error
    // using_functor([](auto arg) { return arg * global_cap; });  // compile error
    // using_functor([&cap](auto arg) { return arg * cap; });     // compile error
    // using_functor([&](auto arg) { return arg * cap; });        // compile error
    // using_functor([=](auto arg) { return arg * cap; });        // compile error
    // using_functor([cap](auto arg) { return arg * cap; });      // compile error
    // using_functor(ff);                                         // compile error
    // using_functor(clz_ff);                                     // compile error

    // using std_func
    using_std_func(regular_function);
    using_std_func([](auto arg) {
        return arg * global_cap;
    });
    using_std_func([&cap](auto arg) {
        return arg * cap;
    });
    using_std_func([&](auto arg) {
        return arg * cap;
    });
    using_std_func([=](auto arg) {
        return arg * cap;
    });
    using_std_func([cap](auto arg) {
        return arg * cap;
    });
    using_std_func(ff);
    using_std_func(clz_ff);
    using_std_func(functor_type());

    return 0;
}
```