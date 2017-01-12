title:  spdlog源码学习
tag: c++ 开源代码
---
[spdlog](https://github.com/gabime/spdlog)是一个用c++11实现的高性能日志库。
接入方便，功能丰富，代码可读性较高。

---
## Features
* Very fast - performance is the primary goal 
* Headers only, just copy and use.
* Feature rich  using the excellent [fmt](https://github.com/fmtlib/fmt) library.
* Extremely fast asynchronous mode (optional) - using lockfree queues and other tricks to reach millions of calls/sec.
* [Custom](https://github.com/gabime/spdlog/wiki/3.-Custom-formatting) formatting.
* Multi/Single threaded loggers.
* Various log targets:
    * Rotating log files.
    * Daily log files.
    * Console logging (colors supported).
    * syslog.
    * Windows debugger 
    * Easily extendable with custom log targets  (just implement a single function in the sink interface).
* Severity based filtering - threshold levels can be modified in runtime as well as in compile time.


---

## 核心设计
spdlog中定义了一系列Sink类，作为实际上把log输出到指定目标的对象，每一个Sink唯一对应一个log的输出目标（如console、文件、db）.
每一个logger包含一个sink列表，每当上层调用logger的log方法写日志时，logger就会调用sink列表中的每一个sink的 `sink(log_msg)` 函数，真正往目标中写日志。

### 实现Rotating file写日志

当前正在写日志的文件名一直都是log.txt。
当log.txt写不下时，把它重命名为log1.txt,再重新打开一个log.txt文件。
多个文件循环也依次类推。

    // Rotate files:
    // log.txt -> log.1.txt
    // log.1.txt -> log2.txt
    // log.2.txt -> log3.txt
    // log.3.txt -> delete


```cpp
/*
 * Rotating file sink based on size
 */
template<class Mutex>
class rotating_file_sink : public base_sink < Mutex >
{
public:
    rotating_file_sink(const filename_t &base_filename, const filename_t &extension,
                       std::size_t max_size, std::size_t max_files                       ) :
    {
        _file_helper.open(calc_filename(_base_filename, 0, _extension));
        _current_size = _file_helper.size(); //expensive. called only once
    }

protected:
    void _sink_it(const details::log_msg& msg) override
    {
        _current_size += msg.formatted.size();
        if (_current_size > _max_size)
        {
            _rotate();
            _current_size = msg.formatted.size();
        }
        _file_helper.write(msg);
    }

private:
    static filename_t calc_filename(const filename_t& filename, std::size_t index, const filename_t& extension)
    {
    //...
        return w.str();
    }

    void _rotate()
    {
        using details::os::filename_to_str;
        _file_helper.close();
        for (auto i = _max_files; i > 0; --i)
        {
            filename_t src = calc_filename(_base_filename, i - 1, _extension);
            filename_t target = calc_filename(_base_filename, i, _extension);

            if (details::file_helper::file_exists(target))
            {
                details::os::remove(target);
            }
            if (details::file_helper::file_exists(src) && details::os::rename(src, target))
            {
				//...
            }
        }
        _file_helper.reopen(true);
    }
    filename_t _base_filename;
    filename_t _extension;
    std::size_t _max_size;
    std::size_t _max_files;
    std::size_t _current_size;
    details::file_helper _file_helper;
};
```
---
### 实现异步写日志
所有异步写日志的请求都会被push到一个固定大小的队列中。
有一个工作线程会不停的从队列中pop一条日志消息，并最终写到日志中。

调用方：
```cpp
void async_example()
{
    size_t q_size = 4096; //queue size must be power of 2
    spdlog::set_async_mode(q_size);
    auto async_file = spd::daily_logger_st("async_file_logger", "logs/async_log.txt");

    for (int i = 0; i < 100; ++i)
        async_file->info("Async message #{}", i);
}
```
具体实现：
开启一个工作线程，在async_log初始时就开始工作了。并且在析构是被关闭
```cpp
// Async Logger implementation
// Use an async_sink (queue per logger) to perform the logging in a worker thread

//一个工作线程，在async_log初始时就开始工作了。并且在析构是被关闭
// worker thread
std::thread _worker_thread;
_worker_thread(&async_log_helper::worker_loop, this)
inline void spdlog::details::async_log_helper::worker_loop()
{
    try
    {
        if (_worker_warmup_cb) _worker_warmup_cb();
        auto last_pop = details::os::now();
        auto last_flush = last_pop;
        while(process_next_msg(last_pop, last_flush));
        if (_worker_teardown_cb) _worker_teardown_cb();
    }
    catch (const std::exception &ex)
    {
        _err_handler(ex.what());
    }
    catch (...)
    {
        _err_handler("Unknown exception");
    }
}
// Send to the worker thread termination message(level=off)
// and wait for it to finish gracefully
inline spdlog::details::async_log_helper::~async_log_helper()
{
    try
    {
        push_msg(async_msg(async_msg_type::terminate));
        _worker_thread.join();
    }
    catch (...) // don't crash in destructor
    {}
}
```
从队列中取消息，如果是写日志的消息，则调用sink写日志。
```cpp
//从队列中取消息，如果是写日志的消息，则调用sink写日志。
// process next message in the queue
// return true if this thread should still be active (while no terminate msg was received)
inline bool spdlog::details::async_log_helper::process_next_msg(log_clock::time_point& last_pop, log_clock::time_point& last_flush)
{
    async_msg incoming_async_msg;

    if (_q.dequeue(incoming_async_msg))
    {
        last_pop = details::os::now();
        switch (incoming_async_msg.msg_type)
        {
        case async_msg_type::flush:
            _flush_requested = true;
            break;

        case async_msg_type::terminate:
            _flush_requested = true;
            _terminate_requested = true;
            break;

        default:
            log_msg incoming_log_msg;
            incoming_async_msg.fill_log_msg(incoming_log_msg);
            _formatter->format(incoming_log_msg);
            for (auto &s : _sinks)
            {
                if(s->should_log( incoming_log_msg.level))
                {
                    s->log(incoming_log_msg);
                }
            }
        }
        return true;
    }

    // Handle empty queue..
    // This is the only place where the queue can terminate or flush to avoid losing messages already in the queue
    else
    {
        auto now = details::os::now();
        handle_flush_interval(now, last_flush);
        sleep_or_yield(now, last_pop);
        return !_terminate_requested;
    }
}
```
调用写日志，往队列里塞一条写日志消息。
```cpp
template<typename T>
inline void spdlog::logger::log(level::level_enum lvl, const T& msg)
{
    if (!should_log(lvl)) return;
    try
    {
        details::log_msg log_msg(&_name, lvl);
        log_msg.raw << msg;
        _sink_it(log_msg);
    }
    catch (const std::exception &ex)
    {
        _err_handler(ex.what());
    }
    catch (...)
    {
        _err_handler("Unknown exception");
    }
}

inline void spdlog::async_logger::_sink_it(details::log_msg& msg)
{
    try
    {
        _async_log_helper->log(msg);
        if (_should_flush_on(msg))
            _async_log_helper->flush(false); // do async flush
    }
    catch (const std::exception &ex)
    {
        _err_handler(ex.what());
    }
    catch (...)
    {
        _err_handler("Unknown exception");
    }
}

// 往队列里塞一条日志进去，准备被写
//Try to push and block until succeeded (if the policy is not to discard when the queue is full)
inline void spdlog::details::async_log_helper::log(const details::log_msg& msg)
{
    push_msg(async_msg(msg));
}

inline void spdlog::details::async_log_helper::push_msg(details::async_log_helper::async_msg&& new_msg)
{
    if (!_q.enqueue(std::move(new_msg)) && _overflow_policy != async_overflow_policy::discard_log_msg)
    {
        auto last_op_time = details::os::now();
        auto now = last_op_time;
        do
        {
            now = details::os::now();
            sleep_or_yield(now, last_op_time);
        }
        while (!_q.enqueue(std::move(new_msg)));
    }
}
```

根据时间间隔让线程等待
```cpp
// spin, yield or sleep. use the time passed since last message as a hint
inline void spdlog::details::async_log_helper::sleep_or_yield(const spdlog::log_clock::time_point& now, const spdlog::log_clock::time_point& last_op_time)
{
    using namespace std::this_thread;
    using std::chrono::milliseconds;
    using std::chrono::microseconds;

    auto time_since_op = now - last_op_time;

    // spin upto 50 micros
    if (time_since_op <= microseconds(50))
        return;

    // yield upto 150 micros
    if (time_since_op <= microseconds(100))
        return std::this_thread::yield();

    // sleep for 20 ms upto 200 ms
    if (time_since_op <= milliseconds(200))
        return sleep_for(milliseconds(20));

    // sleep for 200 ms
    return sleep_for(milliseconds(200));
}
```
