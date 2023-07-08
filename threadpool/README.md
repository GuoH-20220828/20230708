# workspace

**workspace**是基于C++11的轻量级异步执行框架，支持：通用任务异步执行、优先级任务调度、自适应动态线程池、高效静态线程池、异常处理机制等。

## 特点

- 轻量的：Header-Only & 代码量 <= 1000行 & 接口简单。
- 高效的：超轻量级任务支持异步顺序执行，提高了框架的并发性能。
- 灵活的：支持多种任务类型、动态线程调整、可通过workspace构建不同的池模型。
- 稳定的：利用`std::function`的小任务优化减少内存碎片、拥有良好的异步线程异常处理机制。
- 兼容性：纯C++11实现，跨平台，且兼容C++11以上版本。

## 主要模块

### **workbranch**

**workbranch**（工作分支）是动态线程池的抽象，内置了一条线程安全的**任务队列**用于同步任务。其管理的每一条异步工作线程被称为**worker**，负责从任务队列不断获取任务并执行。当你的任务带有返回值时，workbranch会返回一个std::future，否则返回void。
```C++
#include <workspace/workspace.h>

int main() 
{
    // 2 threads
    wsp::workbranch br(2);   //两个线程
    // return void
    br.submit([]{ std::cout<<"hello world"<<std::endl; });  
    // return std::future<int>
    auto result = br.submit([]{ return 2023; });  
    std::cout<<"Got "<<result.get()<<std::endl;   
    // wait for tasks done (timeout: 1000 milliseconds)
    br.wait_tasks(1000); 
}
```

由于返回一个std::future会带来一定的开销，如果你不需要返回值并且希望程序跑得更快，那么你的任务应该是`void()`类型的。
当你有一个任务并且你希望它能尽快被执行时，你可以指定该任务的类型为`urgent`，如下：
```C++
#include <workspace/workspace.h>

int main() 
{
    // 1 threads
    wsp::workbranch br;
    br.submit<wsp::task::nor>([]{ std::cout<<"task B done\n";}); // normal task 
    br.submit<wsp::task::urg>([]{ std::cout<<"task A done\n";}); // urgent task
    br.wait_tasks(); // wait for tasks done (timeout: no limit)
}
```
在这里我们不能保证`task A`一定会被先执行，因为当我们提交`task A`的时候，`task B`可能已经在执行中了。`urgent`标签可以让任务被插入到队列头部，但无法改变已经在执行的任务。
```shell
jack@xxx:~/workspace/example/build$ ./e2
task A done
task B done
```



假如你有几个轻量异步任务，执行他们只需要**非常短暂**的时间。同时，按照**顺序执行**它们对你来说没有影响。那么你可以把任务类型指定为`sequence`，以便提交一个**任务序列**。这个任务序列会被单个线程顺序执行：
任务序列会被打包成一个较大的任务，以此来减轻框架同步任务的负担，提高整体的并发性能。
```c++
#include <workspace/workspace.h>
int main() 
{
    wsp::workbranch br;
    // sequence tasks
    br.submit<wsp::task::seq>([]{std::cout<<"task 1 done\n";},
                              []{std::cout<<"task 2 done\n";},
                              []{std::cout<<"task 3 done\n";},
                              []{std::cout<<"task 4 done\n";});
    // wait for tasks done (timeout: no limit)
    br.wait_tasks();
}
```


