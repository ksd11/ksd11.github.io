---
layout: post
title: syalr-config模块
author: Qiang Liu
# date: 2024-04-03
toc:  true
tag: [sylar]
---
对sylar的配置模块进行说明



## 使用

### 代码

```c++
// ------  1. 定义配置 ------
// 1.1 通过代码定义
sylar::ConfigVar<int>::ptr g_int_value_config =
    sylar::Config::Lookup("system.port", (int)8080, "system port");
sylar::ConfigVar<std::vector<int> >::ptr g_int_vec_value_config =
    sylar::Config::Lookup("system.int_vec", std::vector<int>{1,2}, "system int vec");

// 1.2 从配置文件读取
YAML::Node root = YAML::LoadFile("./bin/conf/log.yml");
sylar::Config::LoadFromYaml(root);
/*
上述的配置通过yaml文件可定义如下：
system:
	port: 8080
	int_vec:
		- 1
		- 2
*/

// ----- 2. 获取配置 ----
// 通过getValue获取值
SYLAR_LOG_INFO(SYLAR_LOG_ROOT()) << g_int_value_config->getValue();

// 可以通过LookUp直接获取ConfigVar对象
sylar::ConfigVar<int>::ptr g_int_value_config =
    sylar::Config::Lookup("system.port", (int)8080, "system port");


// ----- 3. 监听器 -----
// 采用观察者模式，在配置发生改变时会通知所有监听者
g_int_value_config->addListener([](const int& old_value, const int& new_value){
        SYLAR_LOG_INFO(SYLAR_LOG_ROOT()) << "old_value=" << old_value.toString()
                << " new_value=" << new_value.toString();
});


```





## 实现

config模块的类图及其之间的关系如下图所示

![image-20240403152744507](/assets/image-20240403152744507.png)

### 一条log是如何输出的？

1. 用户通过loggerManager获取一个logger，可根据log.h提供的宏`SYLAR_LOG_ROOT`快速获取一个根logger，此宏也是调用了`sylar::LoggerMgr::GetInstance()->getRoot()`方法获取

2.  logger有两个重要的成员，formatter和appender，其中formatter指定日志的输出格式（输出什么）；而appender指定日志输出的位置（往哪里输出）。在Logger构造的时候，会初始化formatter，而appender通过Logger的成员方法addAppender添加。

3. 在初始化formatter的时候，会调用`LogFormatter::init`，对用户传输的格式化字符串解析，如(%t:线程id，%f，文件名...)，然后创建相应的ForamItem对象加入formatter的vector成员`m_items`。至此logger和其成员初始化完毕，等待用户调用Logger->log()进行输出

4. logEvent封装了发生的事件（如：什么时候发生的，在哪个文件哪一行打印，输出日志的线程id，协程Id，消息等）。当用户使用`SYLAR_LOG_DEBUG(logger)`打印debug日志或其他不同宏打印不同等级日志时，会封装一个LogEvent对象，并返回其ostream类型的`m_ss`成员接受msg（如cout<< msg中的msg）。然后，在析构时调用logEvent保存的Logger的log方法将日志输出。如下给出了宏SYLAR_LOG_DEBUG的宏定义，它调用了SYLAR_LOG_LEVEL，SYLAR_LOG_LEVEL根据上下文信息封装了logEvent对象，在if退出时，调用LogEventWrapper的析构，在析构函数中调用其成员logger的log方法将日志输出

   

```c
#define SYLAR_LOG_LEVEL(logger, level) \
    if(logger->getLevel() <= level) \
        sylar::LogEventWrap(sylar::LogEvent::ptr(new sylar::LogEvent(logger, level, \
                        __FILE__, __LINE__, 0, sylar::GetThreadId(),\
                sylar::GetFiberId(), time(0), sylar::Thread::GetName()))).getSS()

#define SYLAR_LOG_DEBUG(logger) SYLAR_LOG_LEVEL(logger, sylar::LogLevel::DEBUG)
...
```



### 各个类功能说明

- LogLevel：定义了日志级别，如debug=1，info=2，warn=3, error=4, fatal=5。其中数字越大级别越高，当用户指定级别为error时，只有大于等于error级别的日志才会输出。还提供了枚举类型转String的相关方法
- LogEvent/LogEventWrapper：LogEvent对事件进行封装，如日志是在那个线程，哪个协程，哪个文件哪一行，什么时间等信息进行封装，LogEventWrapper是对LogEvent的进一层封装，比如LogEventWrapper析构时，会调用LogEvent的logger成员的log方法对日志进行输出
- LoggerManager：LoggerManager用单例模式管理起来，其对系统的所有logger进行管理
- Logger：logger协调logEvent（输出什么）, logAppender（往哪输出），logFormatter（输出格式）
- LogAppender：一个抽象类，定义了日志往哪输出，目前实现了StdoutLogAppender，FileLogAppender，分别往标准输出、文件输出
- LogFormatter：定义了日志输出的格式，包含多个LogFormatItem，定义了输出什么信息
- LogFormatItem：抽象类，有StringFormatItem，DateTimeFormatItem等实现，定义了各个item具体输出实现

