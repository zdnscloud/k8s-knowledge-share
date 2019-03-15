# kubernetes-private-registries使用方法
## 1 目的
允许在使用zke创建k8s cluseter时和在k8s cluster集群中创建pod时使用私有仓库中的镜像
## 2 配置方法

### 2.1 通过kubelet配置文件
通过在kubelet配置文件中指定private registries登录信息来实现在创建pod时自动认证拉取私有仓库镜像，配置示例如下：
```
root@k8s01-03:~# cat /var/lib/kubelet/config.json  |python -m json.tool
{
    "auths": {
        "docker.zdns.cn": {
            "auth": "d2FuZ3lhbndlaTpXYW5neWFud2VpMjAxOQ=="
        }
    }
}
```
`auth`部分为私有仓库用户名:密码的base64转码
```
root@k8s01-03:~# echo "d2FuZ3lhbndlaTpXYW5neWFud2VpMjAxOQ==" |base64 --decode
wangyanwei:Wangyanwei2019
```

### 2.2 通过k8s secret
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

### 2.3 zke配置private registries
* 通过`zke config`生成cluster.yml
* 修改`cluster.yml`中的`private_registries`参数
```
private_registries:
- url: docker.zdns.cn
  user: wangyanwei
  password: Wangyanwei2019
  is_default: false
```