---
layout: post
title:  "Android O系统启动流程--init篇(最后阶段)"
date:   2019-11-06 00:00:02
catalog:  true
tags:
    - android
    - 系统启动
    - init
---

```
system/core/init
 - init.cpp
  - init_parser.cpp
 - init.rc
 - service.cpp
 - action.cpp
 - builtins.cpp
```

[TOC]

## 1 概述

 前面两篇博客已经介绍了init启动的第一和第二阶段，接下来看下 init启动的最后阶段：解析init.rc文件相关的工作。 

## 2 主线代码

[->init.cpp]

init启动的最后阶段。

```cpp
int main(int argc, char** argv) {
    ...
    //定义Action中的function_map_为BuiltinFuntionMap
    const BuiltinFunctionMap function_map;
    Action::set_function_map(&function_map);

    ActionManager& am = ActionManager::GetInstance();
    ServiceManager& sm = ServiceManager::GetInstance();
    //构造出解析文件用的parser对象
    Parser& parser = Parser::GetInstance();
    //为一些类型的关键字，创建特定的parser
    parser.AddSectionParser("service", std::make_unique<ServiceParser>(&sm));
    parser.AddSectionParser("on", std::make_unique<ActionParser>(&am));
    parser.AddSectionParser("import", std::make_unique<ImportParser>(&parser));
    //判断是否存在bootscript
    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    //如果没有bootscript，则解析init.rc文件
    if (bootscript.empty()) {
        parser.ParseConfig("/init.rc");
        parser.set_is_system_etc_init_loaded(
                parser.ParseConfig("/system/etc/init"));
        parser.set_is_vendor_etc_init_loaded(
                parser.ParseConfig("/vendor/etc/init"));
        parser.set_is_odm_etc_init_loaded(parser.ParseConfig("/odm/etc/init"));
    } else {
        //若存在bootscript, 则解析bootscript
        parser.ParseConfig(bootscript);
        parser.set_is_system_etc_init_loaded(true);
        parser.set_is_vendor_etc_init_loaded(true);
        parser.set_is_odm_etc_init_loaded(true);
    }

    // Turning this on and letting the INFO logging be discarded adds 0.2s to
    // Nexus 9 boot time, so it's disabled by default.
    if (false) DumpState();

    am.QueueEventTrigger("early-init");

    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
    am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    // ... so that we can start queuing up actions that require stuff from /dev.
    am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    am.QueueBuiltinAction(set_mmap_rnd_bits_action, "set_mmap_rnd_bits");
    am.QueueBuiltinAction(set_kptr_restrict_action, "set_kptr_restrict");
    am.QueueBuiltinAction(keychord_init_action, "keychord_init");
    am.QueueBuiltinAction(console_init_action, "console_init");

    // Trigger all the boot actions to get us started.
    am.QueueEventTrigger("init");

    // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
    // wasn't ready immediately after wait_for_coldboot_done
    am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");

    // Don't mount filesystems or start core system services in charger mode.
    std::string bootmode = GetProperty("ro.bootmode", "");
    if (bootmode == "charger") {
        am.QueueEventTrigger("charger");
    } else {
        am.QueueEventTrigger("late-init");
    }

    // Run all property triggers based on current state of the properties.
    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");

    while (true) {
        // By default, sleep until something happens.
        int epoll_timeout_ms = -1;

        if (do_shutdown && !shutting_down) {
            do_shutdown = false;
            if (HandlePowerctlMessage(shutdown_command)) {
                shutting_down = true;
            }
        }
        //当前没有事件需要处理时
        if (!(waiting_for_prop || sm.IsWaitingForExec())) {
             //依次执行每个action中携带command对应的执行函数
            am.ExecuteOneCommand();
        }
        //重启一些挂掉的进程
        if (!(waiting_for_prop || sm.IsWaitingForExec())) {
            if (!shutting_down) restart_processes();

            // If there's a process that needs restarting, wake up in time for that.
            if (process_needs_restart_at != 0) {
                epoll_timeout_ms = (process_needs_restart_at - time(nullptr)) * 1000;
                if (epoll_timeout_ms < 0) epoll_timeout_ms = 0;
            }

            // If there's more work to do, wake up again immediately.
            // 有command等着处理的话，不等待
            if (am.HasMoreCommands()) epoll_timeout_ms = 0;
        }

        epoll_event ev;
        //没有事件到来的话，最多阻塞epoll_timeout_ms时间
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, epoll_timeout_ms));
        if (nr == -1) {
            PLOG(ERROR) << "epoll_wait failed";
        } else if (nr == 1) {
            //有事件到来，执行对应处理函数
            //根据上文知道，epoll句柄（即epoll_fd）主要监听子进程结束，及其它进程设置系统属性的请求。
            ((void (*)()) ev.data.ptr)();
        }
    }

    return 0;
}
```

