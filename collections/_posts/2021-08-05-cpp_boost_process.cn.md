---
lang: cn
layout: post
title: "Boost::Process 进程管理"
date: 2021-08-05
author: "[码农森哥](https://twitter.com/senge26430360)"
---

## 概述
Boost.Process提供了一个灵活的C++ 进程管理框架。它允许C++ developer可以像Java和.Net程序developer那样管理进程。它还提供了管理当前执行进程上下文、创建子进程、用C++ 流和异步I/O进行通信的能力。
该库以完全透明的方式将所有进程管理的抽象细节呈现给使用者，且该库是跨平台的。

## 特点
### 进程管理
Boost.Process的长期目标是提供一个抽象于操作系统之上的，可以管理任何运行的进程的框架。由于提供这样的API比较困难，所以现在只专注于管理。Boost.Process的最重要的特征就是启动一个外部应用、控制它们并与它们交互。传统上讲，对于C和C++ 来说，就比较困难了，因为它们要启动新进程、执行外部程序、建立匿名管道来交互、等待进程结束、检查进程退出码等。更糟糕的是不同操作系统，相关的进程模块和API是不同的。所以，Boost.Process的出现就提供了便利条件。

### 输入输出重定向
一般来说一个应用启动了子进程，它们可能会通过传输数据来交流。这种进程间通信是文件句柄层面的，通常涉及stdin、stdout、stderr。如果操作系统支持，那么就需要可重定向的流。不过这对C++ 来说是很容易的。

### 不同操作模式
支持同步、异步、分离

### 管道管理
这样就可以实现一个进程的输出可以作为另一个进程的输入。

## 库的设计图
最重要的类就是Context和Process。Context提供了进程运行的上下文。pistream和postream是为了交互。父进程还可以等待子进程退出，并检查进程退出码。如果有例如包含管道的shell命令要执行，那么pipeline_entry就应运而生了，它可以实现前一个子进程的输出是下一个子进程的输入。

### 使用步骤

1、创建上下文Context
2、创建子进程，获得子进程对象
3、如果有重定向，可以访问到stdin、stdout、stderr
4、进程结束，检查进程退出码

## 教程
一个最简单的例子

```
#include <boost/filesystem.hpp>
#include <boost/process.hpp>
#include <string>
#include <vector>
 
namespace bp = ::boost::process;
 
bp::child start_child()
{
    std::string exec = "bjam";
    std::vector<std::string> args;
    args.push_back("bjam");
    args.push_back("--version");
 
    bp::context ctx;
    ctx.stdout_behavior = bp::capture_stream();
 
    return bp::launch(exec, args, ctx);
}
 
int main()
{
    bp::child c = start_child();
 
    bp::pistream &is = c.get_stdout();
    std::string line;
    while (std::getline(is, line))
        std::cout << line << std::endl;
    bp::status s = c.wait();
 
    return s.exited() ? s.exit_status() : EXIT_FAILURE;
}
```

下面再看一个异步的例子

```
#include <boost/filesystem.hpp>
#include <boost/asio.hpp>
#include <boost/process.hpp>
#include <boost/array.hpp>
#include <boost/bind.hpp>
#include <string>
#include <vector>
#include <iostream>
 
namespace bp = ::boost::process;
namespace ba = ::boost::asio;
 
ba::io_service io_service;
boost::array<char, 4096> buffer;
 
ba::posix::stream_descriptor in(io_service);
 
bp::child start_child()
{
    std::string exec = "bjam";
 
    std::vector<std::string> args;
    args.push_back("bjam");
    args.push_back("--version");
 
    bp::context ctx;
    ctx.stdout_behavior = bp::capture_stream();
    ctx.environment = bp::self::get_environment();
 
    return bp::launch(exec, args, ctx);
}
 
void end_read(const boost::system::error_code &ec, std::size_t bytes_transferred);
 
void begin_read()
{
    in.async_read_some(boost::asio::buffer(buffer),
        boost::bind(&end_read, ba::placeholders::error, ba::placeholders::bytes_transferred));
}
 
void end_read(const boost::system::error_code &ec, std::size_t bytes_transferred)
{
    if (!ec)
    {
        std::cout << std::string(buffer.data(), bytes_transferred) << std::flush;
        begin_read();
    }
}
 
int main()
{
    bp::child c = start_child();
 
    bp::pistream &is = c.get_stdout();
    in.assign(is.handle().release());
 
    begin_read();
    io_service.run();
 
    c.wait();
}
```

