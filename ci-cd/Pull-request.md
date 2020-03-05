# Pull request

## 概述

在使用Git开发的时候，经常会遇到一些协作的问题，比如开发的功能需要经过审核后再合并，同时有多个请求合并需要的处理。
这个时候，就需要使用Pull request来处理。

## 使用场景对比

功能开发完成，需要测试合并

### Git操作：

开发人员通知测试和review对应分支
测试完成后测试通知开发
开发人员自己合并分支到master，如果遇到兼容问题可能需要重新测试和review
回归测试和master的兼容性
开发人员手动删除对应的分支，手动记录合并的功能内容

### Pull request

开发人员发起pull request请求
自动触发检查与master分支合并问题，通过则可以进行pull request
自动通知测试人员进行测试
自动通知review人员review
测试review完成后提交给项目管理
项目管理，同意则批准合并，自动合并到master
自动删除对应的开发分支，自动记录合并内容


通过对比可以看到，在pull request中，增加了对代码的自动审查和批准的流程，方便协作与管理。

根据以往经验，三人或以上同时开发时，开发合并很容易出现冲突，pull request提供里一个自动发现并处理的机制来解决协作问题。

Pull request 还提供了对于合并的审批流程，可以更加便捷的review代码，并且可以对合并功能进行审批，而不是像git那样，需要赋予所有开发人员所有权限。权限的限制有效的减少开发误操作的损失，更好的促进协作与互信。



## Github 中使用pull request

使用github命令行工具，[链接](https://hub.github.com/)

快速安装：

```
brew install hub
```

发起pull request, 在功能开发完成后

```
hub pull-request [-focpd] [-b BASE] [-h HEAD] [-r REVIEWERS ] [-a
       ASSIGNEES] [-M MILESTONE] [-l LABELS]
```

可以制定修复的issue，reviewer和分配归属，创建里程碑，打上label

填写pull request开发的功能或修复的issue，
之后会自动创建一个pull request，并返回pull request的URL

这个时候可以自动触发或者基于chat opts来触发ci系统，执行lint code，unit test，e2e test，并且生成报告，
ci系统可以根据结果给对应的pull request打上标签，标明是否完成，然后再提交给reviewer和测试与审核人员，
最终进行合并，并记录pull request信息内容，可以在之后发布版本的时候追踪版本包含的功能和修复的bug


### 流程举例

1. 需要开发一个功能或修复一个bug时，创建一个issue，label标记为`feature`或`bug`
2. 这时，对应开发接到任务，创建分支，进行开发
3. 开发完成后，发起pull request，指定对应的issue，指定reviewer来进行review，添加label: `need-review`
```
hub pull-request \
    -b <需要合并到的分支，通常是master> \
    -h <开发功能的分支，默认是当前分支> \
    -i <issue 序号，可以通过hub issue查看到> \
    -r <reviewers, 逗号分割，不能有空格> \
    -l <labels, 逗号分隔>
```
4. 运行对应的自动化检查，自动CI检查通过后@reviewer进行review，CI检查失败则返回开发修复
5. reviewer开始review，assign给自己，在review之后，添加label：`reviewed/approved`，@开发人员，
开发收到后，assign给自己，构建好对应镜像，@测试
若review意见需要修改，则添加label：`reviewed/need-optimize`，@开发人员，再进入步骤2
6. 测试看到pull request后，assign给自己，进行测试
7. 测试完成后，
成功添加label：`tested/passed`，@开发；
失败添加label：`tested/failed`，写失败说明，@开发。
8. 开发看到test/passed，沟通可以合并后，通过pull request合并，并删除对应分支

* 补充：
    * assign给自己，表示自己在处理这个任务，需要别人处理的时候@对应人员
    * 如果发现冲突，则需要将目标分支，合并过来，解决冲突后，提交推送到远端
    * 步骤4后，如果base分支（通常是master）发生变化，如果出现合并冲突，这个时候需要将master合并到分支，并解决冲突
    * Pull request合并可以使用squash and merge方式，会将分支修改所有提交合并为一个commit，使用pull request信息来代替commit信息，方便后续生成release node
    * hub 操作
    ```
    # 快速查看pull request列表
    hub pr list
    # checkout pull request分支
    hub pr checkout <pr number>
    # 快速查看issue
    hub issue
    ```
