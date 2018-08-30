需要解决的问题：
1. 如何在qemu中添加一个新的guest构架：在/target/下新建转换为tcg的代码
    a. 如何让qemu加载添加的新构架
    b. 如何配置configure，可以正确的编译新增代码
1. 如何使用qemu模拟执行guest代码



# 编译安装
linux安装指导：https://wiki.qemu.org/Hosts/Linux

## 代码下载
下载页面：https://www.qemu.org/download/

git地址：https://git.qemu.org/?p=qemu.git， 或者：https://github.com/qemu/qemu

直接下载源码包编译会提示/tests等目录无法找到，需要从git clone

```shell
git clone git://git.qemu-project.org/qemu.git
```

还需要执行`git submodule update`下载所有的git子模块

## 安装依赖包
代码下好后安装依赖包（ubuntu环境）
```shell
sudo apt-get install git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev

sudo apt-get install libnfs-dev libiscsi-dev
```

## 代码编译

```shell
cd qemu
mkdir -p build
cd build
../configure --target-list=x86_64-softmmu --enable-debug # 只编译x86版本测试，否则超级慢
```
编译完成后运行
```
./build/x86_64-softmmu/qemu-system-x86_64 -L pc-bios
```
console显示`VNC server running on 127.0.0.1:5900`，在浏览器中打开，可以看见运行结果（一行文本）

# 代码构架
主要翻译自qemu detailed study，只有第7章，其他章节没找到。这篇文章应该是基于v0.13.x版本写的，和现有的构架有很多不同

