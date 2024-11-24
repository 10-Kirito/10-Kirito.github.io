+++
title = 'SpuriousWakeup'
date = 2024-11-24T12:06:31+08:00
tag = ['C++']
categories = ['C++']
summary = 'Spurious Wakeup - 什么是虚假唤醒？如何避免虚假唤醒？'
draft = false
+++

# 1. Reference Links

- ##### [Why does pthread_cond_wait have spurious wakeups?](https://stackoverflow.com/questions/8594591/why-does-pthread-cond-wait-have-spurious-wakeups)

# 2. Spurious Wakeup

虚假唤醒是多线程编程当中的一个现象，值得是线程在等待某一个变量的时候，即使没有其他的线程显式通知（例如通过`notify`或者`notify_all`调用），线程却意外地从等待状态当中被唤醒。

其实简单的理解，运行当中的线程由于某些资源被占用或者不足陷入阻塞状态，相关资源准备之后通知该线程，但是会出现通知该线程成功之后，资源却又被别的线程占用的情况。最终会导致一种虚假唤醒的场景，线程被唤醒，但是资源缺又被别的线程占据。

下面的程序就是一个实际的例子：

```C++
#include <iostream>
#include <memory>
#include <mutex>
#include <queue>
#include <thread>
#include <condition_variable>

std::queue<int> buffer;

std::mutex mtx;
std::condition_variable cv;

void producer(int id) {
  int value = 0;
  while (value < 10) {
    {
      std::lock_guard<std::mutex> lock(mtx);
      buffer.push(value);
    }
    std::cout << "Producer" << id << " produced: " << value << std::endl;
    value++;
    cv.notify_all();
    
    if (value == 10) {
      std::cout << "Producer" << id << " produced all values" << std::endl;
      break;
    }
  }
}

void consumer(int id) {
  while (true) {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock);
    int value = buffer.front();
    buffer.pop();
    std::cout << "Consumer " << id << " consumed: " << value << std::endl;
    cv.notify_all();

    if (value == 9) {
      std::cout << "Consumer consumed all values" << std::endl;
      break;
    }
  }
}

int main() {
  std::thread producer1(producer, 1);
  std::thread consumer1(consumer, 1);
  std::thread consumer2(consumer, 2);

  producer1.join();
  consumer1.detach();
  consumer2.detach();
  return 0;
}
```

运行上面这段程序，会出现各种意外的情况：

```shell
Producer1 produced: 0
Producer1 produced: 1
Consumer 1 consumed: 0
Consumer 2 consumed: 1
Consumer 1 consumed: 0
Producer1 produced: 2
Producer1 produced: 3
Consumer 1 consumed: 3
Consumer 2 consumed: 0
Consumer 1 consumed: 0
Consumer 2 consumed: 0
Producer1 produced: 4
Producer1 produced: 5
Consumer 2 consumed: 0
Producer1 produced: 6
Consumer 1 consumed: 0
Consumer 2 consumed: 0
Producer1 produced: 7
Producer1 produced: 8
Consumer 2 consumed: 0
Consumer 1 consumed: 0
Producer1 produced: 9
Producer1 produced all values
Consumer 1 consumed: 0
Consumer 2 consumed: 0
Consumer 1 consumed: 0
```

在某些情况下，我们可能会发现一些资源已经被消费者消费，但随后又被其他消费者重新消费。这种情况通常是由于**假唤醒现象**的存在，导致消费者线程在不合适的时机被唤醒，从而引发了预期之外的行为。

为了解决因假唤醒引发的问题，通常的做法是在线程被唤醒时对相关条件进行检查，只有在满足特定条件时，线程才会继续执行。如果条件不满足，线程将继续处于阻塞状态，直到条件得到满足为止。

```C++
void consumer(int id) {
  while (true) {
    std::unique_lock<std::mutex> lock(mtx);
    while (buffer.empty()) {
      cv.wait(lock);
    }
    int value = buffer.front();
    buffer.pop();
    std::cout << "Consumer " << id << " consumed: " << value << std::endl;
    cv.notify_all();

    if (value == 9) {
      std::cout << "Consumer consumed all values" << std::endl;
      break;
    }
  }
}
```

在消费者线程 (`consumer`) 被唤醒时，我们只需要检查资源是否真的可以使用，而不是直接响应假唤醒现象。如果是虚假的唤醒，消费者线程会重新进入阻塞状态，直到资源条件真正满足。同样，对于生产者线程，如果有多个生产者，也可能会遇到类似的问题。因此，我们只需确保每个生产者在被唤醒时检查是否需要进行生产操作，以避免不必要的生产。

# 3. 其他的解决办法

为了解决上述提到的假唤醒现象，C++11 及之后的标准在 `std::condition_variable::wait` 方法中增加了一个重载版本，该方法接受一个谓词（即返回布尔值的函数或 lambda 表达式）作为第二个参数。这样，每次线程被唤醒时，都会通过该谓词判断是否是由于假唤醒导致的，从而决定是否继续阻塞或继续执行。

```c++
#include <condition_variable>
#include <iostream>
#include <memory>
#include <mutex>
#include <queue>
#include <thread>

std::queue<int> buffer;

std::mutex mtx;
std::condition_variable cv;

void producer(int id) {
  int value = 0;
  while (value < 10) {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return buffer.size() < 10; });

    buffer.push(value);
    std::cout << "Producer" << id << " produced: " << value << std::endl;
    value++;
    cv.notify_all();

    if (value == 10) {
      std::cout << "Producer" << id << " produced all values" << std::endl;
      break;
    }
  }
}

void consumer(int id) {
  int value = 0;
  while (true) {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, []{ return !buffer.empty(); });

    value = buffer.front();
    buffer.pop();

    std::cout << "Consumer " << id << " consumed: " << value << std::endl;
    cv.notify_all();

    if (value == 9) {
      std::cout << "Consumer consumed all values" << std::endl;
      break;
    }
  }
}

int main() {
  std::thread producer1(producer, 1);
  std::thread consumer1(consumer, 1);
  std::thread consumer2(consumer, 2);

  producer1.join();
  consumer1.detach();
  consumer2.detach();
  return 0;
}
```

执行结果：

```shell
Producer1 produced: 0
Producer1 produced: 1
Producer1 produced: 2
Producer1 produced: 3
Producer1 produced: 4
Producer1 produced: 5
Producer1 produced: 6
Producer1 produced: 7
Producer1 produced: 8
Producer1 produced: 9
Producer1 produced all values
Consumer 2 consumed: 0
Consumer 2 consumed: 1
Consumer 2 consumed: 2
Consumer 2 consumed: 3
Consumer 2 consumed: 4
Consumer 2 consumed: 5
Consumer 2 consumed: 6
Consumer 2 consumed: 7
Consumer 2 consumed: 8
Consumer 2 consumed: 9
Consumer consumed all values
```

