# LINUX NAMESPACE

##  简介

Linux Namespace是Linux提供的一种内核级别环境隔离的方法。不知道你是否还记得很早以前的Unix有一个叫chroot的系统调用（通过修改根目录把用户限制到一个特定目录下），chroot提供了一种简单的隔离模式：chroot内部的文件系统无法访问外部的内容。Linux Namespace在此基础上，提供了对UTS(UNIX Timesharing System)、IPC、mount、PID、network、User等的隔离机制。

```
#ls /home
#chroot /home/testchroot
#export PATH=$PATH:/bin
#ls /home
```

举个例子，我们都知道，Linux下的超级父亲进程的PID是1，所以，同chroot一样，如果我们可以把用户的进程空间限制到某个进程分支下，并像chroot那样让其下面的进程 看到的那个超级父进程的PID为1，于是就可以达到资源隔离的效果了（不同的PID namespace中的进程无法看到彼此）

**Linux Namespace 有如下种类**，官方文档在这里《[Namespace in Operation](http://lwn.net/Articles/531114/)》

| 分类                   | 系统调用参数  | 相关内核版本                                                 |
| :--------------------- | :------------ | :----------------------------------------------------------- |
| **Mount namespaces**   | CLONE_NEWNS   | [Linux 2.4.19](http://lwn.net/2001/0301/a/namespaces.php3)   |
| **UTS namespaces**     | CLONE_NEWUTS  | [Linux 2.6.19](http://lwn.net/Articles/179345/)              |
| **IPC namespaces**     | CLONE_NEWIPC  | [Linux 2.6.19](http://lwn.net/Articles/187274/)              |
| **PID namespaces**     | CLONE_NEWPID  | [Linux 2.6.24](http://lwn.net/Articles/259217/)              |
| **Network namespaces** | CLONE_NEWNET  | [始于Linux 2.6.24 完成于 Linux 2.6.29](http://lwn.net/Articles/219794/) |
| **User namespaces**    | CLONE_NEWUSER | [始于 Linux 2.6.23 完成于 Linux 3.8)](http://lwn.net/Articles/528078/) |

主要是三个系统调用

- **clone****()** – 实现线程的系统调用，用来创建一个新的进程，并可以通过设计上述参数达到隔离。
- **unshare****()** – 使某进程脱离某个namespace
- **setns****()** – 把某进程加入到某个namespace

unshare() 和 setns() 都比较简单，大家可以自己man，我这里不说了。

下面还是让我们来看一些示例（以下的测试程序最好在Linux 内核为3.8以上的版本中运行）。

## clone()系统调用

首先，我们来看一下一个最简单的clone()系统调用的示例base.c，（后面，我们的程序都会基于这个程序做修改）：

``` c
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
 
/* 定义一个给 clone 用的栈，栈大小1M */
#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];
 
char* const container_args[] = {
    "/bin/bash",
    NULL
};
 
int container_main(void* arg)
{
    printf("Container - inside the container!\n");
    /* 直接执行一个shell，以便我们观察这个进程空间里的资源是否被隔离了 */
    execv(container_args[0], container_args); 
    printf("Something's wrong!\n");
    return 1;
}
 
int main()
{
    printf("Parent - start a container!\n");
    /* 调用clone函数，其中传出一个函数，还有一个栈空间的（为什么传尾指针，因为栈是反着的） */
    int container_pid = clone(container_main, container_stack+STACK_SIZE, SIGCHLD, NULL);
    /* 等待子进程结束 */
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

从上面的程序，我们可以看到，这和pthread基本上是一样的玩法。但是，对于上面的程序，父子进程的空间没有差别，都可以互相访问。

下面， 让我们来看几个例子看看，Linux的Namespace是什么样的。

## UTS Namespace

下面的代码，我略去了上面那些头文件和数据结构的定义，只有最重要的部分uts.c。

```c
int container_main(void* arg)
{
    printf("Container - inside the container!\n");
    sethostname("container",10); /* 设置hostname */
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}
 
int main()
{
    printf("Parent - start a container!\n");
    int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | SIGCHLD, NULL); /*启用CLONE_NEWUTS Namespace隔离 */
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

运行上面的程序需要root权限，子进程的hostname变成了 container。

```
#./uts
Parent - start a container!
Container - inside the container!
root@container:~# hostname
container
root@container:~# uname -n
container
```

## IPC Namespace

IPC全称 Inter-Process Communication，是Unix/Linux下进程间通信的一种方式，IPC有共享内存、信号量、消息队列等方法。所以，为了隔离，我们也需要把IPC给隔离开来，这样，只有在同一个Namespace下的进程才能相互通信。如果你熟悉IPC的原理的话，你会知道，IPC需要有一个全局的ID，即然是全局的，那么就意味着我们的Namespace需要对这个ID隔离，不能让别的Namespace的进程看到。

要启动IPC隔离，我们只需要在调用clone时加上CLONE_NEWIPC参数就可以了ipc.c。

```
int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | CLONE_NEWIPC | SIGCHLD, NULL);
```

首先，我们先创建一个IPC的Queue（如下所示，全局的Queue ID是0）

```
hchen@ubuntu:~$ ipcmk -Q 
Message queue id: 0
 
hchen@ubuntu:~$ ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0xd0d56eb2 0          hchen      644        0            0
```

如果我们运行没有CLONE_NEWIPC的程序，我们会看到，在子进程中还是能看到这个全启的IPC Queue。

```
hchen@ubuntu:~$ sudo ./uts
Parent - start a container!
Container - inside the container!
 
root@container:~# ipcs -q
 
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0xd0d56eb2 0          hchen      644        0            0
```

但是，如果我们运行加上了CLONE_NEWIPC的程序，我们就会下面的结果：

```
root@ubuntu:~$ sudo./ipc
Parent - start a container!
Container - inside the container!
 
root@container:~/linux_namespace# ipcs -q
 
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
```

我们可以看到IPC已经被隔离了。

## PID Namespace

我们继续修改上面的程序：

```
int container_main(void* arg)
{
    /* 查看子进程的PID，我们可以看到其输出子进程的 pid 为 1 */
    printf("Container [%5d] - inside the container!\n", getpid());
    sethostname("container",10);
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}
 
int main()
{
    printf("Parent [%5d] - start a container!\n", getpid());
    /*启用PID namespace - CLONE_NEWPID*/
    int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | CLONE_NEWPID | SIGCHLD, NULL); 
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

运行结果如下（我们可以看到，子进程的pid是1了）：

```
hchen@ubuntu:~$ sudo ./pid
Parent [ 3474] - start a container!
Container [    1] - inside the container!
root@container:~# echo $$
1
```

你可能会问，PID为1有个毛用啊？我们知道，在传统的UNIX系统中，PID为1的进程是init，地位非常特殊。他作为所有进程的父进程，有很多特权（比如：屏蔽信号等），另外，其还会为检查所有进程的状态，我们知道，如果某个子进程脱离了父进程（父进程没有wait它），那么init就会负责回收资源并结束这个子进程。所以，要做到进程空间的隔离，首先要创建出PID为1的进程，最好就像chroot那样，把子进程的PID在容器内变成1。

**但是，我们会发现，在子进程的shell里输入ps,top等命令，我们还是可以看得到所有进程**。说明并没有完全隔离。这是因为，像ps, top这些命令会去读/proc文件系统，所以，因为/proc文件系统在父进程和子进程都是一样的，所以这些命令显示的东西都是一样的。

所以，我们还需要对文件系统进行隔离。

## Mount Namespace

下面的例程中，我们在启用了mount namespace并在子进程中重新mount了/proc文件系统。

```
int container_main(void* arg)
{
    printf("Container [%5d] - inside the container!\n", getpid());
    sethostname("container",10);
    /* 重新mount proc文件系统到 /proc下 */
    system("mount -t proc proc /proc");
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}
 
int main()
{
    printf("Parent [%5d] - start a container!\n", getpid());
    /* 启用Mount Namespace - 增加CLONE_NEWNS参数 */
    int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | CLONE_NEWPID | CLONE_NEWNS | SIGCHLD, NULL);
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

运行结果如下：

```
hchen@ubuntu:~$ sudo ./fs
Parent [ 3502] - start a container!
Container [    1] - inside the container!
root@container:~# ps -elf 
F S UID        PID  PPID  C PRI  NI ADDR SZ WCHAN  STIME TTY          TIME CMD
4 S root         1     0  0  80   0 -  6917 wait   19:55 pts/2    00:00:00 /bin/bash
0 R root        14     1  0  80   0 -  5671 -      19:56 pts/2    00:00:00 ps -elf

```

上面，我们可以看到只有两个进程 ，而且pid=1的进程是我们的/bin/bash。我们还可以看到/proc目录下也干净了很多：

```
root@container:~# ls /proc
1          dma          key-users   net            sysvipc
16         driver       kmsg        pagetypeinfo   timer_list
acpi       execdomains  kpagecount  partitions     timer_stats
asound     fb           kpageflags  sched_debug    tty
buddyinfo  filesystems  loadavg     schedstat      uptime
bus        fs           locks       scsi           version
cgroups    interrupts   mdstat      self           version_signature
cmdline    iomem        meminfo     slabinfo       vmallocinfo
consoles   ioports      misc        softirqs       vmstat
cpuinfo    irq          modules     stat           zoneinfo
crypto     kallsyms     mounts      swaps
devices    kcore        mpt         sys
diskstats  keys         mtrr        sysrq-trigger
```

下图，我们也可以看到在子进程中的top命令只看得到两个进程了。

![img](https://coolshell.cn/wp-content/uploads/2015/04/mount.namespace.jpg)

这里，多说一下。在通过CLONE_NEWNS创建mount namespace后，父进程会把自己的文件结构复制给子进程中。而子进程中新的namespace中的所有mount操作都只影响自身的文件系统，而不对外界产生任何影响。这样可以做到比较严格地隔离。

你可能会问，我们是不是还有别的一些文件系统也需要这样mount? 是的。

## Docker的 Mount Namespace

下面我将向演示一个“山寨镜像”，其模仿了Docker的Mount Namespace。

首先，我们需要一个rootfs，也就是我们需要把我们要做的镜像中的那些命令什么的copy到一个rootfs的目录下，我们模仿Linux构建如下的目录：

```
hchen@ubuntu:~/rootfs$ ls
bin  dev  etc  home  lib  lib64  mnt  opt  proc  root  run  sbin  sys  tmp  usr  var
```

然后，我们把一些我们需要的命令copy到 rootfs/bin目录中（sh命令必需要copy进去，不然我们无法 chroot ）

```
hchen@ubuntu:~/rootfs$ ls ./bin ./usr/bin
  
./bin:
bash   chown  gzip      less  mount       netstat  rm     tabs  tee      top       tty
cat    cp     hostname  ln    mountpoint  ping     sed    tac   test     touch     umount
chgrp  echo   ip        ls    mv          ps       sh     tail  timeout  tr        uname
chmod  grep   kill      more  nc          pwd      sleep  tar   toe      truncate  which
 
./usr/bin:
awk  env  groups  head  id  mesg  sort  strace  tail  top  uniq  vi  wc  xargs

```

注：你可以使用ldd命令把这些命令相关的那些so文件copy到对应的目录：

```
hchen@ubuntu:~/rootfs/bin$ ldd bash
    linux-vdso.so.1 =>  (0x00007fffd33fc000)
    libtinfo.so.5 => /lib/x86_64-linux-gnu/libtinfo.so.5 (0x00007f4bd42c2000)
    libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f4bd40be000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f4bd3cf8000)
    /lib64/ld-linux-x86-64.so.2 (0x00007f4bd4504000)
```

下面是我的rootfs中的一些so文件：

```
hchen@ubuntu:~/rootfs$ ls ./lib64 ./lib/x86_64-linux-gnu/
 
./lib64:
ld-linux-x86-64.so.2
 
./lib/x86_64-linux-gnu/:
libacl.so.1      libmemusage.so         libnss_files-2.19.so    libpython3.4m.so.1
libacl.so.1.1.0  libmount.so.1          libnss_files.so.2       libpython3.4m.so.1.0
libattr.so.1     libmount.so.1.1.0      libnss_hesiod-2.19.so   libresolv-2.19.so
libblkid.so.1    libm.so.6              libnss_hesiod.so.2      libresolv.so.2
libc-2.19.so     libncurses.so.5        libnss_nis-2.19.so      libselinux.so.1
libcap.a         libncurses.so.5.9      libnss_nisplus-2.19.so  libtinfo.so.5
libcap.so        libncursesw.so.5       libnss_nisplus.so.2     libtinfo.so.5.9
libcap.so.2      libncursesw.so.5.9     libnss_nis.so.2         libutil-2.19.so
libcap.so.2.24   libnsl-2.19.so         libpcre.so.3            libutil.so.1
libc.so.6        libnsl.so.1            libprocps.so.3          libuuid.so.1
libdl-2.19.so    libnss_compat-2.19.so  libpthread-2.19.so      libz.so.1
libdl.so.2       libnss_compat.so.2     libpthread.so.0
libgpm.so.2      libnss_dns-2.19.so     libpython2.7.so.1
libm-2.19.so     libnss_dns.so.2        libpython2.7.so.1.0
```

包括这些命令依赖的一些配置文件：

```
hchen@ubuntu:~/rootfs$ ls ./etc
bash.bashrc  group  hostname  hosts  ld.so.cache  nsswitch.conf  passwd  profile  
resolv.conf  shadow
```

你现在会说，我靠，有些配置我希望是在容器起动时给他设置的，而不是hard code在镜像中的。比如：/etc/hosts，/etc/hostname，还有DNS的/etc/resolv.conf文件。好的。那我们在rootfs外面，我们再创建一个conf目录，把这些文件放到这个目录中。

```
hchen@ubuntu:~$ ls ./conf
hostname     hosts     resolv.conf
```

这样，我们的父进程就可以动态地设置容器需要的这些文件的配置， 然后再把他们mount进容器，这样，容器的镜像中的配置就比较灵活了。

好了，终于到了我们的程序docker.c。

```
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/mount.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
 
#define STACK_SIZE (1024 * 1024)
 
static char container_stack[STACK_SIZE];
char* const container_args[] = {
    "/bin/bash",
    "-l",
    NULL
};
 
int container_main(void* arg)
{
    printf("Container [%5d] - inside the container!\n", getpid());
 
    //set hostname
    sethostname("container",10);
 
    //remount "/proc" to make sure the "top" and "ps" show container's information
    if (mount("proc", "rootfs/proc", "proc", 0, NULL) !=0 ) {
        perror("proc");
    }
    if (mount("sysfs", "rootfs/sys", "sysfs", 0, NULL)!=0) {
        perror("sys");
    }
    if (mount("none", "rootfs/tmp", "tmpfs", 0, NULL)!=0) {
        perror("tmp");
    }
    if (mount("udev", "rootfs/dev", "devtmpfs", 0, NULL)!=0) {
        perror("dev");
    }
    if (mount("devpts", "rootfs/dev/pts", "devpts", 0, NULL)!=0) {
        perror("dev/pts");
    }
    if (mount("shm", "rootfs/dev/shm", "tmpfs", 0, NULL)!=0) {
        perror("dev/shm");
    }
    if (mount("tmpfs", "rootfs/run", "tmpfs", 0, NULL)!=0) {
        perror("run");
    }
    /* 
     * 模仿Docker的从外向容器里mount相关的配置文件 
     * 你可以查看：/var/lib/docker/containers/<container_id>/目录，
     * 你会看到docker的这些文件的。
     */
    if (mount("conf/hosts", "rootfs/etc/hosts", "none", MS_BIND, NULL)!=0 ||
          mount("conf/hostname", "rootfs/etc/hostname", "none", MS_BIND, NULL)!=0 ||
          mount("conf/resolv.conf", "rootfs/etc/resolv.conf", "none", MS_BIND, NULL)!=0 ) {
        perror("conf");
    }
    /* 模仿docker run命令中的 -v, --volume=[] 参数干的事 */
    if (mount("/tmp/t1", "rootfs/mnt", "none", MS_BIND, NULL)!=0) {
        perror("mnt");
    }
 
    /* chroot 隔离目录 */
    if ( chdir("./rootfs") != 0 || chroot("./") != 0 ){
        perror("chdir/chroot");
    }
 
    execv(container_args[0], container_args);
    perror("exec");
    printf("Something's wrong!\n");
    return 1;
}
 
int main()
{
    printf("Parent [%5d] - start a container!\n", getpid());
    int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID | CLONE_NEWNS | SIGCHLD, NULL);
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

运行上面的程序，你会看到下面的挂载信息以及一个所谓的“镜像”：

```
hchen@ubuntu:~$ sudo ./mount
Parent [ 4517] - start a container!
Container [    1] - inside the container!
root@container:/# mount
proc on /proc type proc (rw,relatime)
sysfs on /sys type sysfs (rw,relatime)
none on /tmp type tmpfs (rw,relatime)
udev on /dev type devtmpfs (rw,relatime,size=493976k,nr_inodes=123494,mode=755)
devpts on /dev/pts type devpts (rw,relatime,mode=600,ptmxmode=000)
tmpfs on /run type tmpfs (rw,relatime)
/dev/disk/by-uuid/18086e3b-d805-4515-9e91-7efb2fe5c0e2 on /etc/hosts type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/dev/disk/by-uuid/18086e3b-d805-4515-9e91-7efb2fe5c0e2 on /etc/hostname type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/dev/disk/by-uuid/18086e3b-d805-4515-9e91-7efb2fe5c0e2 on /etc/resolv.conf type ext4 (rw,relatime,errors=remount-ro,data=ordered)
 
root@container:/# ls /bin /usr/bin
/bin:
bash   chmod  echo  hostname  less  more    mv   ping  rm   sleep  tail  test     top    truncate  uname
cat    chown  grep  ip        ln    mount   nc   ps    sed  tabs   tar   timeout  touch  tty       which
chgrp  cp     gzip  kill      ls    mountpoint  netstat  pwd   sh   tac    tee   toe      tr     umount
 
/usr/bin:
awk  env  groups  head  id  mesg  sort  strace  tail  top  uniq  vi  wc  xargs

```

关于如何做一个chroot的目录，这里有个工具叫[DebootstrapChroot](https://wiki.ubuntu.com/DebootstrapChroot)，你可以顺着链接去看看。

## User Namespace

User Namespace主要是用了CLONE_NEWUSER的参数。使用了这个参数后，内部看到的UID和GID已经与外部不同了，默认显示为65534。那是因为容器找不到其真正的UID所以，设置上了最大的UID（其设置定义在/proc/sys/kernel/overflowuid）。

要把容器中的uid和真实系统的uid给映射在一起，需要修改 **/proc//uid_map** 和 **/proc//gid_map** 这两个文件。这两个文件的格式为：

```
ID-inside-ns ID-outside-ns length
```

其中：

- 第一个字段ID-inside-ns表示在容器显示的UID或GID，
- 第二个字段ID-outside-ns表示容器外映射的真实的UID或GID。
- 第三个字段表示映射的范围，一般填1，表示一一对应。

比如，把真实的uid=1000映射成容器内的uid=0

```
$ cat /proc/2465/uid_map
         0       1000          1
```

再比如下面的示例：表示把namespace内部从0开始的uid映射到外部从0开始的uid，其最大范围是无符号32位整形

```
$ cat /proc/$$/uid_map
         0          0          4294967295
```

另外，需要注意的是：

- 写这两个文件的进程需要这个namespace中的CAP_SETUID (CAP_SETGID)权限（可参看[Capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html)）
- 写入的进程必须是此user namespace的父或子的user namespace进程。
- 另外需要满如下条件之一：1）父进程将effective uid/gid映射到子进程的user namespace中，2）父进程如果有CAP_SETUID/CAP_SETGID权限，那么它将可以映射到父进程中的任一uid/gid。

这些规则看着都烦，我们来看程序吧user.c（下面的程序有点长，但是非常简单，如果你读过《Unix网络编程》上卷，你应该可以看懂）：

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/mount.h>
#include <sys/capability.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
 
#define STACK_SIZE (1024 * 1024)
 
static char container_stack[STACK_SIZE];
char* const container_args[] = {
    "/bin/bash",
    NULL
};
 
int pipefd[2];
 
void set_map(char* file, int inside_id, int outside_id, int len) {
    FILE* mapfd = fopen(file, "w");
    if (NULL == mapfd) {
        perror("open file error");
        return;
    }
    fprintf(mapfd, "%d %d %d", inside_id, outside_id, len);
    fclose(mapfd);
}
 
void set_uid_map(pid_t pid, int inside_id, int outside_id, int len) {
    char file[256];
    sprintf(file, "/proc/%d/uid_map", pid);
    set_map(file, inside_id, outside_id, len);
}
 
void set_gid_map(pid_t pid, int inside_id, int outside_id, int len) {
    char file[256];
    sprintf(file, "/proc/%d/gid_map", pid);
    set_map(file, inside_id, outside_id, len);
}
 
int container_main(void* arg)
{
 
    printf("Container [%5d] - inside the container!\n", getpid());
 
    printf("Container: eUID = %ld;  eGID = %ld, UID=%ld, GID=%ld\n",
            (long) geteuid(), (long) getegid(), (long) getuid(), (long) getgid());
 
    /* 等待父进程通知后再往下执行（进程间的同步） */
    char ch;
    close(pipefd[1]);
    read(pipefd[0], &ch, 1);
 
    printf("Container [%5d] - setup hostname!\n", getpid());
    //set hostname
    sethostname("container",10);
 
    //remount "/proc" to make sure the "top" and "ps" show container's information
    mount("proc", "/proc", "proc", 0, NULL);
 
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}
 
int main()
{
    const int gid=getgid(), uid=getuid();
 
    printf("Parent: eUID = %ld;  eGID = %ld, UID=%ld, GID=%ld\n",
            (long) geteuid(), (long) getegid(), (long) getuid(), (long) getgid());
 
    pipe(pipefd);
  
    printf("Parent [%5d] - start a container!\n", getpid());
 
    int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWUSER | SIGCHLD, NULL);
 
     
    printf("Parent [%5d] - Container [%5d]!\n", getpid(), container_pid);
 
    //To map the uid/gid, 
    //   we need edit the /proc/PID/uid_map (or /proc/PID/gid_map) in parent
    //The file format is
    //   ID-inside-ns   ID-outside-ns   length
    //if no mapping, 
    //   the uid will be taken from /proc/sys/kernel/overflowuid
    //   the gid will be taken from /proc/sys/kernel/overflowgid
    set_uid_map(container_pid, 0, uid, 1);
    set_gid_map(container_pid, 0, gid, 1);
 
    printf("Parent [%5d] - user/group mapping done!\n", getpid());
 
    /* 通知子进程 */
    close(pipefd[1]);
 
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

上面的程序，我们用了一个pipe来对父子进程进行同步，为什么要这样做？因为子进程中有一个execv的系统调用，这个系统调用会把当前子进程的进程空间给全部覆盖掉，我们希望在execv之前就做好user namespace的uid/gid的映射，这样，execv运行的/bin/bash就会因为我们设置了uid为0的inside-uid而变成#号的提示符。

整个程序的运行效果如下：

```shell
hchen@ubuntu:~$ id
uid=1000(hchen) gid=1000(hchen) groups=1000(hchen)
 
hchen@ubuntu:~$ ./user #<--以hchen用户运行
Parent: eUID = 1000;  eGID = 1000, UID=1000, GID=1000 
Parent [ 3262] - start a container!
Parent [ 3262] - Container [ 3263]!
Parent [ 3262] - user/group mapping done!
Container [    1] - inside the container!
Container: eUID = 0;  eGID = 0, UID=0, GID=0 #<---Container里的UID/GID都为0了
Container [    1] - setup hostname!
 
root@container:~# id #<----我们可以看到容器里的用户和命令行提示符是root用户了
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)
```

虽然容器里是root，但其实这个容器的/bin/bash进程是以一个普通用户来运行的。这样一来，我们容器的安全性会得到提高。

我们注意到，User Namespace是以普通用户运行，但是别的Namespace需要root权限，那么，如果我要同时使用多个Namespace，该怎么办呢？一般来说，我们先用一般用户创建User Namespace，然后把这个一般用户映射成root，在容器内用root来创建其它的Namesapce。

## Network Namespace

Network的Namespace比较啰嗦。在Linux下，我们一般用ip命令创建Network Namespace（Docker的源码中，它没有用ip命令，而是自己实现了ip命令内的一些功能——是用了Raw Socket发些“奇怪”的数据，呵呵）。这里，我还是用ip命令讲解一下。

首先，我们先看个图，下面这个图基本上就是Docker在宿主机上的网络示意图（其中的物理网卡并不准确，因为docker可能会运行在一个VM中，所以，这里所谓的“物理网卡”其实也就是一个有可以路由的IP的网卡）

![network.namespace](https://coolshell.cn/wp-content/uploads/2015/04/network.namespace.jpg)

上图中，Docker使用了一个私有网段，172.40.1.0，docker还可能会使用10.0.0.0和192.168.0.0这两个私有网段，关键看你的路由表中是否配置了，如果没有配置，就会使用，如果你的路由表配置了所有私有网段，那么docker启动时就会出错了。

当你启动一个Docker容器后，你可以使用ip link show或ip addr show来查看当前宿主机的网络情况（我们可以看到有一个docker0，还有一个veth22a38e6的虚拟网卡——给容器用的）：

```
hchen@ubuntu:~$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state ... 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc ...
    link/ether 00:0c:29:b7:67:7d brd ff:ff:ff:ff:ff:ff
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    link/ether 56:84:7a:fe:97:99 brd ff:ff:ff:ff:ff:ff
5: veth22a38e6: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc ...
    link/ether 8e:30:2a:ac:8c:d1 brd ff:ff:ff:ff:ff:ff
```

那么，要做成这个样子应该怎么办呢？我们来看一组命令：

```
## 首先，我们先增加一个网桥lxcbr0，模仿docker0
brctl addbr lxcbr0
brctl stp lxcbr0 off
ifconfig lxcbr0 192.168.10.1/24 up #为网桥设置IP地址
 
## 接下来，我们要创建一个network namespace - ns1
 
# 增加一个namesapce 命令为 ns1 （使用ip netns add命令）
ip netns add ns1 
 
# 激活namespace中的loopback，即127.0.0.1（使用ip netns exec ns1来操作ns1中的命令）
ip netns exec ns1   ip link set dev lo up 
 
## 然后，我们需要增加一对虚拟网卡
 
# 增加一个pair虚拟网卡，注意其中的veth类型，其中一个网卡要按进容器中
ip link add veth-ns1 type veth peer name lxcbr0.1
 
# 把 veth-ns1 按到namespace ns1中，这样容器中就会有一个新的网卡了
ip link set veth-ns1 netns ns1
 
# 把容器里的 veth-ns1改名为 eth0 （容器外会冲突，容器内就不会了）
ip netns exec ns1  ip link set dev veth-ns1 name eth0 
 
# 为容器中的网卡分配一个IP地址，并激活它
ip netns exec ns1 ifconfig eth0 192.168.10.11/24 up
 
 
# 上面我们把veth-ns1这个网卡按到了容器中，然后我们要把lxcbr0.1添加上网桥上
brctl addif lxcbr0 lxcbr0.1
 
# 为容器增加一个路由规则，让容器可以访问外面的网络
ip netns exec ns1     ip route add default via 192.168.10.1
 
# 在/etc/netns下创建network namespce名称为ns1的目录，
# 然后为这个namespace设置resolv.conf，这样，容器内就可以访问域名了
mkdir -p /etc/netns/ns1
echo "nameserver 8.8.8.8" > /etc/netns/ns1/resolv.conf
```

上面基本上就是docker网络的原理了，只不过，

- Docker的resolv.conf没有用这样的方式，而是用了[上篇中的Mount Namesapce的那种方式](https://coolshell.cn/articles/17010.html)
- 另外，docker是用进程的PID来做Network Namespace的名称的。

了解了这些后，你甚至可以为正在运行的docker容器增加一个新的网卡：

```
ip link add peerA type veth peer name peerB 
brctl addif docker0 peerA 
ip link set peerA up 
ip link set peerB netns ${container-pid} 
ip netns exec ${container-pid} ip link set dev peerB name eth1 
ip netns exec ${container-pid} ip link set eth1 up ; 
ip netns exec ${container-pid} ip addr add ${ROUTEABLE_IP} dev eth1 ;
```

上面的示例是我们为正在运行的docker容器，增加一个eth1的网卡，并给了一个静态的可被外部访问到的IP地址。

这个需要把外部的“物理网卡”配置成混杂模式，这样这个eth1网卡就会向外通过ARP协议发送自己的Mac地址，然后外部的交换机就会把到这个IP地址的包转到“物理网卡”上，因为是混杂模式，所以eth1就能收到相关的数据，一看，是自己的，那么就收到。这样，Docker容器的网络就和外部通了。

当然，无论是Docker的NAT方式，还是混杂模式都会有性能上的问题，NAT不用说了，存在一个转发的开销，混杂模式呢，网卡上收到的负载都会完全交给所有的虚拟网卡上，于是就算一个网卡上没有数据，但也会被其它网卡上的数据所影响。

**这两种方式都不够完美，我们知道，真正解决这种网络问题需要使用VLAN技术，于是Google的同学们为Linux内核实现了一个[IPVLAN的驱动](https://lwn.net/Articles/620087/)，这基本上就是为Docker量身定制的。即MACVLAN**

## Namespace文件

上面就是目前Linux Namespace的玩法。 现在，我来看一下其它的相关东西。

让我们运行一下上篇中的那个 fs的程序（也就是PID Namespace中那个mount proc的程序），然后不要退出。

```
$ sudo ./fs 
[sudo] password for hchen: 
Parent [ 4599] - start a container!
Container [    1] - inside the container!
```

我们到另一个shell中查看一下父子进程的PID：

```
hchen@ubuntu:~$ pstree -p 4599
pid.mnt(4599)───bash(4600)
```

我们可以到proc下（/proc//ns）查看进程的各个namespace的id（内核版本需要3.8以上）。

下面是父进程的：

```
hchen@ubuntu:~$ sudo ls -l /proc/4599/ns
total 0
lrwxrwxrwx 1 root root 0  4月  7 22:01 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 root root 0  4月  7 22:01 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 root root 0  4月  7 22:01 net -> net:[4026531956]
lrwxrwxrwx 1 root root 0  4月  7 22:01 pid -> pid:[4026531836]
lrwxrwxrwx 1 root root 0  4月  7 22:01 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0  4月  7 22:01 uts -> uts:[4026531838]
```

下面是子进程的：

```
hchen@ubuntu:~$ sudo ls -l /proc/4600/ns
total 0
lrwxrwxrwx 1 root root 0  4月  7 22:01 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 root root 0  4月  7 22:01 mnt -> mnt:[4026532520]
lrwxrwxrwx 1 root root 0  4月  7 22:01 net -> net:[4026531956]
lrwxrwxrwx 1 root root 0  4月  7 22:01 pid -> pid:[4026532522]
lrwxrwxrwx 1 root root 0  4月  7 22:01 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0  4月  7 22:01 uts -> uts:[4026532521]
```

我们可以看到，其中的ipc，net，user是同一个ID，而mnt,pid,uts都是不一样的。如果两个进程指向的namespace编号相同，就说明他们在同一个namespace下，否则则在不同namespace里面。

这些文件还有另一个作用，那就是，一旦这些文件被打开，只要其fd被占用着，那么就算PID所属的所有进程都已经结束，创建的namespace也会一直存在。比如：我们可以通过：mount –bind /proc/4600/ns/uts ~/uts 来hold这个namespace。

另外，我们在上篇中讲过一个setns的系统调用，其函数声明如下：

```
int setns(int fd, int nstype);
```

其中第一个参数就是一个fd，也就是一个open()系统调用打开了上述文件后返回的fd，比如：

```
fd = open("/proc/4600/ns/nts", O_RDONLY);  // 获取namespace文件描述符
setns(fd, 0); // 加入新的namespace
```
