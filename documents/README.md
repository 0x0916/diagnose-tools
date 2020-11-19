# diagnose-tools 2.0 功能使用说明

本文档介绍diagnose-tools-2.0版本的使用方法。

# 支持的版本
## 支持的版本

目前的版本支持Centos 7.5 / 7.6

经过验证，工具也支持如下版本，相应的代码会陆续合入：
* Centos 5.x / 6.x
* Ubuntu
* Linux 4.19

# 编译

推荐在centos 7.6中编译
```
make devel //安装编译环境
make deps  //编译三方开源包
make module  //编译内核模块
make tools  //编译用户态工具
make rpm    //制作rpm包
```

# 安装和卸载KO

在使用模块功能之前，需要用如下命令安装KO模块：
```
diagnose-tools install
```
安装成功后，控制台有如下提示：
```
installed successfully
```

在使用完模块功能后，需要用如下命令卸载KO模块：
```
diagnose-tools uninstall
```
卸载成功后，控制台有如下提示：
```
uninstalled successfully
```

# 2.0正式版本的功能

目前，diagnose-tools-2.0正式发布的版本有如下几个功能：
* 实用小工具pupil：按照tid查询特定线程在主机上的PID/进程名称/进程链/堆栈等等。
* sys-delay：监控syscall长时间运行引起调度不及时。间接引起系统Load高、业务RT高。
* sys-cost：统计系统调用的次数及时间。
* sched-delay : 监控系统调度延迟。找到引起调度延迟的进程。
* irq-delay：监控中断被延迟的时间。
* irq-stats：统计中断/软中断执行次数及时间。
* irq-trace：跟踪系统中IRQ/定时器的执行。
* load-monitor：监控系统Load值。每10ms查看一下系统当前Load，超过特定值时，输出任务堆栈。这个功能多次在线上抓到重大BUG。
可以分别监控Load/Load.R/Load.D/Task.D等指标。
* run-trace：监控进程在某一段时间段内，在用户态/内核态运行情况。
* perf: 对线程/进程进行性能采样，抓取用户态/内核态调用链。
* kprobe：在内核任意函数中，利用kprobe监控其执行，并输出火焰图。
* uprobe：在用户态应用程序中使用探针，在应用中挂接钩子。
* utilization：监控系统资源利用率，找到CPU被哪些野进程干扰，以及进程对内存的使用情况。
* exit-monitor：监控任务退出。在退出时，打印任务的用户态堆栈信息。
* mutex-monitor：监控长时间持有mutex的流程。
* exec-monitor: 监控进程调用exec系统调用创建新进程。
* alloc-top：统计内存分配数量，按序输出内存分配多的进程
* high-order：监控分配大内存的调用链
* drop-packet：监控内核TCP/IP各个流程中的丢包。
* tcp-retrans：监控TCP/IP套接字上的重传。
* ping-delay：监控ping报文在内核中的路径，确认影响报文延迟的原因。
* rw-top：监控写文件。找到突发引起文件读写的进程/调用链。
* fs-shm：本功能监控当前打开的SHM文件。
* fs-orphan：导出文件系统孤儿节点。
* fs-cache：监控文件系统缓存占用情况，统计每个文件占用的缓存数量。
* reboot：监控系统重启信息，打印出调用sys_reboot系统调用的进程名称以及进程链。

##  pupil小工具
### 查看diagnose-tools版本号
可以执行如下命令来查询版本号：
```
diagnose-tools -v
diagnose-tools -V
diagnose-tools --version
```
结果如下：
```
diagnose-tools tools version 2.0-rc4
```
### 查看线程信息

在容器或者宿主机上面，根据线程PID，输出其线程信息：

* 线程所在的CGROUP名称
* PID
* 进程名称
* 进程链信息
* 内核态堆栈

