---
title: Using ATrace(Android system and app trace events)
date: 2020-11-12
---

## atrace是什么

atarce(Android system and app trace events)是Android系统和软件的事件追踪器，支持用户空间的系统事件和软件事件，以及内核空间的函数追踪。它支持设置属性来进行用户空间追踪，也可以写入ftrace的sysfs节点进行事件追踪，还能利用function tracer进行内核中大部分函数进行追踪。
它一方面通过NDK层设置接口给JNI提供API，另一方面又通过zygote在Android内部进程中对属性及时响应，还在通过设置ftrace下的节点属性控制信息输出。

## atrace流程图

![atrace_2](/images/using_atrace/atrace_2.jpg)

## atrace信息捕获

根据atrace.cpp源码可以看出，对于追踪类型的分类依据来源于k_categories结构中定义好的值。k_categories是一个TracingCategory结构体，定义如下：
```
struct TracingCategory {
    // The name identifying the category.
    const char* name;

    // A longer description of the category.
    const char* longname;

    // The userland tracing tags that the category enables.
    uint64_t tags;

    // The fname==NULL terminated list of /sys/ files that the category
    // enables.
    struct {
        // Whether the file must be writable in order to enable the tracing
        // category.
        requiredness required;

        // The path to the enable file.
        const char* path;
    } sysfiles[MAX_SYS_FILES];
};
```
其中成员tags能够对用户空间的追踪选项进行标识，sysfiles则提供了某个追踪类别的节点信息。atrace使用参数-k来标识是否启用内核函数追踪。根据参数-k和k_categories可以知道，atrace主要采用如下几种方式来进行追踪。

#### 1. ATRACE_TAG方式

ATRACE_TAG方式是atrace的默认方式，使能方法是传入默认的追踪类型。它是只针对用户空间的追踪方式，通过将k_categories中各个非零tags的标识位进行按位或运算后来设置系统属性debug.atrace.tags.enableflags。在Android系统中，zygote进程启动后，通过一系列调用判断ATRACE_TAG的设置和ATRACE_CALL()或ATrace_beginSection的位置，以WRITE_MSG的形式写入atrace_marker_fd，即trace_marker的位置，填充ftrace的ring buffer。具体的时序图如下:

![zygote](/images/using_atrace/zygote.jpg)

![getprop](/images/using_atrace/getprop.jpg)

![atrace_call](/images/using_atrace/atrace_call.jpg)

在NDK层封装好API后，通过JNI的机制，就可以在java里调用trace。在`frameworks/base/core/jni/android_os_Trace.cpp`中设置如下:

![jniatrace](/images/using_atrace/jniatrace.png)

对于ATRACE_TAG来说，java层采取和libcutils相同的方式，采用相同的标志位进行声明，确保分类一致。

![tracetag](/images/using_atrace/tracetag.jpg)

之后就可以在framework层中轻松的使用trace了，例如:

![framework](/images/using_atrace/framework.png)

#### 2. appname方式

appname方式能够对设置的进程名称进行追踪，但可能会导致一系列不可控的返回信息。其使能方式是通过参数-a传入包名来进行设置，然后设置系统属性debug.atrace.app_number和debug.atrace.app，确定追踪的包名的个数和对应的属性，之后也是类似ATRACE_TAG的方式使用libcutils进行追踪。具体的时序图如下：

![appname](/images/using_atrace/appname.jpg)

appname方式会在进程内部读取/proc/self/cmdline的第一行，然后和系统属性debug.atrace.app进行比对，当匹配成功时代表该进程需要进行app级别的调试追踪，然后设置ATRACE_TAG_APP，

#### 3. trace event方式

trace event方式是对debugfs中events下的节点使能来进行追踪控制的。通常来说，trace event使用宏TRACE_EVENT()来新增追踪事件。当对应节点被使能后，函数插桩被激活，桩点中的函数被调用时就会被记录下来，将数据通过设置好的filter和trigger存储到ring buffer中，最后通过设置好的format进行格式化输出。
对于format中每个块来说，filed格式为`field:field-type field-name; offset:N; size:N;`，如果一个field-name以`common_`开头的话，说明这个field是公共定义的，在相同的event下都享受这个filed。
对于event的filter来说，可以通过多种方式就行过滤。比如可以通过表达式语法来指定某些条件下field-name的匹配；也可以直接控制某个event下的所有子系统，此时会根据具体的filter下是否有修改的field来进行修改；还可以利用PID的方式直接对`set_event_pid`节点进行设置，
trigger则是用来对event的控制命令进行过滤。它可以控制这个另一个event的开启与关闭当这个event的trigger被激活时，也可以转储堆栈追踪、记录捕获的函数到snapshot中，也能直接控制追踪的开启与关闭。值得注意的是，这些trigger commands都能通过表达式语法来进行更细致的控制，能够直接对每个event定义的`TP_STRUCT__entry`中的成员来用于条件判断。

#### 4. 内核函数追踪方式

内核函数追踪方式能够对内核中大部分函数(不包括notrace、inline、某些特殊函数)进行追踪，其原理是ftrace中的dynamic fucntion tracer。内核函数追踪的使能方式是通过参数-k来设置要追踪哪几个函数。dynamic fucntion tracer有两种方式，一种是function的方式，另一种是function_graph的方式。对于dynamic fucntion tracer来说，它可以把不需要追踪的函数入口处指令`bl _mcount`替换成`nop`，这样基本对性能无影响，对需要追踪的函数替换入口处指令`bl _mcount`替换为`ftrace_caller`。对于dynamic function_graph tracer来说，则需要对函数的入口和出口同时插桩，获得函数的执行时间。需要注意的是，atrace默认的内核函数采用function_graph的方式进行，而修改后的atrace则可以采用function的方式。

## function与function_graph的性能比较

![compare](/images/using_atrace/compare.png)

根据实际测试比较可知，在使用function和function_graph两种tracer时候，在其他参数一致的情况下，两种方式的追踪消耗时间实际上是差不多的，而且内存占用也差不多。但是function tracer比function_graph tracer的cpu消耗多了40%，而且与系统追踪相比，内核追踪的cpu消耗极大。