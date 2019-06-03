# zke harbor配置及使用
## 文档目的
介绍harbor功能、及其在kubernetes集群中的配置使用方法
## harbor简介
Harbor is an an open source trusted cloud native registry project that stores, signs, and scans content. Harbor extends the open source Docker Distribution by adding the functionalities usually required by users such as security, identity and management. Having a registry closer to the build and run environment can improve the image transfer efficiency. Harbor supports replication of images between registries, and also offers advanced security features such as user management, access control and activity auditing.

Harbor is hosted by the CNCF.

### 主要功能
* 容器镜像仓库
* helm chart仓库
* 多用户、多权限
* 镜像扫描（集成clair）
## 使用示例
### 使用zke创建harbor
1. zke config时启用harbor registry
```
[+] Is enabled harbor registry (y/n)? [y]:
[+] Cluster registry disk capacity [50Gi]:
[+] Cluster registry ingress url [registry.kube-registry.cluster.local]:
```
harbor依赖于lvm本地存储，harbor会安装在kube-registry namespace下，具体配置也可以通过编辑cluster.yml进行修改，主要配置有：
* isenabled：是否启用harbor，默认启用
* registry_ingress_url：harbor ingress域名，通过此url对外提供服务
* notary_ingress_url：用于镜像签名服务的ingress域名，通过此url对docker client提供镜像签名验证服务，使用较少
* registry_disk_capacity：可用于镜像存储的磁盘大小，默认50Gi
* database_disk_capacity：db组件最大可使用的磁盘大小，默认5Gi
* redis_disk_capacity：redis组件最大可使用的磁盘大小，默认1Gi
* Chartmuseum_disk_capacity：chart组件用于存储helm chart的最大可用磁盘大小，默认5Gi
* jobservice_disk_capacity：jobservice组件最大可用的磁盘存储大小，默认1Gi

2. zke up创建集群时会自动在集群内部创建harbor

### 上传下载镜像
1. 在浏览器中登录harbor web页面，设置仓库及用户，并下载仓库注册证书（即registry ca证书），harbor默认用户名密码如下：
```
user:admin
password:Harbor12345
```

2. 将下载的ca证书导入docker客户端，常见docker客户端导入方式如下：
* macos
  - 打开钥匙串，并将下载的注册证书拖入钥匙串的证书中
  - 重启docker
  - 使用docker login命令测试能否登录成功harbor

* linux
  - 在/etc/docker/certs.d目录下创建以仓库名称命名的目录，例如`mkdir -p /etc/docker/certs.d/registry.kube-registry.cluster.w`，并将下载的ca证书文件上传至该目录下
  - 使用docker login命令测试能否登录成功harbor

3. 使用docker push和docker pull命令测试能否正常推拉镜像
### 在kubernetes中使用harbor

1. 通过zke设置集群harbor公共账户
修改`cluster.yml`中的`private_registries`部分配置
```
private_registries:
- url: registry.kube-registry.cluster.w
  user: readonlyuser
  password: Readonly123
```
zke在创建集群时会将此部分配置写入各节点kubelet配置文件（/var/lib/kubelet/config.json）中，一般用于配置harbor的公共只读账户，用于kubelet拉取公共镜像时使用


2. 通过创建pod时指定secret使用harbor
用于对权限要求较高的场景
* setup 1 创建私有仓库的secret资源
```
kubectl create secret docker-registry mydockerhub --docker-server=docker.zdns.cn --docker-username=wangyanwei --docker-password=Wangyanwei2019
```
yaml创建方式:
```
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJkb2NrZXIuemRucy5jbiI6eyJ1c2VybmFtZSI6Indhbmd5YW53ZWkiLCJwYXNzd29yZCI6Ildhbmd5YW53ZWkyMDE5IiwiYXV0aCI6ImQyRnVaM2xoYm5kbGFUcFhZVzVuZVdGdWQyVnBNakF4T1E9PSJ9fX0=
kind: Secret
```
data中的.dockerconfigjson为私有仓库url、用户名、密码的base64转码
```
root@k8s01-01:/home/cluster# echo "eyJhdXRocyI6eyJkb2NrZXIuemRucy5jbiI6eyJ1c2VybmFtZSI6Indhbmd5YW53ZWkiLCJwY6Ildhbmd5YW53ZWkyMDE5IiwiYXV0aCI6ImQyRnVaM2xoYm5kbGFUcFhZVzVuZVdGdWQyVnBNakF4T1E9PSJ9fX0=" |base64 --decode |python -m json.tool
{
    "auths": {
        "docker.zdns.cn": {
            "auth": "d2FuZ3lhbndlaTpXYW5neWFud2VpMjAxOQ==",
            "password": "Wangyanwei2019",
            "username": "wangyanwei"
        }
    }
}
```
* setup 2 在创建pod时指定secret
```
apiVersion: v1
kind: Pod
metadata:
  name: private-reg
spec:
  containers:
  - name: private-reg-container
    image: docker.zdns.cn/zdnscloud/hello-py
  imagePullSecrets:
  - name: mydockerhub
```
### 扫描镜像
harbor集成clair，支持手动、自动方式对镜像进行扫描，clair默认使用CVE作为漏洞数据库，定期会自动向CVE同步漏洞数据库，所以镜像扫描功能要求harbor能够访问互联网