## 3  rc文件语法

分析解析init启动的最后流程之前，先简单了解下rc文件的语法。

rc文件语法是以行尾单位，以空格间隔的语法，以#开始代表注释行。rc文件主要包含Action、Service、Command、Options，其中对于Action和Service的名称都是唯一的，对于重复的命名视为无效。

### 3.1 Action

Action： 通过触发器trigger，即以on开头的语句来决定执行相应的service的时机，具体有如下时机：

- on early-init; 在初始化早期阶段触发；
- on init; 在初始化阶段触发；
- on late-init; 在初始化晚期阶段触发；
- on boot/charger： 当系统启动/充电时触发，还包含其他情况，此处不一一列举；
- on property:<key>=<value>: 当属性值满足条件时触发；

### 3.2 Service

服务Service，以 service开头，由init进程启动，一般运行在init的一个子进程，所以启动service前需要判断对应的可执行文件是否存在。init生成的子进程，定义在rc文件，其中每一个service在启动时会通过fork方式生成子进程。

例如： `service servicemanager /system/bin/servicemanager`代表的是服务名为servicemanager，服务执行的路径为/system/bin/servicemanager。

### 3.3 Command

下面列举常用的命令

- class_start <service_class_name>： 启动属于同一个class的所有服务；
- start <service_name>： 启动指定的服务，若已启动则跳过；
- stop <service_name>： 停止正在运行的服务
- setprop <name> <value>：设置属性值
- mkdir <path>：创建指定目录
- symlink <target> <sym_link>： 创建连接到<target>的<sym_link>符号链接；
- write <path> <string>： 向文件path中写入字符串；
- exec： fork并执行，会阻塞init进程直到程序完毕；
- exprot <name> <name>：设定环境变量；
- loglevel <level>：设置log级别

### 3.4 Options

Options是Service的可选项，与service配合使用

- disabled: 不随class自动启动，只有根据service名才启动；
- oneshot: service退出后不再重启；
- user/group： 设置执行服务的用户/用户组，默认都是root；
- class：设置所属的类名，当所属类启动/退出时，服务也启动/停止，默认为default；
- onrestart:当服务重启时执行相应命令；
- socket: 创建名为`/dev/socket/`的socket
- critical: 在规定时间内该service不断重启，则系统会重启并进入恢复模式

**default:** 意味着disabled=false，oneshot=false，critical=false。

## 4  **创建Parser并解析文件** 

```c++
//定义Action中的function_map_为BuiltinFuntionMap
    const BuiltinFunctionMap function_map;
    Action::set_function_map(&function_map);

    ActionManager& am = ActionManager::GetInstance();
    ServiceManager& sm = ServiceManager::GetInstance();
    //构造出解析文件用的parser对象
    Parser& parser = Parser::GetInstance();
    //为一些类型的关键字，创建特定的parser
    parser.AddSectionParser("service", std::make_unique<ServiceParser>(&sm));
    parser.AddSectionParser("on", std::make_unique<ActionParser>(&am));
    parser.AddSectionParser("import", std::make_unique<ImportParser>(&parser));
    //判断是否存在bootscript
    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    //如果没有bootscript，则解析init.rc文件
    if (bootscript.empty()) {
        parser.ParseConfig("/init.rc");
        parser.set_is_system_etc_init_loaded(
                parser.ParseConfig("/system/etc/init"));
        parser.set_is_vendor_etc_init_loaded(
                parser.ParseConfig("/vendor/etc/init"));
        parser.set_is_odm_etc_init_loaded(parser.ParseConfig("/odm/etc/init"));
    } else {
        //若存在bootscript, 则解析bootscript
        parser.ParseConfig(bootscript);
        parser.set_is_system_etc_init_loaded(true);
        parser.set_is_vendor_etc_init_loaded(true);
        parser.set_is_odm_etc_init_loaded(true);
    }
```

