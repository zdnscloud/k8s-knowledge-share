## Dockerfile 指令



* FROM	指定基础镜像
  * 格式 `FROM image:tag`
  FROM 指令必须指定且必须在其它指令之前，FROM 指令之后的的所有指令都依赖于所指定

* FROM…AS… 指定基础镜像的别名
  * 这是Docker 17.05及以上版本新出来的指令，其实就是给这个阶段的镜像起个别名：FROM ...(基础镜像) AS ...(别名)，在后面引用这个阶段的镜像时直接使用别名就可以了

* USER 设置用户
  * 该指令用于设置启动镜像时的用户或者 UID，写在该指令后的 RUN、CMD 以及 ENTRYPOINT 指令都将使用该用户执行命令


* MAINTAINER 指定维护者信息

* CMD 容器启动命令
  * 用于为执行容器提供默认值，每个 Dockerfile 只有一个 CMD 命令
> 如果指定了多个 CMD 命令，那么只有最后一条被执行，如果启动容器时指定了运行的命令，则会被覆盖掉 CMD 指定的指令

* RUN 执行命令
  * 后面跟的其实就是一些shell命令;
  * 可以通过 && 将这些脚本连接在一行执行，这么做的原因是为了减少镜像的层数，每多一行RUN都会给镜像增加一层

* ARG	设置构建参数
  * 这个指令可以进行一些宏定义
  * 比如我定义ENV JAVA_HOME=/opt/jdk，之后RUN后面的shell命令中的${JAVA_HOME}都会被/opt/jdk代替

* ENV	环境变量

> ARG 指令用于设置构建参数，类似与 ENV，不同的是，ARG 设置的是构建时的环境变量，在容器运行时是不会存在这些变量的

&emsp;

* COPY	复制文件
  * 比如 `COPY . /root/workspace/agent`表示将当前文件夹（.表示当前文件夹，即Dockerfile所在文件夹）的所以文件拷贝到容器的/root/workspace/agent文件夹中。
  * 通过--from参数也可以从前面阶段的镜像中拷贝文件过来，比如--from=builder表示文件来源不是本地文件系统，而是之前的别名为builder的容器

* ADD 复制文件 

> ADD指令不仅能够将构建命令所在的主机本地的文件或目录，而且能够将远程URL所对应的文件或目录，作为资源复制到镜像文件系统。
所以，可以认为ADD是增强版的COPY，支持将远程URL的资源加入到镜像的文件系统。

&emsp;

* VOLUME 指定挂载点
  * 该指令使容器中的一个目录具有持久化存储功能，该目录可被容器本身使用，也可以共享给其它容器
  * 当容器中的应用有持久化数据的需求时可以在 Dockerfile 中使用该指令

* WORKDIR	指定工作目录
  * 切换目录指令，类似于 cd 命令，写在该指令后的 RUN、CMD 以及 ENTRYPOINT 都将该工作目录作为当前目录，并执行指令

* ENTRYPOINT	镜像入口点
 * 镜像打包完成之后，使用`docker run`命令运行这个镜像时，其实就是执行这个ENTRYPOINT后面的可执行文件（一般是一个shell脚本文件）
 * 也可以通过["可执行文件", "参数1", "参数2"]这种方式来赋予可执行文件的执行参数，这个“入口”执行的工作目录也是WORKDIR后面的那个目录 

* LABEL 为镜像添加元数据
  * 格式 `LABEL key=value key=value  ...`
  * 使用 " 和 \ 转换命令


* EXPOSE 声明暴露的端口
 * 该指令的作用主要是帮助镜像使用者理解该镜像服务的守护端口，其次是当运行时使用随机映射时，会自动映射 EXPOSE 的端口


附: singlecloud-ui/Dockerfile

```go

FROM node:12-alpine as uibuild

RUN apk --no-cache add make

COPY . /singlecloud-ui
RUN cd /singlecloud-ui && make build

FROM alpine:latest

ARG version
ARG buildtime
ARG branch

LABEL ui.zcloud/version=$version ui.zcloud/buildtime=$buildtime ui.zcloud/branch=$branch

COPY --from=uibuild /singlecloud-ui/packages/ui/build /www

COPY  --from=uibuild /singlecloud-ui/packages/helm-icons  /www/helm/icons/

WORKDIR /www

```