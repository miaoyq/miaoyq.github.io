---
layout: article
title: "Dockerfile中Entrypoint指令使用方法分析"
date: 2017-03-07 14:40:43
categories: docker
toc: true
---

# Dockerfile中Entrypoint指令使用方法分析
[toc]

ENTRYPOINT的作用是设置容器运行时的执行命令，即容器的默认入口
ENTRYPOINT有两种使用格式
- **shell格式：** ENTRYPOINT command param1 param2
- **exec格式：**ENTRYPOINT ["executable", "param1", "param2"] (**推荐优先使用改格式**)

## 1、shell格式
命令和参数没有被中括号括起来，命令/参数之间用空格隔开，而不是逗号。

++在制作镜像过程中，如果使用shell格式，则`ENTRYPOINT`会**自动忽略**通过`CMD`指令和`docker run`命令输入的参数。++使用该方式制作的镜像，容器运行是的命令就固定死了，在运行时无法更改，即使在运行时添加了其他的命令，也会被或略。

shell格式有一个缺点，即ENTRYPOINT将以“/bin/sh -c”作为入口命令，也就是说用户指定执行实体不是作为容器的`PID 1`来运行。该命令不能传递信号量到子进程，因此当执行`docker stop [container]`命令时，执行实体无法接收到`SIGTERM`信号。这种情况下，docker结束执行实体的方法是，**等待超时**（即发送`SIGTERM`没响应）后，再发一条`SIGKILL`信号来强制kill掉用户指定执行实体。

下面举例说明使用shell格式的缺点，以及应对策略：
- Dockerfile文件内容如下，制作镜像`shell：v1`
```
FROM ubuntu
ENTRYPOINT top -b
```
- 运行容器，我们发现top并不是`PID 1`，而是`PID 6`
```bash
$ docker run -it --name shell-v1 shell:v1
top - 06:52:00 up 31 days,  6:13,  0 users,  load average: 0.00, 0.00, 0.00
Tasks:   2 total,   1 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  7.2 us, 24.4 sy,  0.0 ni, 68.4 id,  0.0 wa,  0.0 hi,  0.0 si,  0.1 st
KiB Mem :  8175676 total,  2553136 free,  1006204 used,  4616336 buff/cache
KiB Swap:  2095100 total,  2095100 free,        0 used.  6840964 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    1 root      20   0    4508    808    728 S   0.0  0.0   0:00.00 sh
    6 root      20   0   36532   3060   2704 R   0.0  0.0   0:00.00 top
```
- 计算结束容器的时间
```bash
$ time docker stop shell-v1
shell-v2
real	0m10.469s
user	0m0.016s
sys	0m0.004s
```
++通过上述时间可以看出，通过`docker stop`停止该容器用了 10s 多的时间++


为了在使用shell格式的情况下，用户进程也能通过`docker stop`命令正常结束，我们可以Dockerfile文件做一点微调，即在执行实体的前面加上`exec`命令
- Dockerfile文件内容
```
FROM ubuntu
ENTRYPOINT exec top -b
```
- 运行容器，发现top变成了`PID 1`
```bash
docker run -it --name shell-v1 shell:v1
top - 07:26:48 up 31 days,  6:47,  0 users,  load average: 0.02, 0.01, 0.00
Tasks:   1 total,   1 running,   0 sleeping,   0 stopped,   0 zombie
%Cpu(s):  7.2 us, 24.3 sy,  0.0 ni, 68.4 id,  0.0 wa,  0.0 hi,  0.0 si,  0.1 st
KiB Mem :  8175676 total,  2560696 free,   996316 used,  4618664 buff/cache
KiB Swap:  2095100 total,  2095100 free,        0 used.  6850660 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    1 root      20   0   36532   3064   2704 R   0.0  0.0   0:00.00 top
```
- 计算结束容器的时间
```bash
$ time docker stop shell-v1
shell-v1
real	0m0.540s
user	0m0.016s
sys	0m0.004s
zte@ubuntu:~$
```
++发现`docker stop`命令的执行时间明显减少（0.54s），这里的头号功臣是`exec`命令。++
<br\>
系统调用`exec`是以新的进程去代替原来的进程，但进程的PID保持不变。因此，可以这样认为，exec系统调用并没有创建新的进程，只是替换了原来进程上下文的内容。原进程的代码段，数据段，堆栈段被新的进程所代替。因此在容器运行起来后，`top`进程会替换`/bin/sh -c`进程，成为`PID 1`进程。

## 2、exec格式
命令和参数都有`"`，且之间用`,`隔开，整体用中括号括起来。

**在制作镜像是，使用了exec格式，则系统会组合`ENTRYPOINT`和`CMD`两个命令，作为容器运行时的默认执行命令，即`ENTRYPOINT`+`CMD`**。该方式下制作的镜像`ENTRYPOINT`指令指定的内容是**固定不变**的，运行时无法修改，而`CMD`指定的部分是可变的，在`docker run`的时候，增加的参数会覆盖`CMD`指定的参数。

