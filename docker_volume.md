参考：https://zhuanlan.zhihu.com/p/686938176

# 目的

我们知道容器产生的数据是存放在自己的可读写层的，当容器进程退出的时候，对应的可读写层也就消失了。那如果我们想保留容器生成的数据怎么办呢？<br>

这就是 volume 要解决的问题。

# 使用

有两种类型的 volume:

- 命名 volume

```sh
docker run -v volume_name:/path/in/container ...
```

上述命令就指定了从 volume_name 到 /path/in/container 的映射，当我们在容器中对 /path/in/container 修改的时候，实际修改的是 volume_name 对应的宿主机 path。如果 volume_name 不存在的话，上述命令会自动创建 volume_name，每个 volume_name 对应的宿主机中的路径，我们可以通过 `docker volume inspect volume_name` 查看。

- 直接指定宿主机路径

另一种方式就是直接指定宿主机路径(绝对/相对路径)。命令如下：

```sh
docker run -v /path/in/host:/path/in/container ...
```

# 原理

那么 volume 实现的原理是什么呢？<br>

我们知道每个容器有自己的 mount namespace，实际上在拉起容器进程的时候，先创建一个新的 mount namespace，**这个 mount namespace 是源 namespace 的拷贝，此时还没有通过 chroot 修改根目录，然后将宿主机中的目录挂载到容器对应的目录中，命令类似： mount /path/in/host /var/lib/docker/xxx/merged/path/in/container**，最后再 chroot。<br>

这样容器对应 path 的变更，实际更改的是宿主机对应的目录。