描述 docker 容器相关内容。


# 使用

- 创建容器

docker container run -it image cmd

- 查看容器

docker container ls // 查看运行中的容器
docker container ls -a // 查看所有容器

- 停止容器

docker container stop container_id

- 启动容器

docker container start container_id

- 移除容器

docker container rm container_id

- 移除所有停止的容器

docker container prune -f

- docker attach 与 docker exec 区别

docker container exec 是在 docker 容器中新拉起一个进程，退出后，因为 1 号进程还在运行，所以容器并不会结束。<br>

一个好的最佳实践是：尽量使用 docker exec，避免 docker 的退出。

# 原理

## PID Namespace

每个容器有自己的 PID Namespace，容器中的进程只能看到这个命名空间中的进程。容器中的进程**在宿主机中也是可见的，在宿主机的角度看，就是一些普通进程而已。**

## 主进程

我们说容器就是个进程，这里的进程指的就是主进程(PID=1)，也就是通过 `docker run` 创建的进程。实际上，我们还可以通过 `docker exec container_name cmd` 在容器中创建新的进程。<br>

容器的生命周期跟主进程一样，当主进程结束的时候，容器也就退出了。<br>

举个例子：

首先，我们通过 docker run -it /bin/bash 创建一个容器，然后再通过 docker exec 在容器中创建一个新的进程。然后我们在容器中，通过 `ps -efj` 查看容器中所有的进程：

```sh
UID     PID     PPID    CMD
root    1       0       /bin/bash
root    13      0       /bin/bash
root    22      13      ps -efj
```

我看可以看到有三个进程。然后我们在宿主机执行 `ps -efj | grep /bin/bash` 看下结果：
```sh
UID     PID     PPID    CMD
root    121717  121685  /bin/bash
root    121847  121685  /bin/bash
```

我们可以看到在宿主机视角看，进程号不同。另外，我们发现，这两个进程的父进程一致，表明都是由某个进程 fork 出来的，我们看看这个进程：

```sh
UID     PID     PPID    CMD
root    780     1       /usr/bin/containerd
root    121685  780     containerd-shim
```

什么是 containerd-shim 进程呢？

每当我们创建一个新的容器的时候，containerd 守护进程就会创建一个 containered-shim 进程，该进程专门用来管理对应的容器，包括启动、暂停、退出、创建新的进程的。当接受 docker container exec 命令的时候，shim 会拉起一个新进程，并加入到容器对应的 pid namespace 中。


## mount namespace

每个 namespace 有自己的挂载信息，互相隔离。关于 mount namespace 注意以下几点：

- 新的 mount namespace 会拷贝当前的 mount namespace
源代码如下：
```c
int copy_namespace(int flags, struct task_struct *tsk)
{
	struct namespace *namespace = tsk->namespace;
	struct namespace *new_ns;
	struct vfsmount *rootmnt = NULL, *pwdmnt = NULL, *altrootmnt = NULL;
	struct fs_struct *fs = tsk->fs;
	struct vfsmount *p, *q;

    // 分配 namespace 空间
	new_ns = kmalloc(sizeof(struct namespace), GFP_KERNEL);

	/* First pass: copy the tree topology */
    // 第一阶段：拷贝 namespace 对应的 vfsmount tree
	new_ns->root = copy_tree(namespace->root, namespace->root->mnt_root);

	/*
	 * Second pass: switch the tsk->fs->* elements and mark new vfsmounts
	 * as belonging to new namespace.  We have already acquired a private
	 * fs_struct, so tsk->fs->lock is not needed.
	 */
    
    // 第二阶段：将 vfsmount tree 所有节点的 namespace 对象指向新的 namespace
	p = namespace->root;
	q = new_ns->root;
	while (p) {
		q->mnt_namespace = new_ns;
		if (fs) {
            // 更改进程 fs_struct 中文件系统相关成员
			if (p == fs->rootmnt) {
				rootmnt = p;
				fs->rootmnt = mntget(q);
			}
			if (p == fs->pwdmnt) {
				pwdmnt = p;
				fs->pwdmnt = mntget(q);
			}
			if (p == fs->altrootmnt) {
				altrootmnt = p;
				fs->altrootmnt = mntget(q);
			}
		}
		p = next_mnt(p, namespace->root);
		q = next_mnt(q, new_ns->root);
	}

    // 进程的 namespace 指向新的 namespace
	tsk->namespace = new_ns;

	return 0;
}
```

### bind mount

bind mount 机制允许我们可以将一个目录/文件，而不是整个设备，挂载到一个指定目录上。工作原理就是：将挂载点的 inode 更改为被挂载目录/文件的 inode。
这样对挂载点文件的修改，就是对被挂载文件的修改。

docker 中的 volume 就是使用了 bind mount，从而实现在容器中访问宿主机上的文件。

