
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
../configure --target-list=x86_64-softmmu --enable-debug # 只编译x86版本，否则超级慢
```
编译完成后运行
```
./build/x86_64-softmmu/qemu-system-x86_64 -L pc-bios
```
如果如果成功，则显示`VNC server running on 127.0.0.1:5900`，在浏览器中打开，可以看见运行结果

# 代码构架
## 参考资料
[各种姿势折腾 QEMU](https://blog.csdn.net/kids412kelly/article/details/52509670)
[qemu源码架构](https://blog.csdn.net/dj0379/article/details/54926443)
[QEMU开发新的架构一](https://blog.csdn.net/ivanx_cc/article/details/46122783)
[QEMU代码分析（1）－module_init()构造函数](https://blog.csdn.net/miaohongyu1/article/details/25954005)
[关于qemu的二三事（5）————qemu源码分析之参数解析](https://blog.csdn.net/Benjamin_Xu/article/details/72824904)



## 构架
QEMU模拟的架构叫`目标架构target`，运行 QEMU的系统架构叫`主机架构host`，QEMU中有一个模块叫做`微型代码生成器（TCG）`，它用来将目标代码翻译成主机代码。

![avatar](构架.jpg)

我们将运行在虚拟cpu上的代码叫做客户机代码(guest code)，QEMU的主要功能就是不断提取客户机代码并且转化成主机指定架构的代码(host code)。整个翻译任务分为两个部分：第一个部分是将做目标代码（TB）转化成TCG中间代码，然后再将中间代码转化成主机代码。





主要文件：
/vl.c: 它也是执行的起点，这个函数的功能主要是建立一个虚拟的硬件环境。它通过参数的解析，将初始化内存，需要的模拟的设备初始化，CPU参数，初始化KVM等等。接着程序就跳转到其他的执行分支文件如：/cpus.c, /exec-all.c, /exec.c, /cpu-exec.c。
/cpus.c: 