从上面的代码来看，8.0引入了bootScript的概念，应该是方便厂商的定制。
如果没有定义bootScript，那么init进程还是会解析init.rc文件。 init.rc文件是在init进程启动后执行的启动脚本，文件中记录着init进程需执行的操作。 此处解析函数传入的参数为“/init.rc”，解析的是运行时与init进程同在根目录下的init.rc文件。 该文件在编译前，定义于system/core/rootdir/init.rc中。
init.rc文件大致分为两大部分，一部分是以“on”关键字开头的动作列表（action list）

```c++
on early-init
    # Set init and its forked children's oom_adj.
    write /proc/1/oom_score_adj -1000
    .........
    start ueventd
```

 另一部分是以“service”关键字开头的服务列表（service list）： 

```c++
service ueventd /sbin/ueventd
    class core
    critical
    seclabel u:r:ueventd:s0
```

### 4.1  ParseConfig 

 接下来，我们从Parser的`ParseConfig`函数入手，逐步分析整个文件解析的过程。 

[->init_parser.cpp]

```c++
bool Parser::ParseConfig(const std::string& path) {
    if (is_dir(path.c_str())) {
        //传入参数为目录地址
        return ParseConfigDir(path);
    }
    //传入参数为文件地址
    return ParseConfigFile(path);
}
```

 跟进一下ParseConfigFile函数： 

```c++
bool Parser::ParseConfigFile(const std::string& path) {
    LOG(INFO) << "Parsing file " << path << "...";
    android::base::Timer t;
    std::string data;
    std::string err;
    //读取路径指定文件中的内容，保存为字符串形式
    if (!ReadFile(path, &data, &err)) {
        LOG(ERROR) << err;
        return false;
    }

    data.push_back('\n'); // TODO: fix parse_config.
    //解析获取的字符串
    ParseData(path, data);
    for (const auto& [section_name, section_parser] : section_parsers_) {
        section_parser->EndFile();
    }

    LOG(VERBOSE) << "(Parsing " << path << " took " << t << ".)";
    return true;
}
```

容易看出，ParseConfigFile只是读取文件的内容并转换为字符串，实际的解析工作被交付给ParseData。

### 4.2 ParseData

```cpp
void Parser::ParseData(const std::string& filename, const std::string& data) {
    //TODO: Use a parser with const input and remove this copy
     //copy数据
    std::vector<char> data_copy(data.begin(), data.end());
    data_copy.push_back('\0');
    //解析用的结构体
    parse_state state;
    state.line = 0;
    state.ptr = &data_copy[0];
    state.nexttoken = 0;

    SectionParser* section_parser = nullptr;
    std::vector<std::string> args;

    for (;;) {
        //next_token获取分割符，初始没有分割符时，进入T_TEXT分支
        switch (next_token(&state)) {
        case T_EOF:
            if (section_parser) {
                //EOF,解析结束
                section_parser->EndSection();
            }
            return;
        case T_NEWLINE:
            state.line++;
            if (args.empty()) {
                break;
            }
            // If we have a line matching a prefix we recognize, call its callback and unset any
            // current section parsers.  This is meant for /sys/ and /dev/ line entries for uevent.
            for (const auto& [prefix, callback] : line_callbacks_) {
                if (android::base::StartsWith(args[0], prefix.c_str())) {
                    if (section_parser) section_parser->EndSection();

                    std::string ret_err;
                    if (!callback(std::move(args), &ret_err)) {
                        LOG(ERROR) << filename << ": " << state.line << ": " << ret_err;
                    }
                    section_parser = nullptr;
                    break;
                }
            }
            //在前文创建parser时，我们为service，on，import定义了对应的parser 
            //这里就是根据第一个参数，判断是否有对应的parser
            if (section_parsers_.count(args[0])) {
                if (section_parser) {
                   //结束上一个parser的工作，将构造出的对象加入到对应的service_list与action_list中
                    section_parser->EndSection();
                }
                //获取参数对应的parser
                section_parser = section_parsers_[args[0]].get();
                std::string ret_err;
                //调用实际parser的ParseSection函数
                if (!section_parser->ParseSection(std::move(args), filename, state.line, &ret_err)) {
                    LOG(ERROR) << filename << ": " << state.line << ": " << ret_err;
                    section_parser = nullptr;
                }
            } else if (section_parser) {
                //如果新的一行，第一个参数不是service，on，import
                //则调用前一个parser的ParseLineSection函数
                //这里相当于解析一个参数块的子项
                std::string ret_err;
                if (!section_parser->ParseLineSection(std::move(args), state.line, &ret_err)) {
                    LOG(ERROR) << filename << ": " << state.line << ": " << ret_err;
                }
            }
            //清空本次解析的数据
            args.clear();
            break;
        case T_TEXT:
            //将本次解析的内容写入到args中
            args.emplace_back(state.text);
            break;
        }
    }
}
```