### 路径查找

我们来描述下，linux 中路径查找的流程。当在一个进程中查找一个路径的时候，流程是什么样的呢？<br>

首先每个进程的 task_struct 结构体有个字段指向 fs_struct，内部有几个核心字段：

```c
struct fs_struct {
    struct dentry * root; // 进程根目录对应的目录项
    struct dentry * pwd; // 当前目录对应的目录项
};
```

我们先来了解下什么是 dentry，dentry 就是目录项对象，存在的目的就是为了实现路径查找，每当进程进行路径查找的时候，针对路径中的每个 path，
如果对应的 dentry 还没创建，内核就会创建一个 dentry，dentry 会跟目录文件对应的 inode 对象关联起来。注意，在磁盘中，并不存在与 dentry 对应的数据结构。<br>

每个挂载的文件系统对应一个 `vfsmount` 结构，该结构核心成员包括：
```c
struct vfsmount {
    struct vfsmount * mnt_parent; // 挂载点所在的文件系统
    struct dentry * mnt_mountpoint; // 挂载点对应的 dentry
    struct dentry * mnt_root; // 文件系统的根目录对应的目录项，同一个文件系统可以挂载多次，多个 vfsmount 的 mnt_root 都指向同一个对象：文件系统的根目录对象。
    struct super_block *mnt_sb;// 文件系统对应的超级块
};
```

在一个 mount namespace 中，实际维护的就是 vfsmount 树，mount namespace 结构体的一个核心字段就是 `struct vfsmount * root`，该文件系统对应的根目录，默认就是该 mount namespace 下进程的根目录。<br>

当我们挂载后，除了生成并注册 vfsmount 外，还会在修改挂载点对应的 dentry.d_mounted 成员，这样我们在遍历路径的时候，就知道某个目录是否被挂载。<br>

有了上述基础知识，我们来看看在一个进程中，路径查找的流程(我们先不管符号链接相关内容)：

如果是绝对路径，就从进程对应的根目录 dentry 出发，内核中会维护一个 dentry 的全局哈希表，就是为了方便查找，对应的 key 为：父母录 dentry 地址 + 子目录名，我们首先在这个 hash 表中查找子目录对应的 dentry，如果不存在，我们就通过 dentry 对应的 inode从文件系统中获取目录文件对应的内容，如果子目录存在则创建一个新的 dentry 对象并插入到全局 hash 表中。<br>

对于已经存在的 dentry，其完全有可能被挂载，所以我们还需要查看 dentry.d_mounted 成员，如果 dentry.d_mounted > 0 ，我们就知道这是个挂载点，那我们怎么知道挂载点对应的 vfsmount 对象呢？实际上，内核也为 vfsmount 维护了一个全局哈希表，key 为父 vfsmount 对象地址 + 挂载目录名，然后我们在哈希表中找到这个 vfsmount 对象。这里要特别注意：**一个挂载点是可以同时挂载多个文件系统的，最新挂载的文件系统会隐藏旧的文件系统，我们现在查找的 vfsmount 只是最先关在的文件系统，后续如果有挂载，就是挂载到这个 vfsmount 的根目录上的，因此我们还需要查找当前 vfsmount 的根目录是否有挂载文件系统**，如此循环，直到 vfsmount 根 dentry.d_mounted == 0。<br>

然后我们就从这个子 vfsmount 的根目录开始，继续上述流程。

# CGroup

control group 的简称。linux 中，cgroup 实际上是个特殊的(虚拟)文件系统。我们可以通过 cgroup 限制进程的 cpu、mem 使用。在 linux 中执行 `mount | grep cgroup` 结果如下：
```sh
tmpfs on /sys/fs/cgroup type tmpfs
cgroup on /sys/fs/cgroup/memory type cgroup
cgroup on /sys/fs/cgroup/cpu type cgroup
#...
```

我们可以看到 /sys/fs/cgroup 下的每个目录都是 cgroup 文件系统的挂载点。这里我们以 cpu 使用率为例，看如何通过 cgroup 方式限制。<br>

目录 /sys/fs/cgroup/cpu 下包含很多配置文件以及一个 tasks 文件，系统中的进程默认都在这个 tasks 文件中，因此这些进程对应的 cpu 限制，也由与 tasks 文件同级的配置文件指定。我们重点关注两个文件：

cpu.cfs_quota_us / cpu.cfs_period_us，这两个配置文件决定了进程的 cpu 使用率，默认情况下 cpu.cfs_quota_us == -1，即不限制 cpu 使用率。如果要自定义 cpu 限制怎么办呢？<br>

首先在 /sys/fs/cgroup/cpu 下创建一个目录，然后在该目录的 tasks 文件中加入要限制的进程，然后更改对应的 cpu.cfs_quota_us 值，docker 就是这么做的。

