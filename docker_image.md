描述 docker 镜像相关内容。

# 使用

- 查找镜像

docker search name

- 拉取镜像

docker pull name:version

- 构建镜像

两种构建方式：
    - 基于容器

        容器运行完毕后退出，然后使用 `docker commit -m="description" -a="author" container-id image_name:version`

    - 基于 dockerfile

        dockerfile 的内容会单独描述。

- 删除镜像

docker image rm -f name

- 查看镜像详情

docker image inspect name

> 镜像本质是个 json 文件，里面记录了对应的 layer，以及 WorkDir、Env、EntryPoint、Volumes 内容。

# 原理

镜像本身分为多个层，每层代表的就是文件系统的变更。当 docker 通过一个镜像拉起一个容器的时候，本质就是将镜像的这几层文件系统的变更通过 union file system 的方式，挂载到一个指定的目录，这个目录就是容器进程的 **/** 目录。<br>

同时，docker 会为容器本身创建一个可写的 layer，容器本身针对文件系统的所有变更，都会写到这个 layer，镜像本身关联的其他 layer 是只读的。

## overlay 联合文件系统

在我的 ubuntu 中，docker 使用的 ufs 是 overlay。当我们拉取一个镜像后，就会在 `/var/lib/docker/overlay2`创建若干个目录，每个目录名是镜像的每个层，在每个目录的 diff 子目录中，存放的就是该层的变更。


当我们拉起一个容器的时候，docker 会额外创建两层，一层用来存放容器对文件系统的变更，另一层用也用来存放一些文件系统变更内容，但是这些变更不作为镜像一部分，当我们创建镜像的时候，会忽略这一层。此外，docker 还会创建一个挂载点，会将所有层对应的目录挂载到一个指定目录，这个目录就是 docker 容器的更目录。

```sh
cat < /proc/mounts | grep overlay

overlay /var/lib/docker/overlay2/xxxx/merged(挂载点) overlay lowerdir=/var/lib/docker/overlay2/xxx(init 层):/var/lib/docker/overlay2/xxx(镜像对应的层) upperdir=/var/lib/docker/overlay2/xxx(容器的可写层)/diff
```
当我们在容器中变更文件系统后，对应的内容就会生成到 upperdir 目录中。而 merged 目录是所有层的 merge，上层覆盖下层。

## layer

怎么理解 layer 呢？有什么优势呢？

如果没有 layer，镜像是如何在用户之间共享的呢？用户 A 打包一个镜像，然后分享给用户 B，用户 B 基于这个镜像又创建了一个新的镜像，这两个镜像实际上大部分内容都是相同的。但现在在新旧镜像中对于相同的变更维护了两份内容。这导致的问题就是，给其他用户分享的时候，需要下载整个完整的镜像，效率低，又占用存储空间。<br>

layer 就是用来解决此问题的，每一个 layer 只保留增量的变更，这样底层的 layer 都可以直接复用，在分享、打包很多场景下，都可以提高效率。