主要工作是负责根据关键字解析出服务和动作。 动作与服务会以链表节点的形式注册到service_list与action_list中， `service_list`与`action_list`是init进程中声明的全局结构体。 

回头看下之前创建parser时的代码：

```c++
Parser& parser = Parser::GetInstance();
parser.AddSectionParser("service", std::make_unique<ServiceParser>(&sm));
parser.AddSectionParser("on", std::make_unique<ActionParser>(&am));
parser.AddSectionParser("import", std::make_unique<ImportParser>(&parser));
```

 三种Parser均是继承SectionParser，具体的实现各有不同。 接下来，我们以比较常用的ServiceParser和ActionParser为例， 看看解析的结果如何处理。 

### 4.3 ServiceParser

[->service.cpp]

从前面的代码我们知道，解析一个service块时，首先需要调用ParseSection函数， 接着利用ParseLineSection处理子块，解析完所有数据后，最后调用EndSection。 因此，我们着重看看ServiceParser的这三个函数。 

#### 4.3.1 ParseSection

```c++
bool ServiceParser::ParseSection(std::vector<std::string>&& args, const std::string& filename,
                                 int line, std::string* err) {
    //参数检验
    if (args.size() < 3) {
        *err = "services must have a name and a program";
        return false;
    }
    //服务名
    const std::string& name = args[1];
    //服务名校验
    if (!IsValidName(name)) {
        *err = StringPrintf("invalid service name '%s'", name.c_str());
        return false;
    }

    Service* old_service = service_manager_->FindServiceByName(name);
    if (old_service) {
        *err = "ignored duplicate definition of service '" + name + "'";
        return false;
    }
    //构造出一个service对象
    std::vector<std::string> str_args(args.begin() + 2, args.end());
    service_ = std::make_unique<Service>(name, str_args);
    return true;
}
```

 ParseSection主要校验参数的有效性，并创建出Service结构体。 

#### 4.3.2  **ParseLineSection** 

```c++
bool ServiceParser::ParseLineSection(std::vector<std::string>&& args, int line, std::string* err) {
    //调用service对象的HandleLine
    return service_ ? service_->ParseLine(std::move(args), err) : false;
}
```

```c++
bool Service::ParseLine(const std::vector<std::string>& args, std::string* err) {
    //OptionParserMap继承自keywordMap<OptionParser>
    static const OptionParserMap parser_map;
    //根据子项的内容，找到对应的处理函数
    //FindFunction利用OptionParserMap的map，根据参数找到对应的处理函数
    auto parser = parser_map.FindFunction(args, err);

    if (!parser) {
        return false;
    }
    //调用对应的处理函数
    return (this->*parser)(args, err);
}
```

 为了了解这部分内容，我们需要看看OptionParserMap中的map函数： 

