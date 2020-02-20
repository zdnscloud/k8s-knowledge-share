# Docker buildx
## 是什么
Buildx是一个Docker CLI插件，扩展了docker build命令，并完全支持Moby BuildKit构建器工具箱提供的功能。
> 启用docker buildx后，首次buildx build时，会在本地起一个buildkit server的容器，作为buildx的后端
## 用buildx构建镜像
```bash
$ docker buildx build .
[+] Building 8.4s (23/32)
 => ...
 ```
 Buildx构建命令支持可用于docker build包括Docker 19.03中的新功能的功能，例如输出配置，内联构建缓存或指定目标平台。此外，buildx还支持常规功能尚不可用的新功能，docker build例如构建清单清单，分布式缓存，将构建结果导出到OCI映像tarball等。

Buildx应该是灵活的，并且可以在通过驱动程序概念公开的不同配置中运行。当前，我们支持使用绑定到docker守护进程二进制文件中的BuildKit库的“ docker”驱动程序，以及自动在Docker容器内启动BuildKit的“ docker-container”驱动程序。我们计划在将来添加更多驱动程序，例如，将允许在一个（非特权）容器内运行buildx的驱动程序。

跨驱动程序使用buildx的用户体验非常相似，但是“ docker”驱动程序当前不支持某些功能，因为捆绑到docker守护程序中的BuildKit库当前使用不同的存储组件。相反，默认情况下，所有使用“ docker”驱动程序构建的映像都会自动添加到“ docker images”视图中，而在使用其他驱动程序时，需要使用来选择输出映像的方法--output。
## 使用构建器实例
默认情况下，如果支持，buildx最初将使用“ docker”驱动程序，从而提供与native非常相似的用户体验docker build。但是，使用本地共享守护程序只是构建应用程序的一种方法。

Buildx允许您创建隔离的构建器的新实例。这可用于为您的CI构建获取范围内的环境，该环境不会更改共享守护程序的状态，也可用于隔离不同项目的构建。您可以为一组远程节点创建一个新实例，形成一个构建场，并在它们之间快速切换。

可以使用docker buildx create命令创建新实例。这将基于当前配置创建一个具有单个节点的新构建器实例。要使用远程节点，可以DOCKER_HOST在创建新构建器时指定或远程上下文名称。创建一个新的实例后，你可以管理它与生命周期inspect，stop并rm命令并列出所有可用的建设者ls。创建新的构建器之后，您还可以向其附加新节点。

要在不同的构建器之间切换，请使用docker buildx use <name>。运行此命令后，构建命令将自动继续使用此构建器。

Docker 19.03还具有一个新docker context命令，可用于为远程Docker API端点提供名称。Buildx与之集成，docker context以便您的所有上下文自动获得默认的构建器实例。在创建新的构建器实例或向其添加节点时，还可以将上下文名称设置为目标。
## 构建多平台镜像
在build时指定`--platform`标志，需先启用宿主机上的binfmt_misc，并至少创建一个构建器实例
> dbuildx默认构建器不启用多平台构建
## 高级构建选项
Buildx还旨在为高级构建概念提供支持，而不仅仅是调用单个构建命令。我们希望支持一起构建应用程序中的所有映像，并让用户定义项目特定的可重用构建流程，然后任何人都可以轻松调用它们。

BuildKit为有效处理多个并发构建请求和重复数据删除工作提供了强大的支持。尽管可以将构建命令与通用命令运行程序（例如make）组合使用，但这些工具通常会按顺序调用构建，因此无法充分利用BuildKit并行化的全部潜力或为用户组合BuildKit的输出。对于此用例，我们添加了一个名为的命令docker buildx bake。

当前，bake命令支持从撰写文件构建映像，类似于compose build但允许将所有服务作为单个请求的一部分并发构建。

还支持HCL / JSON文件中的自定义构建规则，从而可以更好地重复使用代码和使用不同的目标组。bake的设计还处于初期阶段，我们正在寻求用户的反馈。




