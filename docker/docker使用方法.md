## Docker 的基本使用

`docker info`  
返回所有容器和镜像的数量，docker使用执行驱动和存储驱动，以及基本配置


*	docker hello world

Docker 允许你在容器内运行应用程序， 使用 docker run 命令来在容器内运行一个应用程序。输出Hello world      
&emsp;

`docker run ubuntu:15.10 /bin/echo "Hello world"`  
> 各个参数解析：  
> docker: Docker 的二进制执行文件。  
> run:与前面的 docker 组合来运行一个容器。  
> ubuntu:15.10指定要运行的镜像，Docker首先从本地主机上查找镜像是否存在，如果不存在，Docker 就会从镜像仓库 Docker Hub 下载公共镜像并保存到本地宿主机中。  
> /bin/echo "Hello world": 在启动的容器里执行的命令.  
  
以上命令完整的意思可以解释为：Docker 以 ubuntu15.10 镜像创建一个新容器，然后在容器里执行 bin/echo "Hello world"，然后输出结果。


&emsp;

* 运行交互式的容器
我们通过docker的两个参数 -i -t，让docker运行的容器实现"对话"的能力. 

`docker run -i -t ubuntu:15.10 /bin/bash` 
> 各个参数解析：  
> -t:在新容器内指定一个伪终端或终端。  
> -i:允许你对容器内的标准输入 (STDIN) 进行交互
> 进入容器之后执行的 /bin/bash，就是 /bin 目录下的可执行文件，与宿主机的 /bin/bash 完全不同。
此时我们已进入一个 ubuntu15.10系统的容器
我们尝试在容器中运行命令 cat /proc/version和ls分别查看当前系统的版本信息和当前目录下的文件列表.   
我们可以通过运行exit命令或者使用CTRL+D来退出容器。

&emsp;

* 运行守护式的容器（能够长期运行，没有交互式会话，适合运行应用程序和服务）
使用以下命令创建一个以进程方式运行的容器
`docker run -d ubuntu:15.10 /bin/sh -c "while true; do echo hello world; sleep 1; done"`
> 在输出中，我们没有看到期望的"hello world"，而是一串长字符  
> 这个长字符串叫做容器ID，对每个容器来说都是唯一的    

我们需要确认容器有在运行，可以通过 docker ps 来查看  
`docker ps`
> CONTAINER ID:容器ID  
> NAMES:自动分配的容器名称
> -a: 查看所有的容器
&emsp;

* 在容器内使用docker logs命令，查看容器内的标准输出  
`docker logs`

* 停止容器
`docker stop`

* 删除容器
`docker rm`



## Docker 镜像使用

* 使用 `docker images` 来列出本地主机上的镜像
> 各个选项说明:
> REPOSITORY：表示镜像的仓库源. (eg: ubuntu)
> TAG：镜像的标签(可以有多个,代表这个仓库源的不同个版本,默认选择最新的,eg: 15.10)
> IMAGE ID：镜像ID


* 创建镜像

 当我们从docker镜像仓库中下载的镜像不能满足我们的需求时，我们可以通过以下两种方式对镜像进行更改:

  1.使用 Dockerfile 指令来创建一个新的镜像
  * 构建镜像 
  `docker build` 	
  零开始来创建一个新的镜像。为此，我们需要创建一个 Dockerfile 文件，其中包含一组指令来告诉 Docker 如何构建我们的镜像。  
  eg: `docker build --tag=hi .`
  > 注意命令的最后有一个.，这个表示打包的上下文（其实就是Dockerfile所在目录）是在当前目录，然后目录下的Dockerfile就会被编译执行。再执行`docker images` 可以看到新增的结果

  * 使用新的镜像运行 
  `docker run -d -p 4000:80 hi python app.py` 
  > -p 4000:80 :指定端口 
  > 此时在浏览器中输入http://localhost:4000 可看到运行的效果；或者输入`curl -4 http://localhost:4000`

  * 查看容器信息
  ` docker container ls`  
  
  * 标记图像格式 
   ` docker tag image repository:tag`
  > eg: ` docker tag hi zhanqq/test:v1`

  * 提交到docker hub
  登录：`docker login`
  `docker push repository:tag`
  > eg: `docker push zhanqq/test:v1`  
  前往: https://cloud.docker.com/repository/list 可以查看

  &emsp; 
  
  2.从已经创建的容器中更新镜像，并且提交这个镜像
  * 使用 `docker search` 	查询镜像
  > eg: `docker search zhanqq`
  > 各个选项说明: 
  > NAME:镜像仓库源的名称.  
  > DESCRIPTION:镜像的描述.  
  > OFFICIAL:是否docker官方发布. 

  * 使用 `docker pull` 	获取新镜像
  > eg: `docker pull zhanqq/qq-repositoty:part1`
  > 下载之后 可以使用`docker run`运行上述操作 

  * 设置镜像标签  
  `docker tag 21f7d3330ee0 zhanqq/qq-repositoty:part2` 
  > 此时的IMAGE ID 是相同的

  * 更新镜像
  `docker run -t -i  zhanqq/qq-repositoty:part2 /bin/bash`
  > 使用 `apt-get update` 进行更新
  > 在完成操作之后，输入 exit命令来退出这个容器

  * 提交容器副本
  `docker commit  -m="has update" -a="zhanqq" e18ef951a0f7 test`
  > -m:提交的描述信息
  > -a:指定镜像作者
  > e18ef951a0f7：容器ID
  > test :指定要创建的目标镜像名 
  > `docker images` 此时的IMAGE ID 是不相同的

  * 使用新的镜像运行 
  `docker run -d -p 4000:80 test python app.py` 
  > -p 4000:80 :指定端口 
  > 此时在浏览器中输入http://localhost:4000 可看到运行的效果；或者输入`curl -4 http://localhost:4000` 

  同上述方法，也能同步到Docker Hub上

  * 删除镜像
  `docker rmi` 
  > 后面可以IMAGE ID 或者 REPOSITORY 或者 REPOSITORY:TAGxit
  > 删除所有的镜像 `docker image rm $(docker image ls -a -q）` 
  > 先暂停使用了的镜像的container

  &emsp;
  ***
  &emsp;