```c++
class Service::OptionParserMap : public KeywordMap<OptionParser> {
  public:
    OptionParserMap() {}

  private:
    const Map& map() const override;
};

const Service::OptionParserMap::Map& Service::OptionParserMap::map() const {
    constexpr std::size_t kMax = std::numeric_limits<std::size_t>::max();
    // clang-format off
    //定义了各种关键子对应的处理函数
    static const Map option_parsers = {
        {"capabilities",
                        {1,     kMax, &Service::ParseCapabilities}},
        {"class",       {1,     kMax, &Service::ParseClass}},
        {"console",     {0,     1,    &Service::ParseConsole}},
        {"critical",    {0,     0,    &Service::ParseCritical}},
        {"disabled",    {0,     0,    &Service::ParseDisabled}},
        {"group",       {1,     NR_SVC_SUPP_GIDS + 1, &Service::ParseGroup}},
        {"ioprio",      {2,     2,    &Service::ParseIoprio}},
        {"priority",    {1,     1,    &Service::ParsePriority}},
        {"keycodes",    {1,     kMax, &Service::ParseKeycodes}},
        {"oneshot",     {0,     0,    &Service::ParseOneshot}},
        {"onrestart",   {1,     kMax, &Service::ParseOnrestart}},
        {"oom_score_adjust",
                        {1,     1,    &Service::ParseOomScoreAdjust}},
        {"memcg.swappiness",
                        {1,     1,    &Service::ParseMemcgSwappiness}},
        {"memcg.soft_limit_in_bytes",
                        {1,     1,    &Service::ParseMemcgSoftLimitInBytes}},
        {"memcg.limit_in_bytes",
                        {1,     1,    &Service::ParseMemcgLimitInBytes}},
        {"namespace",   {1,     2,    &Service::ParseNamespace}},
        {"seclabel",    {1,     1,    &Service::ParseSeclabel}},
        {"setenv",      {2,     2,    &Service::ParseSetenv}},
        {"shutdown",    {1,     1,    &Service::ParseShutdown}},
        {"socket",      {3,     6,    &Service::ParseSocket}},
        {"file",        {2,     2,    &Service::ParseFile}},
        {"user",        {1,     1,    &Service::ParseUser}},
        {"writepid",    {1,     kMax, &Service::ParseWritepid}},
    };
    // clang-format on
    return option_parsers;
}
```

 我们以class对应的处理函数为例，看看对应的代码： 

```c++
bool Service::ParseClass(const std::vector<std::string>& args, std::string* err) {
    classnames_ = std::set<std::string>(args.begin() + 1, args.end());
    return true;
}
```

这部分代码其实就是填充service对象对应的域。因此，可以推断ParseLineSection函数的作用，就是根据关键字填充Service对象。

#### 4.3.3  **EndSection**  

```c++
//注意此时service对象已经构造完毕
void ServiceParser::EndSection() {
    if (service_) {
        service_manager_->AddService(std::move(service_));
    }
}
```

 我们继续跟进AddService函数： 

```c++
void ServiceManager::AddService(std::unique_ptr<Service> service) {
    //将service对象加入到services_里
    services_.emplace_back(std::move(service));
}
```

#### 4.3.4 小结

从上面的一系列代码，我们可以看出ServiceParser的工作流程就是： 

1. 根据第一行的名字和参数创建出service对象； 
2. 根据子项的内容填充service对象； 
3. 将创建出的service对象加入到vector类型的service链表中。 

### 4.4  **ActionParser**  

 Action的解析过程，其实与Service一样，也是先后调用ParseSection， ParseLineSection和EndSection。 

[->action.cpp]

#### 4.4.1  **ParseSection** 

```c++
bool ActionParser::ParseSection(std::vector<std::string>&& args, const std::string& filename,
                                int line, std::string* err) {
    std::vector<std::string> triggers(args.begin() + 1, args.end());
    if (triggers.size() < 1) {
        *err = "actions must have a trigger";
        return false;
    }

    auto action = std::make_unique<Action>(false, filename, line);
    if (!action->InitTriggers(triggers, err)) {
        return false;
    }

    action_ = std::move(action);
    return true;
}
```

 与Service类似，Action的ParseSection函数用于构造出Action对象。 

#### 4.4.2 ParseLineSection

```c++
bool ActionParser::ParseLineSection(std::vector<std::string>&& args, int line, std::string* err) {
    return action_ ? action_->AddCommand(std::move(args), line, err) : false;
}
```

```c++
bool Action::AddCommand(const std::vector<std::string>& args, int line, std::string* err) {
    if (!function_map_) {
        *err = "no function map available";
        return false;
    }
    //找出action对应的执行函数
    auto function = function_map_->FindFunction(args, err);
    if (!function) {
        return false;
    }
    //利用所有信息构造出command，加入到action对象中
    AddCommand(function, args, line);
    return true;
}

void Action::AddCommand(BuiltinFunction f, const std::vector<std::string>& args, int line) {
    commands_.emplace_back(f, args, line);
}
```

