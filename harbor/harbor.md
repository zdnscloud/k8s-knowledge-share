# Harbor
## 简介
Harbor是一个开源的云原生的镜像仓库，用来存储、签名和镜像漏洞扫描。
Harbor通过提供可信、合规性、性能和互操作性来解决共同的调整。它填补了那些无法使用公共或基于云的镜像仓库的组织和应用程序以及希望得到跨云的一致体验的市场空白。
## 特性
* 安全性
    * 镜像漏洞扫描
    * 镜像签名及验证
* 管理
    * 多租户支持
    * 多实例间镜像拷贝
    * 可扩展的api和web ui
    * 身份验证和基于角色的访问控制
## 架构
微服务架构，按照功能划分，包含以下组件：
* chartmuseum：helm chart仓库，helm的一个子项目
* clair：镜像漏洞扫描，coreos的一个开源项目
* core：harbor核心组件提供用户验证、系统管理等功能
* database：postgresql，用于保存harbor的job、项目、用户权限等数据
* redis：数据缓存
* protal：ui
* registry：docker镜像仓库，docker官方项目
* jobservice：镜像拷贝、垃圾清理、漏洞扫描等定时执行的任务管理
* notary：镜像签名及验证，docker官方项目
![''](images/harbor-arch.png)
## 安装
官方提供两种部署方式：
* docker-compose：https://github.com/goharbor/harbor/releases
* helm chart：https://github.com/goharbor/harbor-helm
## 使用
### 推拉镜像
### 镜像扫描
### 镜像复制
### 项目管理
### Ldap联动
### API