在控制台中运行如下命令查看线程信息：
```
diagnose-tools task-info --pid=$PID
```
其中，$PID是要查看的进程ID。
也可以用如下命令查看进程中所有线程的信息：
```
diagnose-tools task-info --tgid=$PID
```
最后，用如下命令获得结果：
```
diagnose-tools task-info --report
下图是运行结果示例：
线程详细信息： 4959
    时间：[1584776688:293341].
    进程信息： [/ / JS Helper]， PID： 4935 / 4959
##CGROUP:[/]  4959      [013]  采样命中
    内核态堆栈：
#@        0xffffffff8111ac34 futex_wait_queue_me  ([kernel.kallsyms])
#@        0xffffffff8111b8a6 futex_wait  ([kernel.kallsyms])
#@        0xffffffff8111dcb5 do_futex  ([kernel.kallsyms])
#@        0xffffffff8111e055 SyS_futex  ([kernel.kallsyms])
#@        0xffffffff81003c04 do_syscall_64  ([kernel.kallsyms])
#@        0xffffffff8174bfce entry_SYSCALL_64_after_swapgs  ([kernel.kallsyms])
    用户态堆栈：
#~        0x7f28339f9965 __pthread_cond_wait ([symbol])
#~        0x7f282bc82f25 _ZN2JS15PerfMeasurement19canMeasureSomethingEv ([symbol])
#~        0x7f282c080b9e _ZN2JS19PeakSizeOfTemporaryEPK9JSContext ([symbol])
#~        0x7f282c09fc02 _ZN2JS14AddServoSizeOfEP9JSContextPFmPKvEPNS_20ObjectPrivateVisitorEPNS_10ServoSizesE ([symbol])
#~        0x7f28339f5dd5 start_thread ([symbol])
#*        0xffffffffffffff JS Helper (UNKNOWN)
    进程链信息：
#^        0xffffffffffffff /usr/bin/gnome-shell  (UNKNOWN)
#^        0xffffffffffffff /usr/libexec/gnome-session-binary --session gnome-classic  (UNKNOWN)
#^        0xffffffffffffff gdm-session-worker [pam/gdm-password]  (UNKNOWN)
#^        0xffffffffffffff /usr/sbin/gdm  (UNKNOWN)
#^        0xffffffffffffff /usr/lib/systemd/systemd --switched-root --system --deserialize 22  (UNKNOWN)
##
线程详细信息： 4960
    时间：[1584776688:293352].
    进程信息： [/ / llvmpipe-0]， PID： 4935 / 4960
##CGROUP:[/]  4960      [014]  采样命中
    内核态堆栈：
```

注意：启动进程的父进程可能已经退出，这样有可能找不到直接启动进程的父进程。
同样的，可以从上面的输出结果中提取出火焰图。
如：
```
diagnose-tools task-info --tgid=$PID --report > task.log
diagnose-tools flame --input=task.log --output=task.svg
```
## 临时文件转火焰图
sys-delay / irq-delay / load-monitor / perf等功能都能够输出进程堆栈信息，可以将这些信息保存在临时文件中，例如tmp.txt中。
使用如下命令，可以将临时文件中的数据生成火焰图：

```
diagnose-tools flame --input=tmp.txt --output=perf.svg
```
该命令指定了数据来源文件为tmp.txt，并指定火焰图文件为perf.svg。成功后，可以使用浏览器直接打开perf.svg。如下所示：

你可以在浏览器中与火焰图互动：将鼠标移到不同层级的块中，看其详细信息，也可以点击块。
关于火焰图的说明，请参见：
http://www.ruanyifeng.com/blog/2017/09/flame-graph.html

## sys-delay
## sys-cost
## sched-delay
## irq-delay
## irq-stats
## load-monitor
## run-trace
## perf
## kprobe
## uprobe
## utilization
## exit-monitor
## mutex-monitor
## exec-monitor
## alloc-top
## high-order
## drop-packet
## tcp-retrans
## ping-delay
## rw-top
## fs-shm
## fs-orphan
## fs-cache
## reboot

# 测试命令
## test-md5
这是一个测试CPU速率的小工具。
使用方法：
```
diagnose-tools test-md5
```
该命令默认执行1000,0000次md5计算。
也可以使用-c参数指定计算次数，如：
```
[root@localhost diagnoise-tool]# diagnose-tools test-md5 -c 5000000
加密前:admin
加密后:21232f297a57a5a743894a0e4a801fc3
real	0m3.600s
user	0m3.581s
sys	0m0.006s
```