ParseLineSection主要是根据参数填充Action的command域。 一个Action对象中可以有多个command。 Action中调用`function_map_->FindFunction`时， 实际上调用的是BuiltinFunctionMap的`FindFunction`函数。与查找Service一样，这里也是根据键值查找对应的信息， 因此重点是看看BuiltinFunctionMap的`map`函数。

[-> builtins.cpp]

```c++
const BuiltinFunctionMap::Map& BuiltinFunctionMap::map() const {
    constexpr std::size_t kMax = std::numeric_limits<std::size_t>::max();
    // clang-format off
    static const Map builtin_functions = {
        {"bootchart",               {1,     1,    do_bootchart}},
        {"chmod",                   {2,     4,    do_chmod}},
        {"chown",                   {2,     5,    do_chown}},
        {"class_reset",             {1,     1,    do_class_reset}},
        {"class_restart",           {1,     1,    do_class_restart}},
        {"class_start",             {1,     1,    do_class_start}},
        {"class_stop",              {1,     1,    do_class_stop}},
        {"copy",                    {2,     2,    do_copy}},
        {"domainname",              {1,     1,    do_domainname}},
        {"enable",                  {1,     1,    do_enable}},
        {"exec",                    {1,     kMax, do_exec}},
        {"exec_start",              {1,     1,    do_exec_start}},
        {"export",                  {2,     2,    do_export}},
        {"hostname",                {1,     1,    do_hostname}},
        {"ifup",                    {1,     1,    do_ifup}},
        {"init_user0",              {0,     0,    do_init_user0}},
        {"insmod",                  {1,     kMax, do_insmod}},
        {"installkey",              {1,     1,    do_installkey}},
        {"load_persist_props",      {0,     0,    do_load_persist_props}},
        {"load_system_props",       {0,     0,    do_load_system_props}},
        {"loglevel",                {1,     1,    do_loglevel}},
        {"mkdir",                   {1,     4,    do_mkdir}},
        {"mount_all",               {1,     kMax, do_mount_all}},
        {"mount",                   {3,     kMax, do_mount}},
        {"umount",                  {1,     1,    do_umount}},
        {"restart",                 {1,     1,    do_restart}},
        {"restorecon",              {1,     kMax, do_restorecon}},
        {"restorecon_recursive",    {1,     kMax, do_restorecon_recursive}},
        {"rm",                      {1,     1,    do_rm}},
        {"rmdir",                   {1,     1,    do_rmdir}},
        {"setprop",                 {2,     2,    do_setprop}},
        {"setrlimit",               {3,     3,    do_setrlimit}},
        {"start",                   {1,     1,    do_start}},
        {"stop",                    {1,     1,    do_stop}},
        {"swapon_all",              {1,     1,    do_swapon_all}},
        {"symlink",                 {2,     2,    do_symlink}},
        {"sysclktz",                {1,     1,    do_sysclktz}},
        {"trigger",                 {1,     1,    do_trigger}},
        {"verity_load_state",       {0,     0,    do_verity_load_state}},
        {"verity_update_state",     {0,     0,    do_verity_update_state}},
        {"wait",                    {1,     2,    do_wait}},
        {"wait_for_prop",           {2,     2,    do_wait_for_prop}},
        {"write",                   {2,     4,    do_write}},
    };
    // clang-format on
    return builtin_functions;
}
```

 上述代码的第四项就是Action每个command对应的执行函数。 

#### 4.4.3  **EndSection** 

```c++
void ActionParser::EndSection() {
    //Action有效时，才需要加入
    if (action_ && action_->NumCommands() > 0) {
        action_manager_->AddAction(std::move(action_));
    }
}
```

```c++
void ActionManager::AddAction(std::unique_ptr<Action> action) {
    //加入到action链表中，类型也是vector，其中装的是指针
    actions_.emplace_back(std::move(action));
}
```

 从上面的代码可以看出，加载action块的逻辑和service一样，不同的是需要填充trigger和command域。 当然，最后解析出的action也需要加入到action链表中。 

## 5  **向Action队列中添加其它action**  

介绍完init进程解析init.rc文件的过程后， 我们继续将视角拉回到init进程的main函数： 

