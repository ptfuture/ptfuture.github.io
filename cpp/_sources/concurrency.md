# Concurrency

## Dead lock prevention using std::lock

Learn how to use std::lock for locking multiple mutex and preventing dead lock.

**Video lecture: https://youtu.be/71qO7ybwlUA**

```cpp
#include <iostream>
#include <mutex>
#include <thread>

using namespace std::chrono_literals;

class mutexdead1 {
    std::mutex mut1;
    std::mutex mut2;

   public:
    void accessor1() {
        // std::lock_guard lock1(mut1); // could cause deadlock
        // std::lock_guard lock2(mut2);
        std::lock(mut1, mut2);
        std::lock_guard lock1(mut1, std::adopt_lock);
        std::lock_guard lock2(mut2, std::adopt_lock);
        std::cout << "accessor1" << std::endl;
    }

    void accessor2() {
        // std::lock_guard lock1(mut1); // could cause deadlock
        // std::lock_guard lock2(mut2);
        std::lock(mut2, mut1);
        std::lock_guard lock2(mut2, std::adopt_lock);
        std::lock_guard lock1(mut1, std::adopt_lock);

        std::cout << "accessor2" << std::endl;
    }
};

void thread1(std::shared_ptr<mutexdead1> p) {
    for (int i : {1, 2, 3, 4, 5, 6}) {
        std::this_thread::sleep_for(2000ms - std::chrono::milliseconds(i));
        p->accessor1();
    }
}

void thread2(std::shared_ptr<mutexdead1> p) {
    for (int i : {1, 1, 1, 1, 1, 1}) {
        std::this_thread::sleep_for(2000ms - std::chrono::milliseconds(i));
        p->accessor2();
    }
}

int main() {
    auto p = std::make_shared<mutexdead1>();
    std::thread t1(thread1, p);
    std::thread t2(thread2, p);
    t1.join();
    t2.join();
}
```

## std::atomic memory orders

Compare relaxed, consume, acquire, release, sequence consistent memory orders by example.

**Video lecture: https://youtu.be/UzYVDki31Hg**

```cpp
#include <iostream>
#include <atomic>
#include <thread>
#include <random>
#include <vector>
#include <cassert>

using namespace std::chrono_literals;

void test_atomic_relaxed() {
    // Atomic operations tagged memory_order_relaxed are not synchronization operations; they do not
    // impose an order among concurrent memory accesses. They only guarantee atomicity and
    // modification order consistency.
    std::atomic<int> x = {0};
    std::atomic<int> y = {0};
    std::jthread t1([&]() {
        auto r1 = y.load(std::memory_order_relaxed);  // A
        x.fetch_add(r1, std::memory_order_relaxed);   // B
    });

    std::jthread t2([&]() {
        auto r2 = x.load(std::memory_order_relaxed);  // C
        y.fetch_add(42, std::memory_order_relaxed);   // D
    });
    // Possible outcomes
    // CDAB: r1 = 42, r2 = 0
    // ABCD: r1 = 0, r2 = 42
    // DABC: r1 = 42, r2 = 42
    // ...
}

void test_consume_release() {
    // If an atomic store in thread A is tagged memory_order_release and an atomic load in thread B
    // from the same variable that read the stored value is tagged memory_order_consume, all memory
    // writes (non-atomic and relaxed atomic) that happened-before the atomic store from the point
    // of view of thread A, become visible side-effects within those operations in thread B into
    // which the load operation carries dependency, that is, once the atomic load is completed,
    // those operators and functions in thread B that use the value obtained from the load are
    // guaranteed to see what thread A wrote to memory.

    // The synchronization is established only between the threads releasing and consuming the same
    // atomic variable. Other threads can see different order of memory accesses than either or both
    // of the synchronized threads.

    std::atomic<std::string*> ptr{};
    int data;
    std::string* p{};
    std::jthread producer([&]() {
        p = new std::string("Hello");
        data = 42;
        ptr.store(p, std::memory_order_release);
    });

    std::jthread consumer([&]() {
        std::string* p2{};
        while (!(p2 = ptr.load(std::memory_order_consume)))
            ;
        assert(*p == "Hello");   // always true
        assert(*p2 == "Hello");  // always true: *p2 carries dependency from ptr
        assert(data == 42);      // may and may not be true: data does not carry dependency from ptr
        delete p2;
    });
}

void test_acquire_release_1() {
    // If an atomic store in thread A is tagged memory_order_release and an atomic load in thread B
    // from the same variable is tagged memory_order_acquire, all memory writes (non-atomic and
    // relaxed atomic) that happened-before the atomic store from the point of view of thread A,
    // become visible side-effects in thread B. That is, once the atomic load is completed, thread B
    // is guaranteed to see everything thread A wrote to memory.

    std::atomic<std::string*> ptr{};
    int data;
    std::string* p{};
    std::jthread producer([&]() {
        p = new std::string("Hello");
        data = 42;
        ptr.store(p, std::memory_order_release);
    });

    std::jthread consumer([&]() {
        std::string* p2{};
        while (!(p2 = ptr.load(std::memory_order_acquire)))
            ;
        assert(*p == "Hello");   // always true
        assert(*p2 == "Hello");  // always true
        assert(data == 42);      // always true
        delete p2;
    });
}

void test_acquire_release_2() {
    // If an atomic store in thread A is tagged memory_order_release and an atomic load in thread B
    // from the same variable is tagged memory_order_acquire, all memory writes (non-atomic and
    // relaxed atomic) that happened-before the atomic store from the point of view of thread A,
    // become visible side-effects in thread B. That is, once the atomic load is completed, thread B
    // is guaranteed to see everything thread A wrote to memory.

    std::vector<int> data;
    std::atomic<int> flag = {0};
    std::jthread producer([&]() {
        data.push_back(42);
        flag.store(1, std::memory_order_release);
    });

    std::jthread consumer([&]() {
        int expected = 1;
        // Compares the contents of the flag with expected:
        // - if true, it replaces the flag value with 2. (performs read-modify-write operation)
        // - if false, it replaces expected with the flag. (performs load operation)
        // returns
        // - true if expected compares equal to the contained value.
        // - false otherwise.
        while (!flag.compare_exchange_weak(expected, 2, std::memory_order_acq_rel)) {
            expected = 1;
        }
        assert(data.at(0) == 42);  // always true
    });
}

void test_seq_cst() {
    // A load operation with this memory order performs an acquire operation, a store performs a
    // release operation, and read-modify-write performs both an acquire operation and a release
    // operation, plus a single total order exists in which all threads observe all modifications in
    // the same order

    std::atomic<bool> x = {false};
    std::atomic<bool> y = {false};
    std::atomic<int> z = {0};

    {
        std::jthread write_x([&]() {
            //
            x.store(true, std::memory_order_seq_cst);
        });
        std::jthread write_y([&]() {
            y.store(true, std::memory_order_seq_cst);
        });
        std::jthread read_x_then_y([&]() {
            while (!x.load(std::memory_order_seq_cst))
                ;
            if (y.load(std::memory_order_seq_cst)) {
                ++z;
            }
        });
        std::jthread read_y_then_x([&]() {
            while (!y.load(std::memory_order_seq_cst))
                ;
            if (x.load(std::memory_order_seq_cst)) {
                ++z;
            }
        });
    }
    assert(z.load() != 0);
}

int main() {
    test_atomic_relaxed();
    test_consume_release();
    test_acquire_release_1();
    test_acquire_release_2();
    test_seq_cst();
    return 0;
}
```