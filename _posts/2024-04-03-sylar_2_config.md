---
layout: post
title: syalr-2-config
author: Qiang Liu
# date: 2024-04-03
toc:  true
tag: [sylar]
---
对sylar的配置模块进行说明



## 使用

配置模块比较简单，其对整个系统的相关配置进行管理，比如：日志输出格式，监听的端口号等。整个配置模块实现较简单，有个config类管理所有的configVar配置项，config类全部都是静态方法，统一提供加载，查找配置项的接口。



下面先对配置模块如何使用进行说明

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

![image-20240404101725138](\assets\sylar\image-20240404101725138.png)

### 各个类功能说明

- ConfigVarBase：配置项的抽象基类，为所有配置项提供通用接口，如获取配置项名称`getName()`，获取配置项描述`getDescription()`等
- ConfigVar：配置项的具体实现，是一个模板类，因为不同的配置有不同的值类型，提供`getValue()`，`setValue()`等操作值的方法，还有和Listener相关的一些方法，可以添加一个Listener，在值发生变化时进行通知。最后，值得注意的是，用户要提供此value类型和String相互转换的方法。
- Config：管理所有的配置项，此类所有方法均为静态方法，用户可通过Config类快速创建，或是从yaml文件中加载配置。



### LogDefine例子

此节以LogDefine配置的加载和监听过程，对配置模块的使用进行说明。下面的代码给出了配置项相关的一下代码

```c++
struct LogAppenderDefine {
    int type = 0; //1 File, 2 Stdout
    LogLevel::Level level = LogLevel::UNKNOW;
    std::string formatter;
    std::string file;
	...
};

struct LogDefine {
    std::string name;
    LogLevel::Level level = LogLevel::UNKNOW;
    std::string formatter;
    std::vector<LogAppenderDefine> appenders;
	...
};

sylar::ConfigVar<std::set<LogDefine> >::ptr g_log_defines =
    sylar::Config::Lookup("logs", std::set<LogDefine>(), "logs config");

struct LogIniter {
    LogIniter() {
        g_log_defines->addListener([](\
            const std::set<LogDefine>& old_value,
            const std::set<LogDefine>& new_value ){ ... })}}

static LogIniter __log_init;

YAML::Node root = YAML::LoadFile("./bin/conf/log.yml");
sylar::Config::LoadFromYaml(root);
```

- 首先，LogDefine是对日志的定义，指明了需要什么logger，他的名称是什么，日志级别，日志格式，以及相应的logAppender
- 然后定义了一个名称为logs的配置项，其值类型为LogDefine的集合，保存了多个Logger的信息
- 然后定义了一个LogIniter的结构体，并声明一个全局静态变量。在使得在main函数执行之前，就为g_log_defines添加了一个监听器，此监听器的作用是在g_log_defines改变时，改变系统的log配置
- 最后是加载Log配置文件，这会触发logs配置的变更，然后触发上述监听器，从而使得系统log配置变更生效