```c++
am.QueueEventTrigger("early-init");

    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
    am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    // ... so that we can start queuing up actions that require stuff from /dev.
    am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    am.QueueBuiltinAction(set_mmap_rnd_bits_action, "set_mmap_rnd_bits");
    am.QueueBuiltinAction(set_kptr_restrict_action, "set_kptr_restrict");
    am.QueueBuiltinAction(keychord_init_action, "keychord_init");
    am.QueueBuiltinAction(console_init_action, "console_init");

    // Trigger all the boot actions to get us started.
    am.QueueEventTrigger("init");

    // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
    // wasn't ready immediately after wait_for_coldboot_done
    am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");

    // Don't mount filesystems or start core system services in charger mode.
    std::string bootmode = GetProperty("ro.bootmode", "");
    if (bootmode == "charger") {
        am.QueueEventTrigger("charger");
    } else {
        am.QueueEventTrigger("late-init");
    }

    // Run all property triggers based on current state of the properties.
    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");
```

 init进程中调用了大量的QueueBuiltinAction和QueueEventTrigger函数。接下来，我们就来看看这两个函数进行了哪些工作。 

### 5.1  **QueueBuiltinAction**  

[->action.cpp]

```c++
void ActionManager::QueueBuiltinAction(BuiltinFunction func, const std::string& name) {
    //创建action
    auto action = std::make_unique<Action>(true, "<Builtin Action>", 0);
    std::vector<std::string> name_vector{name};
    //保证trigger name的唯一性
    if (!action->InitSingleTrigger(name)) {
        return;
    }
    //创建action的cmd，指定执行函数和参数
    action->AddCommand(func, name_vector, 0);

    event_queue_.emplace(action.get());
    actions_.emplace_back(std::move(action));
}
```

 QueueBuiltinAction函数中将构造新的action加入到actions链表中，并将trigger事件加入到`trigger_queue`中。

### 5.2 QueueEventTrigger

```c++
void ActionManager::QueueEventTrigger(const std::string& trigger) {
    event_queue_.emplace(trigger);
}
```

此处QueueEventTrigger函数就是利用参数构造EventTrigger，然后加入到trigger_queue_中。 后续init进程处理trigger事件时，将会触发相应的操作。 

## 6  **处理添加到运行队列的事件**  

```c++
while (true) {
        // By default, sleep until something happens.
        int epoll_timeout_ms = -1;

        if (do_shutdown && !shutting_down) {
            do_shutdown = false;
            if (HandlePowerctlMessage(shutdown_command)) {
                shutting_down = true;
            }
        }
        //当前没有事件需要处理时
        if (!(waiting_for_prop || sm.IsWaitingForExec())) {
            //依次执行每个action中携带command对应的执行函数
            am.ExecuteOneCommand();
        }
        if (!(waiting_for_prop || sm.IsWaitingForExec())) {
            //重启一些挂掉的进程
            if (!shutting_down) restart_processes();

            // If there's a process that needs restarting, wake up in time for that.
            if (process_needs_restart_at != 0) {
                epoll_timeout_ms = (process_needs_restart_at - time(nullptr)) * 1000;
                if (epoll_timeout_ms < 0) epoll_timeout_ms = 0;
            }

            // If there's more work to do, wake up again immediately.
            // 有command等着处理的话，不等待
            if (am.HasMoreCommands()) epoll_timeout_ms = 0;
        }

        epoll_event ev;
        //没有事件到来的话，最多阻塞epoll_timeout_ms时间
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, epoll_timeout_ms));
        if (nr == -1) {
            PLOG(ERROR) << "epoll_wait failed";
        } else if (nr == 1) {
            //有事件到来，执行对应处理函数
            //根据上文知道，epoll句柄（即epoll_fd）主要监听子进程结束，及其它进程设置系统属性的请求。
            ((void (*)()) ev.data.ptr)();
        }
    }
```

从上面代码可以看出，最终init进程将进入无限循环中， 不断处理运行队列中的事件、完成重启进程、监听`epoll_fd`等操作。接下来，我们关注一下其中比较关键的函数ExecuteOneCommand和restart_processes。 

### 6.1 ExecuteOneCommand

[->action.cpp]