当任务中抛出了一个异常，workbranch有两种处理方式：A-将其捕获并输出到终端 B-将其捕获并通过std::future传递到主线程。第二种需要你提交一个**带返回值**的任务。
```C++
#include <workspace/workspace.h>
// self-defined
class excep: public std::exception 
{
    const char* err;
public:
    excep(const char* err): err(err) {}
    const char* what() const noexcept override {
        return err;
    }
}; 
int main() 
{
    wsp::workbranch wbr;
    wbr.submit([]{ throw std::logic_error("A logic error"); });     // log error
    wbr.submit([]{ throw std::runtime_error("A runtime error"); }); // log error
    wbr.submit([]{ throw excep("XXXX");});                          // log error

    auto future1 =  wbr.submit([]{ throw std::bad_alloc(); return 1; }); // catch error
    auto future2 =  wbr.submit([]{ throw excep("YYYY"); return 2; });    // catch error
    try {
        future1.get();
    } catch (std::exception& e) {
        std::cerr<<"Caught error: "<<e.what()<<std::endl;
    }
    try {
        future2.get();
    } catch (std::exception& e) {
        std::cerr<<"Caught error: "<<e.what()<<std::endl;
    }
}


---

### **supervisor**

supervisor是异步管理者线程的抽象，负责监控workbranch的负载情况并进行动态调整。它允许你在每一次调控workbranch之后执行一个小任务，你可以用来**写日志**或者做一些其它调控等。
每一个supervisor可以管理多个workbranch。此时workbranch之间共享supervisor的所有设定。
```c++
#include <workspace/workspace.h>
int main() 
{
    wsp::workbranch br1(2);
    wsp::workbranch br2(2);

    // 2 <= thread number <= 4 
    // time interval: 1000 ms 
    wsp::supervisor sp(2, 4, 1000);

    sp.set_tick_cb([&br1, &br2]{
        auto now = std::chrono::system_clock::now();
        std::time_t timestamp = std::chrono::system_clock::to_time_t(now);
        std::tm local_time = *std::localtime(&timestamp);
        static char buffer[40];
        std::strftime(buffer, sizeof(buffer), "%Y-%m-%d %H:%M:%S", &local_time);
        std::cout<<"["<<buffer<<"] "<<"br1: [workers] "<<br1.num_workers()<<" | [blocking-tasks] "<<br1.num_tasks()<<'\n';
        std::cout<<"["<<buffer<<"] "<<"br2: [workers] "<<br2.num_workers()<<" | [blocking-tasks] "<<br2.num_tasks()<<'\n';
    });

    sp.supervise(br1);  // start supervising
    sp.supervise(br2);  // start supervising

    for (int i = 0; i < 1000; ++i) {
        br1.submit([]{std::this_thread::sleep_for(std::chrono::milliseconds(10));});
        br2.submit([]{std::this_thread::sleep_for(std::chrono::milliseconds(20));});
    }

    br1.wait_tasks();
    br2.wait_tasks();
}
```
---

### **workspace**
workspace是一个**托管器**/**任务分发器**，你可以将workbranch和supervisor托管给它，并用workspace分配的**组件专属ID**来访问它们。将组件托管至workspace至少有以下几点好处：
- 堆内存正确释放：workspace在内部用unique指针来管理组件，确保没有内存泄漏
- 分支间任务负载均衡：workspace支持任务分发，在workbranch之间实现了简单高效的**负载均衡**。
- 避免空悬指针问题：当workbranch先于supervisor析构会造成**空悬指针**的问题，使用workspace可以避免这种情况
- 更低的框架开销：workspace的任务分发机制能减少与工作线程的竞争，提高性能（见下Benchmark）。
我们可以通过workspace自带的任务分发机制来异步执行任务（调用`submit`）。
```C++
#include <workspace/workspace.h>
int main() 
{
    wsp::workspace spc;
    auto bid1 = spc.attach(new wsp::workbranch);
    auto bid2 = spc.attach(new wsp::workbranch);
    auto sid1 = spc.attach(new wsp::supervisor(2, 4));
    auto sid2 = spc.attach(new wsp::supervisor(2, 4));
    spc[sid1].supervise(spc[bid1]);  // start supervising
    spc[sid2].supervise(spc[bid2]);  // start supervising

    // Automatic assignment
    spc.submit([]{std::cout<<std::this_thread::get_id()<<" executed task"<<std::endl;});
    spc.submit([]{std::cout<<std::this_thread::get_id()<<" executed task"<<std::endl;});

    spc.for_each([](wsp::workbranch& each){each.wait_tasks();});
}
```
当我们需要等待任务执行完毕的时候，我们可以调用`for_each`+`wait_tasks`，并为每一个workbranch指定等待时间，单位是毫秒。


## 辅助模块
### futures 
wsp::futures是一个std::future收集器(collector)，可以缓存同类型的std::future，并进行批量操作。一个简单的操作如下:
这里`futures.get()`返回的是一个`std::vector<int>`，里面保存了所有任务的返回值。
```C++
#include <workspace/workspace.h>
int main() 
{
    wsp::futures<int> futures;
    wsp::workspace spc;
    spc.attach(new wsp::workbranch("br", 2));
    
    futures.add_back(spc.submit([]{return 1;}));
    futures.add_back(spc.submit([]{return 2;}));

    futures.wait();
    auto res = futures.get();
    for (auto& each: res) 
    {
        std::cout<<"got "<<each<<std::endl;
    }
}
```

## 空跑测试

测试原理：通过快速提交大量的空任务以考察框架同步任务的开销。
测试环境：Ubuntu 9.4.0-1ubuntu1~20.04.1 : 4核 
<**测试1**><br> 在测试1中我们调用了`submit<wsp::task::seq>`，每次打包10个空任务并提交到**workbranch**中执行。结果如下：
```C++
#include <workspace/workspace.h>
#include "timewait.h"
int main(int argn, char** argvs) 
{
    int task_nums, thread_nums;
    if (argn == 3) 
    {
        thread_nums = atoi(argvs[1]);
        task_nums = atoi(argvs[2]);
    } 
    else 
    {
        fprintf(stderr, "Invalid parameter! usage: [threads + tasks]\n");
        return -1;
    }  
    wsp::workbranch wb(thread_nums);
    auto time_cost = timewait([&]
    {
        auto task = []{/* empty task */};
        for (int i = 0; i < task_nums/10; ++i) 
        {
            wb.submit<wsp::task::seq>(task, task, task, task, task, task, task, task, task, task);
        }
        wb.wait_tasks();
    });
    std::cout<<"threads: "<<std::left<<std::setw(2)<<thread_nums<<" |  tasks: "<<task_nums<<"  |  time-cost: "<<time_cost<<" (s)"<<std::endl;
}
```
```
threads: 1  |  tasks: 100000000  |  time-cost: 2.68801 (s)
threads: 2  |  tasks: 100000000  |  time-cost: 3.53964 (s)
threads: 3  |  tasks: 100000000  |  time-cost: 3.99903 (s)
threads: 4  |  tasks: 100000000  |  time-cost: 5.26045 (s)
threads: 5  |  tasks: 100000000  |  time-cost: 6.65157 (s)
threads: 6  |  tasks: 100000000  |  time-cost: 8.40907 (s)
threads: 7  |  tasks: 100000000  |  time-cost: 10.5967 (s)
threads: 8  |  tasks: 100000000  |  time-cost: 13.2523 (s)
```

<**测试2**> 在测试2中我们同样将10个任务打成一包，但是是将任务提交到**workspace**中，利用workspace进行任务分发，且在workspace托管的workbranch只拥有 **1条** 线程。结果如下
```C++
#include <workspace/workspace.h>
#include "timewait.h"
int main(int argn, char** argvs) 
{
    int task_nums, thread_nums;
    if (argn == 3) 
    {
        thread_nums = atoi(argvs[1]);
        task_nums = atoi(argvs[2]);
    } 
    else 
    {
        fprintf(stderr, "Invalid parameter! usage: [threads + tasks]\n");
        return -1;
    } 
    wsp::workspace spc;
    for (int i = 0; i < thread_nums; ++i) 
    {
        spc.attach(new wsp::workbranch());
    }
    auto time_cost = timewait([&]{
        auto task = []{/* empty task */};
        for (int i = 0; i < task_nums/10; ++i) 
        {
            spc.submit<wsp::task::seq>(task, task, task, task, task, task, task, task, task, task);
        }
        spc.for_each([](wsp::workbranch& each){each.wait_tasks();});
    });
    std::cout<<"threads: "<<std::left<<std::setw(2)<<thread_nums<<" |  tasks: "<<task_nums<<"  |  time-cost: "<<time_cost<<" (s)"<<std::endl;
}
```
```
threads: 1  |  tasks: 100000000  |  time-cost: 4.38221 (s)
threads: 2  |  tasks: 100000000  |  time-cost: 4.01103 (s)
threads: 3  |  tasks: 100000000  |  time-cost: 3.6797 (s)
threads: 4  |  tasks: 100000000  |  time-cost: 3.39314 (s)
threads: 5  |  tasks: 100000000  |  time-cost: 3.03324 (s)
threads: 6  |  tasks: 100000000  |  time-cost: 3.16079 (s)
threads: 7  |  tasks: 100000000  |  time-cost: 3.04612 (s)
threads: 8  |  tasks: 100000000  |  time-cost: 3.11893 (s)
```

<**测试3**> 在测试3中我们同样将10个任务打成一包，并且将任务提交到**workspace**中，但是workspace管理的每个**workbranch**中都拥有 **2条** 线程。结果如下：
```C++
#include <workspace/workspace.h>
#include "timewait.h"

