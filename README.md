ThreadPool
==========

A simple C++11 Thread Pool implementation.

Basic usage:
```c++
// create thread pool with 4 worker threads
ThreadPool pool(4);

// enqueue and store future
auto result = pool.enqueue([](int answer) { return answer; }, 42);

// get result from future
std::cout << result.get() << std::endl;

```
Build && Run
```c++
g++ -o example example.cpp -std=c++11 -pthread && ./example
```
代码的结构
```c++
 1.ThreadPool.h 线程类

 2.example.cpp 是对线程池类的调用

```
### ThreadPool.h 的分析
1. 类成员变变量
   - std::vector< std::thread > workers;  用来记录工作线程的数量以及知道那个线程正在工作
   -  std::queue< std::function<void()> > tasks; 用于存放任务的队列，用queue队列进行保存。任务类型为std::function<void()>。因为 std::function是通用多态函数封装器，也就是说本质上任务队列中存放的是一个个函数)
   - std::mutex queue_mutex;   访问队列的互斥锁，在任务插入队伍或者从队伍中取出任务保证线程的安全性
   -  std::condition_variable condition;  用于通知线程任务的条件变量，当有任务时候通知线程队列，当无任务的时候进入wait状态。
   -  bool stop; 标识线程池的状态
2. 类成员函数
    - ThreadPool(size_t);
    - ~ThreadPool();
    - auto enqueue(F&& f, Args&&... args) -> std::future<typename std::result_of<F(Args...)>::type>;
   将任务添加进线程队列中

    其中，std::future<typename std::result_of<F(Args...)>::type>; 是其中的返回函数)
    这一个函数如果没接触到相关的内容会比较吧难以理解
### 构造函数 ThreadPool(size_t)   
```c++
inline ThreadPool::ThreadPool(size_t threads)
      :   stop(false)
  {
      for(size_t i = 0;i<threads;++i)
          workers.emplace_back(
              [this]
              {
                  for(;;)
                  {
                      std::function<void()> task;
  ​
                      {
                          std::unique_lock<std::mutex> lock(this->queue_mutex);
                          this->condition.wait(lock,
                              [this]{ return this->stop || !this->tasks.empty(); });
                          if(this->stop && this->tasks.empty())
                              return;
                          task = std::move(this->tasks.front());
                          this->tasks.pop();
                      }
  ​
                      task();
                  }
              }
          );
  }

```
    
函数是一个内联函数，指定创建线程是数目，初始化线程池 stop为false ，表示线程池启动着