```c++
void ActionManager::ExecuteOneCommand() {
    // Loop through the event queue until we have an action to execute
    //当有可执行的action或trigger queue为空时结束
    while (current_executing_actions_.empty() && !event_queue_.empty()) {
        //轮询actions链表
        for (const auto& action : actions_) {
            //依次查找trigger表
            if (std::visit([&action](const auto& event) { return action->CheckEvent(event); },
                           event_queue_.front())) {
                //当action与trigger对应时，就可以执行当前action
                //一个trigger可以对应多个action，均加入current_executing_actions_
                current_executing_actions_.emplace(action.get());
            }
        }
        //trigger event出队
        event_queue_.pop();
    }
    //上面的代码说明，执行的顺序又trigger queue决定，没有可执行的action时，直接退出
    if (current_executing_actions_.empty()) {
        return;
    }
    //每次只执行一个action，下次init进程while循环时，接着执行
    auto action = current_executing_actions_.front();

    if (current_command_ == 0) {
        std::string trigger_name = action->BuildTriggersString();
        LOG(INFO) << "processing action (" << trigger_name << ") from (" << action->filename()
                  << ":" << action->line() << ")";
    }
    //实际的执行过程，此处仅处理当前action中的一个cmd，current_command_记录当前command对应的编号
    //实际上就是执行该command对应的处理函数
    action->ExecuteOneCommand(current_command_);

    // If this was the last command in the current action, then remove
    // the action from the executing list.
    // If this action was oneshot, then also remove it from actions_.
    //适当地清理工作，注意只有当前action中所有的command均执行完毕后，
    //才会将该action从current_executing_actions_移除
    ++current_command_;
    if (current_command_ == action->NumCommands()) {
        current_executing_actions_.pop();
        current_command_ = 0;
        if (action->oneshot()) {
            auto eraser = [&action] (std::unique_ptr<Action>& a) {
                return a.get() == action;
            };
            actions_.erase(std::remove_if(actions_.begin(), actions_.end(), eraser));
        }
    }
}
```

从代码可以看出，当while循环不断调用ExecuteOneCommand函数时，将按照trigger表的顺序， 依次取出action链表中与trigger匹配的action。每次均仅仅执行一个action中的一个command对应函数（一个action可能携带多个command）。 当一个action所有的command均执行完毕后，再执行下一个action。 当一个trigger对应的action均执行完毕后，再执行下一个trigger对应action。

### 6.2 restart_processes

[->init.cpp]

```c++
static void restart_processes()
{
    process_needs_restart_at = 0;
    ServiceManager::GetInstance().ForEachServiceWithFlags(SVC_RESTARTING, [](Service* s) {
        s->RestartIfNeeded(&process_needs_restart_at);
    });
}
```

从上面可以看出，该函数将轮询service对应的链表， 对于有SVC_RESTARING标志的service执行RestartIfNeeded函数。 前文已经提到过，当子进程终止时，init进程会将可被重启进程的服务标志位置为SVC_RESTARTING。我们进一步看看RestartIfNeeded函数：

[->service.cpp]

```c++
void Service::RestartIfNeeded(time_t* process_needs_restart_at) {
    boot_clock::time_point now = boot_clock::now();
    boot_clock::time_point next_start = time_started_ + 5s;
    //两次服务启动进程的间隔要大于5s
    if (now > next_start) {
        flags_ &= (~SVC_RESTARTING);
        //满足时间间隔的要求后，重启进程。Start将会重新fork服务进程，并做相应的配置
        Start();
        return;
    }
    //更新process_needs_restart_at的值，将影响前文epoll_wait的等待时间
    time_t next_start_time_t = time(nullptr) +
        time_t(std::chrono::duration_cast<std::chrono::seconds>(next_start - now).count());
    if (next_start_time_t < *process_needs_restart_at || *process_needs_restart_at == 0) {
        *process_needs_restart_at = next_start_time_t;
    }
}
```

 当init子进程退出时，会产生SIGCHLD信号，并发送给init进程，通过socket套接字传递数据，调用`wait_for_one_process()`方法，根据是否是oneshot，来决定是重启子进程，还是放弃启动。 

![restart-service](/images/startup/restart-service.png)

## 7 总结

 至此，init进程的启动流程分析完毕， init启动的最后阶段主要的工作如下：

1. 创建Parser并解析rc文件，如启动各种service，如java的第一个进程Zygote
2. 向Action队列添加其他action
3. 处理添加到运行队列的事件