这里的使用场景，应该也比较好理解。镜像中不变的命令放在`ENTRYPOINT`指令中，可能变化的指令或者参数放在`CMD`指令中。

另外，用exec格式制作的镜像，容器运行时的`PID 1`一定是`ENTRYPOINT`指定的运行实体，而不是`/bin/sh -c`

下面距离说明：
- 镜像制作的Dockerfile文件内容，制作exec:v1镜像
```
exec格式的ENTRYPOINT举例：
FROM ubuntu
ENTRYPOINT ["top", "-b"]
CMD ["-c"]
```
- 默认命令运行容器，我们发现top是`PID 1`，且命令为`top -b -c`（即`ENTRYPOINT`+`CMD`）
```bash
$ docker run -it --rm --name exec-c1  exec:v1
top - 08:18:16 up 31 days,  7:39,  0 users,  load average: 0.05, 0.01, 0.00
Tasks:   1 total,   1 running,   0 sleeping,   0 stopped,   0 zombie
%Cpu(s):  7.2 us, 24.3 sy,  0.0 ni, 68.4 id,  0.0 wa,  0.0 hi,  0.0 si,  0.1 st
KiB Mem :  8175676 total,  2538936 free,  1015276 used,  4621464 buff/cache
KiB Swap:  2095100 total,  2095100 free,        0 used.  6831488 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    1 root      20   0   36532   3064   2704 R   0.0  0.0   0:00.01 top -b -c
```

- `docker run`命令中增加参数`-H`，发现`PID 1`的命令变为`top -v -H`，即运行时输入的参数把`CMD`指令指定的参数覆盖了
```bash
    docker run -it --rm --name exec-c2  exec:v1 -H
    top - 08:20:46 up 31 days,  7:41,  0 users,  load average: 0.00, 0.00, 0.00
    Threads:   1 total,   1 running,   0 sleeping,   0 stopped,   0 zombie
    %Cpu(s):  7.2 us, 24.3 sy,  0.0 ni, 68.4 id,  0.0 wa,  0.0 hi,  0.0 si,  0.1 st
    KiB Mem :  8175676 total,  2535600 free,  1018364 used,  4621712 buff/cache
    KiB Swap:  2095100 total,  2095100 free,        0 used.  6828368 avail Mem 

      PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
        1 root      20   0   36532   3072   2708 R  0.0  0.0   0:00.00 top

    $ docker exec -it exec-c2 ps -ef
    UID        PID  PPID  C STIME TTY          TIME CMD
    root         1     0  0 08:22 ?        00:00:00 top -b -H
    root         7     0  0 08:22 ?        00:00:00 ps -ef
```

## 3、不设置`ENTRYPOINT`的情况
通过上面两种情况的介绍，可能有人会问，上面两种情况，当默认命令设置好后，都不能修改，那如果我们需要把**可执行实体和参数**都设置为运行时变的，该怎么办呢？这时候我们可以不设置`ENTRYPOINT`指令，而把执行的命令和参数都设置在`CMD`指令中，这样就可以修改了。


## 4、总结`ENTRYPOINT`和`CMD`指令不同组合的用法
`CMD`指令也分`exec`和`shell`两种格式，跟`ENTRYPOINT`类似，这里就不再赘述了。

`CMD`和`ENTRYPOINT`都可以用来指定容器运行时的默认执行命令，下面是使用这两个字段的几条规则：
1. 在镜像制作中，docker文件中至少需要这只`CMD`和`ENTRYPOINT`中的一个
2. `CMD`可以作为`ENTRYPOINT`字段的补充命令，为`ENTRYPOINT`中的可执行命令设置默认参数；也可以单独设置容器执行的命令和参数。
3. **当容器运行时设置了命令参数，则`CMD`字段中的内容将被覆盖。**
4. 使用shell格式时，`ENTRYPOINT`指令会忽略任何`CMD`和`docker run`命令的参数，并会运行在`/bin/sh -c`中（即用户进程的父进程是`/bin/sh -c`）

下表给出了`ENTRYPOINT`和`CMD`配合使用时各种场景：

|                                | No ENTRYPOINT              | ENTRYPOINT exec_entry p1_entry | ENTRYPOINT ["exec_entry", "p1_entry"]          |
|--------------------------------|----------------------------|--------------------------------|------------------------------------------------|
| **No CMD**                     | *error, not allowed*       | **/bin/sh -c** exec_entry p1_entry | exec_entry p1_entry                            |
| **CMD ["exec_cmd", "p1_cmd"]** | exec_cmd p1_cmd            | **/bin/sh -c** exec_entry p1_entry | exec_entry p1_entry exec_cmd p1_cmd            |
| **CMD ["p1_cmd", "p2_cmd"]**   | p1_cmd p2_cmd              | **/bin/sh -c** exec_entry p1_entry | exec_entry p1_entry p1_cmd p2_cmd              |
| **CMD exec_cmd p1_cmd**        | **/bin/sh -c** exec_cmd p1_cmd | **/bin/sh -c** exec_entry p1_entry | exec_entry p1_entry **/bin/sh -c** exec_cmd p1_cmd |