这个例子中用到了asio库，涉及到许多回调函数。关于异步(asio)暂时不做讲解，写这个例子是为了展示该库的异步功能。对异步感兴趣的同学可以看一下[《Boost.Asio C++ Network Programming》](https://www.boost.org/doc/libs/1_76_0/doc/html/process.html)


## 部分文件和类
### stream_behaviour.hpp文件


对于流的描述，可分为六种类型


| --- | ---| --- |
| 序号 | 流描述 | 含义 |
| --- | --- | --- |
| 1 |capture | 父子进程之间通过无名管道相互接收数据 |
| 2 |close 	 | 启动时关闭  |
| 3 |inherit  | 父子进程共用一个，也即继承  |
| 4 |redirect_to_stdout  | 主要用在stderr时，重定向到stdout  |
| 5 |silence  |输出重定向到/dev/null |
| 6 |posix_redirect  | 将输出重定向到指定的文件描符，是对redirect_to_stdout的扩展  |

以下是等价的
boost::process::child::get_stdin() <==> boost::process::posix_child::get_input(STDIN_FILENO)
boost::process::child::get_stdout() <==> boost::process::posix_child::get_output(STDOUT_FILENO)
boost::process::child::get_stderr() <==> boost::process::posix_child::get_output(STDERR_FILENO)


### 重定向的例子

```
#include <boost/process.hpp>
#include <boost/filesystem.hpp>
#include <string>
#include <vector>
#include <iostream>
#include <cstdlib>
#include <unistd.h>
 
namespace bp = ::boost::process;
 
bp::posix_child start_child()
{
    std::string exec = bp::find_executable_in_path("dbus-daemon");
 
    std::vector<std::string> args;
    args.push_back("dbus-daemon");
    args.push_back("--fork");
    args.push_back("--session");
    args.push_back("--print-address=3");
    args.push_back("--print-pid=4");
 
    bp::posix_context ctx;
    ctx.output_behavior.insert(bp::behavior_map::value_type(STDOUT_FILENO, bp::inherit_stream()));
    ctx.output_behavior.insert(bp::behavior_map::value_type(STDERR_FILENO, bp::inherit_stream()));
    ctx.output_behavior.insert(bp::behavior_map::value_type(3, bp::capture_stream()));
    ctx.output_behavior.insert(bp::behavior_map::value_type(4, bp::capture_stream()));
 
    return bp::posix_launch(exec, args, ctx);
}
 
int main()
{
    try
    {
        bp::posix_child c = start_child();
 
        std::string address;
        pid_t pid;
        c.get_output(3) >> address;
        c.get_output(4) >> pid;
 
        bp::status s = c.wait();
        if (s.exited())
        {
            if (s.exit_status() == EXIT_SUCCESS)
            {
                std::cout << "D-BUS daemon's address is: " << address << std::endl;
                std::cout << "D-BUS daemon's PID is: " << pid << std::endl;
            }
            else
                std::cout << "D-BUS daemon returned error condition: " << s.exit_status() << std::endl;
        }
        else
            std::cout << "D-BUS daemon terminated abnormally" << std::endl;
 
        return s.exited() ? s.exit_status() : EXIT_FAILURE;
    }
    catch (boost::filesystem::filesystem_error &ex)
    {
        std::cout << ex.what() << std::endl;
        return EXIT_FAILURE;
    }
}
```

### boost::process::context类

```
template <class Path>
class basic_context : public basic_work_directory_context<Path>, public environment_context
{
public:
    /**
     * Child's stdin behavior.
     */
    stream_behavior stdin_behavior;
 
    /**
     * Child's stdout behavior.
     */
    stream_behavior stdout_behavior;
 
    /**
     * Child's stderr behavior.
     */
    stream_behavior stderr_behavior;
};
 
typedef basic_context<std::string> context;
```

而basic_work_directory_context是用来设置工作目录的；environment_context实质上是个包装了boost::process::environment的类，boost::process::environment是一个map<string, string>,用以保存环境变量。

### boost::process::posix_context类

```
typedef std::map<int, stream_behavior> behavior_map;
 
template <class Path>
class posix_basic_context : public basic_work_directory_context<Path>, public environment_context
{
public:
    /**
     * Constructs a new POSIX-specific context.
     *
     * Constructs a new context. It is configured as follows:
     * * All communcation channels with the child process are closed.
     * * There are no channel mergings.
     * * The initial work directory of the child processes is set to the
     *   current working directory.
     * * The environment variables table is empty.
     * * The credentials are the same as those of the current process.
     */
    posix_basic_context()
        : uid(::getuid()),
        euid(::geteuid()),
        gid(::getgid()),
        egid(::getegid())
    {
    }
 
    /**
     * List of input streams that will be redirected.
     */
    behavior_map input_behavior;
 
    /**
     * List of output streams that will be redirected.
     */
    behavior_map output_behavior;
 
    /**
     * The user credentials.
     *
     * UID that specifies the user credentials to use to run the %child
     * process. Defaults to the current UID.
     */
    uid_t uid;
 
    /**
     * The effective user credentials.
     *
     * EUID that specifies the effective user credentials to use to run
     * the %child process. Defaults to the current EUID.
     */
    uid_t euid;
 
    /**
     * The group credentials.
     *
     * GID that specifies the group credentials to use to run the %child
     * process. Defaults to the current GID.
     */
    gid_t gid;
 
    /**
     * The effective group credentials.
     *
     * EGID that specifies the effective group credentials to use to run
     * the %child process. Defaults to the current EGID.
     */
    gid_t egid;
 
    /**
     * The chroot directory, if any.
     *
     * Specifies the directory in which the %child process is chrooted
     * before execution. Empty if this feature is not desired.
     */
    Path chroot;
};
 
/**
 * Default instantiation of posix_basic_context.
 */
typedef posix_basic_context<std::string> posix_context;
```

函数boost::process::self::get_environment()可以得到当前进程的环境变量。
我们可以对环境变量进行修改，如
boost::process::environment_context env;
env.insert(boost::process::environment::valuetype(“A”, “a”));

### 进程结束码类信息

```
class status
{
    friend class child;
 
public:
    /**
     * 进程是否正常退出
     */
    bool exited() const;
 
    /**
     * 进程返回值
     */
    int exit_status() const;
 
protected:
    status(int flags);
 
    ...
};
 
class posix_status : public status
{
public:
    posix_status(const status &s);
 
    /**
     * 进程是否因为信号终止
     */
    bool signaled() const;
 
    /**
     * 如果因为信号终止，那么是因为哪个信号终止的
     */
    int term_signal() const;
    /**
     * 是否core dump了
     */
    bool dumped_core() const;
 
    /**
     * 进程是否因为收到信号停止
     */
    bool stopped() const;
 
    /**
     * 如果进程因为收到信号停止，那么信号是哪个
     */
    int stop_signal() const;
}
```


### 进程类对象信息

```
class process
{
public:
    typedef pid_t id_type;
    process(id_type id);
 
    /**
     * Returns the process' identifier.
     */
    id_type get_id() const;
 
    /**
     * 强制终止一个进程，force为真则用SIGKILL杀死，否则用SIGTERM杀死
     */
    void terminate(bool force = false) const ;
private:
    ...
};
 
class child : public process
{
public:
    /**
     * 获得标准输出
     */
    postream &get_stdin() const;
 
    /**
     * 获得标准输入
     */
    pistream &get_stdout() const;
 
    /**
     * 获得标准错误输入
     */
    pistream &get_stderr() const;
 
    /**
     * 阻塞等待进程退出，返回状态码对象
     */
    status wait()；
 
    /**
     * 创建一个子进程对象
     */
    child(id_type id, detail::file_handle fhstdin, detail::file_handle fhstdout, detail::file_handle fhstderr, detail::file_handle fhprocess = detail::file_handle());
 
private:
    ...
};
 
class posix_child : public child
{
public:
    /**
     * 从指定描述符获得一个输出流
     */
    postream &get_input(int desc) const;
 
    /**
     * 从指定描述符获得一个输入流
     */
    pistream &get_output(int desc) const;
 
    /**
     *构造函数
     */
    posix_child(id_type id, detail::info_map &infoin, detail::info_map &infoout);
 
private:
    ...
};
```

### children类

```
children类实际上std::vector<child>。children的启动方式是一个输出流被链接到下一个子进程的输入流上。

#include <boost/process.hpp>
#include <string>
#include <vector>
#include <iostream>
#include <fstream>
#include <cstdlib>
 
namespace bp = ::boost::process;
 
bp::children start_children()
{
    bp::context ctxin;
    ctxin.stdin_behavior = bp::capture_stream();
 
    bp::context ctxout;
    ctxout.stdout_behavior = bp::inherit_stream();
    ctxout.stderr_behavior = bp::redirect_stream_to_stdout();
 
    std::string exec1 = bp::find_executable_in_path("cut");
    std::vector<std::string> args1;
    args1.push_back("cut");
    args1.push_back("-d ");
    args1.push_back("-f2-5");
 
    std::string exec2 = bp::find_executable_in_path("sed");
    std::vector<std::string> args2;
    args2.push_back("sed");
    args2.push_back("s,^,line: >>>,");
 
    std::string exec3 = bp::find_executable_in_path("sed");
    std::vector<std::string> args3;
    args3.push_back("sed");
    args3.push_back("s,$,<<<,");
 
    std::vector<bp::pipeline_entry> entries;
    entries.push_back(bp::pipeline_entry(exec1, args1, ctxin));
    entries.push_back(bp::pipeline_entry(exec2, args2, ctxout));
    entries.push_back(bp::pipeline_entry(exec3, args3, ctxout));
 
    return bp::launch_pipeline(entries);
}
 
int main(int argc, char *argv[])
{
    try
    {
        if (argc < 2)
        {
            std::cerr << "Please specify a file name" << std::endl;
            return EXIT_FAILURE;
        }
 
        std::ifstream file(argv[1]);
        if (!file)
        {
            std::cerr << "Cannot open file" << std::endl;
            return EXIT_FAILURE;
        }
 
        bp::children cs = start_children();
 
        bp::postream &os = cs.front().get_stdin();
        std::string line;
        while (std::getline(file, line))
            os << line << std::endl;
        os.close();
 
        bp::status s = bp::wait_children(cs);
 
        return s.exited() ? s.exit_status() : EXIT_FAILURE;
    }
    catch (boost::filesystem::filesystem_error &ex)
    {
        std::cout << ex.what() << std::endl;
        return EXIT_FAILURE;
    }
}
```


需要注意的是，wait_children出错时，返回第一个子进程的退出码，所有子进程都正常退出时，返回最后一个子进程的退出码。

master3中大量用到进程管理这个库。这个Boost.Process库可以在这里获得点这里。