int main(int argn, char** argvs)
{
    int task_nums, thread_nums;
    if (argn == 3) 
    {
        thread_nums = atoi(argvs[1]);
        task_nums = atoi(argvs[2]);
    } 
    else 
    {
        fprintf(stderr, "Invalid parameter! usage: [threads + tasks]\n");
        return -1;
    } 
    wsp::workspace spc;
    for (int i = 0; i < thread_nums/2; ++i) 
    {
        spc.attach(new wsp::workbranch(2));
    }
    auto time_cost = timewait([&]{
        auto task = []{/* empty task */};
        for (int i = 0; i < task_nums/10; ++i) 
        {
            spc.submit<wsp::task::seq>(task, task, task, task, task, task, task, task, task, task);
        }
        spc.for_each([](wsp::workbranch& each){each.wait_tasks();});
    });
    std::cout<<"threads: "<<std::left<<std::setw(2)<<thread_nums<<" |  tasks: "<<task_nums<<"  |  time-cost: "<<time_cost<<" (s)"<<std::endl;
}
```
```
threads: 2  |  tasks: 100000000  |  time-cost: 4.53911 (s)
threads: 4  |  tasks: 100000000  |  time-cost: 7.0178 (s)
threads: 6  |  tasks: 100000000  |  time-cost: 6.00101 (s)
threads: 8  |  tasks: 100000000  |  time-cost: 5.97501 (s)
threads: 10 |  tasks: 100000000  |  time-cost: 5.63834 (s)
threads: 12 |  tasks: 100000000  |  time-cost: 5.17316 (s)
```

**总结**：利用workspace进行任务分发，且**workbranch**线程数为1的情况下，整个任务同步框架是静态的，任务同步开销最小。当**workbranch**内的线程数越多，面对大量空任务时对任务队列的竞争越激烈，框架开销越大。


## 如何使用
```shell
# 项目代码与workspace同级（Linux）
g++ -I workspace/include xxx.cc -lpthread && ./a.out
```

```shell
# 在"workspace/example"中
cmake -B build 
cd build
make
./e1
```

```shell
# 在"workspace/"中
cmake -B build 
cd build
sudo make install
```

## 注意事项

#### 
1. 不要在任务中操纵组件，如：`submit([&br]{br.wait_tasks();});` 会阻塞线程 
2. 不要在回调中操纵组件，如：`set_tick_cb([&sp]{sp.suspend();});` 
3. 不要让workbranch先于supervisor析构（空悬指针问题）。

#### 接口安全性

|组件接口|是否线程安全|
| :-- | :--: |
|workspace|否|
|workbranch|是|
|supervisor|是|
|futures|否|

#### 时间单位
workspace有关时间的接口单位都是 -> 毫秒（ms）

#### 参考书目
《C++并发编程》 

#### 联系我
邮箱: 15091721302@163.com