## test-pi
这是另一个测试CPU速率的小工具。
使用方法：

```
diagnose-tools test-pi
```
可以附带两个参数，其中：
-c --cpu可以指定将测试用例绑定到哪一个CPU上运行。
-v --verbose可以打开详细信息

##  test-memcpy
这是一个测试内存速率的小工具。
使用方法：
```
diagnose-tools test-memcpy
```
可以附带两个参数，其中：
-c --cpu可以指定将测试用例绑定到哪一个CPU上运行。
-v --verbose可以打开详细信息

## test-run-trace

## test-run-trace-java

## test-presure-java


# 实验版本的功能

目前，diagnose-tools-2.0实验版本有如下几个功能：
* kern-demo：展示如何在diagnose-tools中添加一个功能，供开发同学使用。
* sys-broken：监控系统调用被中断/软中断/定时器打断的时间。
* mm-leak：统计内核态一段时间内，分配了但是没有释放的内存。并输出分配这些内存的调用链，以及泄漏次数。

##  pupil小工具
无

## kern-demo
略
## mm-leak
[mm-leak](./mm-leak.md)

#  btrace和uprobe
##  btrace

btrace类似arthas，但是由于btrace可以定义脚本，所以在使用上相对arthas更加灵活，在某些arthas无法解决的场景，可以考虑使用btrace进行定位，其帮助文档位于:
https://github.com/btraceio/btrace/wiki/BTrace-Annotations?spm=ata.13261165.0.0.249b2086sNijQN
注意：本机需要指定JAVA_HOME环境变量，供btrace使用，同时btrace内部提供了比较多的samples脚本:
```
export JAVA_HOME=/opt/taobao/java/
```
## 示例：查看ThreadPoolExecutor初始化的堆栈
用法：
```
./bin/btrace 2184 ./samples/ThreadPoolExecutorInit.class  > /tmp/init.log
```
代码：
```
$more ThreadPoolExecutorInit.java
package samples;

import com.sun.btrace.BTraceUtils;
import com.sun.btrace.annotations.BTrace;
import com.sun.btrace.annotations.OnMethod;
import com.sun.btrace.annotations.ProbeClassName;
import com.sun.btrace.annotations.ProbeMethodName;

import static com.sun.btrace.BTraceUtils.println;

@BTrace public class ThreadPoolExecutorInit {


    @OnMethod(
            clazz = "java.util.concurrent.ThreadPoolExecutor",
            method = "<init>"
    )
    public static void logOnInit(@ProbeClassName String probeClass, @ProbeMethodName String probeMethod){
        println("==== " +  probeClass + " " + probeMethod);

        BTraceUtils.Threads.jstack();

        println("==== ================================");
    }

}
```
## 示例：btrace使用diagnose-tools脚本

在btrace中使用diagnose-tools脚本进行排查定位，在进入com.taobao.tair.comm.TairClientFactory.createClient 的时候开始开启btrace，在该接口返回后退出btrace
注意使用unsafe=true 才可以在btrace脚本中调用外部代码，同时指定需要将btrace 增加启动参数： -Dcom.sun.btrace.unsafe=true ， 开启非安全模式后，方可执行非安全脚本
```
./bin/btrace 3496 ./samples/BtraceMain.java
```
代码：
```
package com.sun.btrace.samples;

import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;

import com.sun.btrace.BTraceUtils;
import com.sun.btrace.annotations.*;

@BTrace(unsafe=true)
public class BtraceMain {

    @OnMethod(clazz = "com.taobao.tair.comm.TairClientFactory", method = "createClient")
    public static void start_run_trace() {
        FileOutputStream out = null;

        try {
            File file = new File("/proc/ali-linux/diagnose/kern/run-trace-settings");
            if (file.exists()) {
                out = new FileOutputStream("/proc/ali-linux/diagnose/kern/run-trace-settings");
                out.write("start\0".getBytes());
                out.write(0);
                out.write(System.getProperty("line.separator").getBytes());
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (out != null) {
                try {
                    out.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }

        BTraceUtils.print("enter");
    }
}
```
