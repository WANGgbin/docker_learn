描述 dockerfile 相关知识。

# ARG 与 ENV 的区别

ARG 构建镜像过程中的变量。<br>

ENV 环境变量。<br>

两者都可以通过 $NAME 的方式来引用。

# COPY 与 ADD 的区别

COPY 就是从上下文中拷贝内容到镜像(目的目录不存在会自动创建)，需要注意的是：copy 内容的属性保持不变。<br>

ADD 命令的功能不是很清晰，不建议使用。

# CMD 与 ENTRYPOINT 的区别
使用一个镜像创建容器的时候，这个容器执行什么命令呢？这就是 CMD 和 ENTRYPOINT 存在的意义。<br>

两者都有两种格式，指定要执行的命令：

- shell 格式
  
    CMD echo "hello docker"

    等价于 CMD ["/bin/bash", "-c", "echo", "hello docker"]

    shell 格式可以**引用环境变量**

- exec 格式 

    CMD ["echo", "hello docker"]

    exec 格式**无法引用环境变量**

建议使用 exec 格式，因为更加清晰，而且 ENTRYPOINT shell 格式不能引用 CMD 指定的参数。

## CMD

可以通过 docker container run 的时候覆盖 CMD 指定的默认值。

## ENTRYPOINT

docker run 不能覆盖 ENTRYPOINT 的默认值。有了 CMD 为什么还要 ENTRYPOINT 呢？一个典型的场景式**可变参数**。<br>

如果要支持可变参数场景，使用 CMD 能实现吗？可以，不过，需要我们每次在 docker run 的时候要指定完整的命令(cmd + 可变参数)，但是这样即繁琐，dockerfile 中 CMD 也就没有了意义，而如果我们在 docker run 中只指定可变参数，因为覆盖特性，docker 会**认为这些可变参数才是要执行的命令**<br>

因此引入了 ENTRYPOINT，在 ENTRYPOINT 存在的前提下，CMD 指定的内容会被当做命令行参数而不是命令。这样 CMD 和 ENTRYPOINT 结合使用就可以实现变参的目的。不过需要注意的式，ENTRYPOINT 格式会忽略 CMD 指定的参数<br>

另一个场景式，我们在执行命令前，经常需要做一些初始化工作，这个时候，就可以通过 ENTRYPOINT 来指定一个 sh 脚本结合 CMD 参数来完成. <br>

参考：https://zhuanlan.zhihu.com/p/695490907

# 上下文是什么

在 docker.md 中我们描述了 docker 的架构。在 dockerd 创建镜像的过程中，可能涉及 COPY 指定，需要从指定目录拷贝内容到镜像中，而 dockerd 是感知不到 src 目录的，因此，需要在 docker cli 中指定一个目录，docker cli 会打包该目录同时发送给 dockerd。而这个目录就是上下文。<br>

注意：dockerfile 所在的目录跟上下文目录没有任何关系。唯一的联系是：如果没有通过 -f 指定 dockerfile，默认取上下文中的 dockerfile。<br>

一个最佳实践是：上下文目录中只存储必要的内容。因此跟 .gitignore 我们可以在上下文目录中创建一个 .dockerignore 文件指定需要忽略的内容，这样，在 docker cli 打包的时候，就会忽略这些内容。<br>

参考：https://zhuanlan.zhihu.com/p/670003782

# dockerfile 构建镜像的原理

在 dockerfile 之前，我们是怎么构建镜像的呢？

- 基于镜像拉起一个 container
- 变更文件系统
- 退出 container
- docker container commit 

实际上，dockerfile 构建镜像的过程，跟前面的步骤是一样的。对于 **dockerfile 中每一条指令**重复上述步骤，构建出最终的镜像。但是并不是每一条指令都涉及文件系统的变更，只有涉及文件系统变更的指令比如：WORKDIR(如果目录不存在会自动创建)、RUN 才会生成新的 layer，像 ENV 指令并不涉及文件系统的变更，也就不会创建 layer。

所以一个最佳实践是，我们尽量要让镜像的层少且每一层只引入必要的内容。对于相关的命令，我们可以放在一个 RUN 命令中执行，对于不必要的内容，还需要在 RUN 命令中删除。

参考：https://zhuanlan.zhihu.com/p/670003782

# WORKDIR

指定后面各层的工作目录，如果目录不存在，会自动创建。当然，我们可以指定多个 WORKDIR。最后容器的开始目录以 **最后一个 WORKDIR 指定的目录为准**，执行 ENTRYPOINT/CMD 指定的命令的相对路径，就是相对于容器的工作目录的。

# 多阶段构建

将镜像的构建过程分为多个阶段，每个阶段通过 FROM 开始。最终，只有最后一个阶段的内容会被合并到镜像中。
参考：https://zhuanlan.zhihu.com/p/697432802 中关于多阶段的描述。

# 最佳实践

参考：https://zhuanlan.zhihu.com/p/697432802

# 资料

- [使用 Dockerfile 定制镜像](https://zhuanlan.zhihu.com/p/670003782)