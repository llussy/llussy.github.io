---
layout: post
title: "Dockerfile中CMD和ENTRYPOINT"
date: 2019-08-14 12:10:30 +0800
catalog: ture  
multilingual: false
tags: 
    - kubernetes
    - docker
---

> 主要参考sparkdev [地址](https://www.cnblogs.com/sparkdev/p/8461576.html)

CMD 和 ENTRYPOINT 指令都是用来指定容器启动时运行的命令。

Docker 的 stop 和 kill 命令都是用来向容器发送信号的。注意，只有容器中的 1 号进程能够收到信号，这一点非常关键！

stop 命令会首先发送 SIGTERM 信号，并等待应用优雅的结束。如果发现应用没有结束(用户可以指定等待的时间`k8s 默认等待30s`)，就再发送一个 SIGKILL 信号强行结束程序。
kill 命令默认发送的是 SIGKILL 信号，当然你可以通过 -s 选项指定任何信号。

**容器中的 1 号进程是非常重要的，如果它不能正确的处理相关的信号，那么应用程序退出的方式几乎总是被强制杀死而不是优雅的退出。究竟谁是 1 号进程则主要由 EntryPoint, CMD, RUN 等指令的写法决定，所以这些指令的使用是很有讲究的。**

### CMD 指令
CMD 指令的目的是：为容器提供默认的执行命令。

```bash
CMD 指令有三种使用方式，其中的一种是为 ENTRYPOINT 提供默认的参数
CMD ["param1","param2"]

另外两种使用方式分别是 exec 模式和 shell 模式：
CMD ["executable","param1","param2"]  #这是 exec 模式的写法，注意需要使用双引号。
CMD command param1 param2*    # 这是 shell 模式的写法。
```



### ENTRYPOINT 指令

ENTRYPOINT 指令的目的也是为容器指定默认执行的任务。

```bash
ENTRYPOINT 指令有两种使用方式，就是我们前面介绍的 exec 模式和 shell 模式：
ENTRYPOINT ["executable", "param1", "param2"] # 这是 exec 模式的写法，注意需要使用双引号。

ENTRYPOINT command param1 param2    # 这是 shell 模式的写法。
exec 模式和 shell 模式的基本用法和 CMD 指令是一样的.
```



### 同时使用 CMD 和 ENTRYPOINT 的情况

对于 CMD 和 ENTRYPOINT 的设计而言，多数情况下它们应该是单独使用的。当然，有一个例外是 CMD 为 ENTRYPOINT 提供默认的可选参数。

我们大概可以总结出下面几条规律:

• 如果 ENTRYPOINT 使用了 shell 模式，CMD 指令会被忽略。

• 如果 ENTRYPOINT 使用了 exec 模式，CMD 指定的内容被追加为 ENTRYPOINT 指定命令的参数。

• 如果 ENTRYPOINT 使用了 exec 模式，CMD 也应该使用 exec 模式。
真实的情况要远比这三条规律复杂，好在 docker 给出了官方的解释，如下图所示：

![image-20190308233150145](https://llussy.github.io/images/kubernetes/cmd.png)



exec 模式是建议的使用模式，因为当运行任务的进程作为容器中的 1 号进程时，我们可以通过 docker 的 stop 命令优雅的结束容器。

**exec 模式的特点是不会通过 shell 执行相关的命令，所以像 $HOME 这样的环境变量是取不到的**：

```bash
FROM ubuntu
CMD [ "echo", "$HOME" ] 
```

通过 exec 模式执行 shell 可以获得环境变量：

```bash
FROM ubuntu
CMD [ "sh", "-c", "echo $HOME" ]
```


### 参考

[在 docker 容器中捕获信号](https://www.cnblogs.com/sparkdev/p/7598590.html)

[Dockerfile 中的 CMD 与 ENTRYPOINT](https://www.cnblogs.com/sparkdev/p/8461576.html)

