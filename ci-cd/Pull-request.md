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