其他资料：
[各种姿势折腾 QEMU](https://blog.csdn.net/kids412kelly/article/details/52509670)

[qemu源码架构](https://blog.csdn.net/dj0379/article/details/54926443)

[QEMU开发新的架构一](https://blog.csdn.net/ivanx_cc/article/details/46122783)

[QEMU代码分析（1）－module_init()构造函数](https://blog.csdn.net/miaohongyu1/article/details/25954005)

[关于qemu的二三事（5）————qemu源码分析之参数解析](https://blog.csdn.net/Benjamin_Xu/article/details/72824904)



## 代码基本构架
QEMU模拟的架构叫`目标架构target`，运行 QEMU的系统架构叫`主机架构host`，QEMU中有一个模块叫做`微型代码生成器（TCG）`，它用来将目标代码翻译成主机代码。

![avatar](构架.jpg)

我们将运行在虚拟cpu上的代码叫做客户机代码(guest code)，QEMU的主要功能就是不断提取客户机代码并且转化成主机指定架构的代码(host code)。整个翻译任务分为两个部分：第一个部分是将做目标代码（TB）转化成TCG中间代码，然后再将中间代码转化成主机代码。

1. 程序入口。主要的文件包括/vl.c,/cpus.c, /execall.c, /exec.c, /cpu-exec.c。`main`函数在/vl.c中，这个文件中的其他函数设置了虚拟机的其他参数，例如ram大小，cpu数量等。在虚拟机设置完成后，`main`函数会调用其他文件中的函数例如/cpus.c, /exec-all.c, /exec.c, /cpu-exec.c（execution branches out through files such as /cpus.c, /exec-all.c, /exec.c, /cpu-exec.c.） （main函数在2910行）
 
1. 硬件模拟。所有模拟虚拟硬件的代码都放在/hw/中。
 
1. Guest (Target) Specific: 现在QEMU中已经模拟的处理器结构包括Alpha, ARM, Cris, i386, M68K, PPC, Sparc, Mips, MicroBlaze, S390X and SH4。将这些结构的TB转换为TCG操作的代码实现在/target/arch/中（添加新的结构应该就在这里），例如i386构架的代码在/target/i386/。这部分代码教程TCG前端（frontend of TCG）.。
 
1. Host (TCG) Specific: 从TCG操作中生成主机代码的代码在/tcg/host_arch/中，例如i386构架的代码在/tcg/i386/。这部分代码叫做TCG后端（backend of TCG）。 

1. 总结。
```
/vl.c : 主模拟器循环，虚拟机设置和CPU执行（The main emulator loop, the virtual machine is  setup and CPUs are executed）。
 
/target/xyz/translate.c : 将客户端代码（客户端指令集）转换成与指令集构架无关的TCG操作。
 
/tcg/tcg.c : TCG的主循环.  
 
/tcg/*/tcg-target.c :  将TCG操作转换为本机ISA代码。 
 
/cpu-exec.c :  函数cpu-exec()寻找下一个TB，如果没有找打，则生成未找到TB的信号来产生一个新TB，最后执行生成的代码（应该是从TB->TCG，执行的是TCG？）。 cpu-exec() finds the next translation block (TB), if not found calls are made to generate the next TB  and finally to execute the generated code. 
```

## TCG-动态翻译
QEMU v0.9.1版本之前动态翻译都是由DynGen完成的。TB被DynGen转换为C代码，再由GCC将C代码转换为主机代码(host specific code)。为了解除与GCC的紧密联系，产生了一种新机制:TCG。

动态翻译使得代码再需要时才被转换。这个思想的主要目的是用最多的时间去执行生成后的代码而不是去生成代码(The idea was to spend the maximum time executing the generated code that executing the code generation)。每当从TB转换为代码（应该指的是TCG）后，这些代码会再执行前先被存储起来。多数时候相同的TB会被多次调用，这样通过本地引用（Locality Reference）可以重复使用之前转换好的代码。当指令缓存(code cache)填满时，整个缓存会被清空而不是使用LRU算法（least recently used，缓存淘汰算法）。

![avatar](构架2.png)

在执行前编译器从源代码(source code)中生成结果代码(object code)。为了生成一个函数调用的结果代码，编译器（例如GCC）会在函数调用之前和之后插入一些特殊的汇编码（assembly code），这些汇编码称作函数序曲和尾声(Function Prologue and Epilogue)。

如果体系结构（应该指的是target的结构）有一个基指针和一个栈指针，则Function Prologue通常执行以下操作：
 
1. 将当前基指针压入栈中，以便后续恢复。
1. 将旧的基指针替换为当前栈指针，这样新栈会在旧栈的顶端产生。（应该是将基指针指向栈顶） Replaces the old base pointer with the current stack pointer such that the a new stack will be created on top of the old stack. 
1. 将栈指针向当前栈顶移动，给函数中的局部变量在栈中腾出存储空间。Moves the stack pointer further along the stack to make room in the current stack frame for the function's local variables. 

Function Epilogue恢复 function prologue执行的操作，并将控制权交会调用它的函数（ and returns control to the calling function，应该是只qemu的CPU调用循环）。它通常执行以下操作: 

1. 将栈指针替换为当前基指针，这样栈指针就恢复成prologue之前的值。
1. 将之前的基指针出栈，这样基指针就恢复成prologue之前的值。
1. 弹出之前的程序指针并跳转，回到之前调用的函数。（Returns to the calling function, by popping the previous frame's program counter off the stack and jumping to it）

TCG可以被看作一个事实生成结果代码的编译器。通过TCG生成的代码存储在缓存(code buffer)中，通过TCG的 Prologue和Epilogue功能来执行code buffer中的代码或者从中跳出（The execution control is passed to and from the code cache through TCG’s very on Prologue and Epilogue）。执行的流程见下图

![avatar](构架3.png)

下面4幅图介绍了TCG是如何工作的：

![avatar](构架4.png)
![avatar](构架5.png)
![avatar](构架6.png)
![avatar](构架7.png)

## TB链（Chaining of TBs）
从code cache返回到静态代码（QEMU程序），或者跳转到code cache通常都十分缓慢。为了解决这一问题，QEMU将每一个TB都链接到下一个TB。这样在执行完一个TB后会直接执行下一个TB而不是返回静态代码（QEMU程序）。当不存在链接(no chaining)的TB1执行完返回静态代码后，紧接着发现、转换、执行了TB2，那么当TB2返回时就会被自动的链接到TB1上。这样下次TB1执行完成后就会直接执行TB2，而不返回静态代码。如下图所示。

![avatar](TB_chain.png)

## 执行过程(Execution trace)

本节会追踪QEMU的执行过程，并着重指出被调用函数的生明位置和所在文件。本节主要介绍TCG部分。(This section will focus mainly on the TCG part of QEMU and will thus be key in finding the code sections that generate the Host code. A good understanding of code generation in QEMU will be necessary to help patch up QEMU in order to make the EVM.)

本节是代码基本构架一节的扩充。

1. main(..){/vl.c}: main函数解析命令行输入参数，本根据参数设置虚拟机(VM)，例如ram，磁盘大小，启动盘等。当VM设置完成后，main()调用main_loop()。

1. main_loop(...){/vl.c}: [Function main_loop initially calls qemu_main_loop_start() and then does infinite looping of cpu_exec_all() and profile_getclock() within a do-while for which the condition is vm_can_run(). The infinite for-loop continues with checking some VM halting situations like qemu_shutdown_requested(), qemu_powerdown_requested(), qemu_vmstop_requested() etc. These halting conditions will not be investigated further.] v3.0已经不是这个结构，
    ``` C
    static void main_loop(void)
    {
    #ifdef CONFIG_PROFILER
        int64_t ti;
    #endif
        while (!main_loop_should_exit()) {
    #ifdef CONFIG_PROFILER
            ti = profile_getclock();
    #endif
            main_loop_wait(false);
    #ifdef CONFIG_PROFILER
            dev_time += profile_getclock() - ti;
    #endif
        }
    }
    ```
    1. ti应该是内部时间，
    1. main_loop_should_exit()检查是否退出循环，main_loop_should_exit()中检查了runstate_check(),qemu_debug_requested(),qemu_suspend_requested(),qemu_shutdown_requested(),qemu_kill_report(),qapi_event_send_shutdown()...等信号
    1. profile_getclock{/include/qemu/timer.h}, 和profile计时有关
    1. main_loop_wait(){/include/qemu/main-loop.h,/util/main-loop.c}是循环执行内容的主题(Run one iteration of the main loop)。
    调用函数
        - main_loop_wait()
            - g_array_set_size() 未找到定义位置，g_代表全局函数？，有很多g_array_xxx()，处理数组用的？
            - slirp_pollfds_fill() {slirp/libslirp.h, slirp/slirp.c}
            - qemu_soonest_timeout() {/include/qemu/timer.h} Calculates the soonest of two timeout values. -1 means infinite, which is later than any other value.
                - timerlistgroup_deadline_ns() {/include/qemu/timer.h} Determine the deadline of the soonest timer to expire associated with any timer list linked to the timer list group. Only clocks suitable for deadline calculation are included.
            - os_host_main_loop_wait(){/util/main-loop.c} 
            - slirp_pollfds_poll()
            - qemu_start_warp_timer() {/cpus.c} 
            - qemu_clock_run_all_timers() {/include/qemu/timer.h} Run all the timers associated with the default timer list of every clock.

1. cpu_exec(...){/accel/tcg/cpu-exec.c}主要执行过程，找不到和main_loop()之间是如何调用的
    可能的调用层次
    qemu_init_vcpu
        qemu_tcg_init_vcpu 
            qemu_tcg_cpu_thread_fn 多线程tcg
            qemu_tcg_rr_cpu_thread_fn 单线程tcg
                tcg_cpu_exec
                    cpu_exec 主要执行过程

    - struct CPUState{/include/qom/cpu.h} cpu_exec()的参数，在{/target/xxx/cpu.h}中还有一个类似
        ``` C
        typedef struct MoxieCPU {
            /*< private >*/
            CPUState parent_obj;
            /*< public >*/

            CPUMoxieState env;
        } MoxieCPU;
        typedef struct CPUMoxieState {
            //...
        } CPUMoxieState;
        ```
        的结构体，好像是各个CPU的单独实现
    - v0.13的代码解析：Function cpu_exec is referred to as the ‘main execution loop’. Here for the first time a translation Block TB is initialized (TranslationBlock *tb) the code then basically continues with handling exceptions. Deep within two nested infinite for-loops one can find tb_find_fast() and tcg_qemu_tb_exec(). tb_find_fast() initiates the search for the next TB for the Guest and then generate the Host code. The generated Host code is then executed through tcg_qemu_tb_exec().

    - 和v0.13类似，处理中断还有其他，然后到
        ``` C
            while (!cpu_handle_exception(cpu, &ret)) {
                TranslationBlock *last_tb = NULL;
                int tb_exit = 0;

                while (!cpu_handle_interrupt(cpu, &last_tb)) {
                    uint32_t cflags = cpu->cflags_next_tb;
                    TranslationBlock *tb;

                    /* When requested, use an exact setting for cflags for the next
                    execution.  This is used for icount, precise smc, and stop-
                    after-access watchpoints.  Since this request should never
                    have CF_INVALID set, -1 is a convenient invalid value that
                    does not require tcg headers for cpu_common_reset.  */
                    if (cflags == -1) {
                        cflags = curr_cflags();
                    } else {
                        cpu->cflags_next_tb = -1;
                    }

                    tb = tb_find(cpu, last_tb, tb_exit, cflags);
                    cpu_loop_exec_tb(cpu, tb, &last_tb, &tb_exit);
                    /* Try to align the host and virtual clocks
                    if the guest is in advance */
                    align_clocks(&sc, cpu);
                }
            }
        ```
        这里开始处理TB
        - tb_find()
        - cpu_loop_exec_tb()

1. struct TranslationBlock{/include/exec/exec-all.h} TB定义

