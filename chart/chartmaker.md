# chart制作
## 目录结构
每个chart文件夹都是以chart名字命名， chart文件夹下为每个版本的文件夹，版本文件夹下面是每个版本的具体chart文件，包含Chart.yaml, values.yaml, config文件夹和templates模版文件夹，其中Chart.yaml为chart基础信息，values为用来渲染模版的参数，模版文件夹下存放生成k8s资源的yaml文件，config文件夹下的config.json为暴露出来的可以修改的参数，如果不需要暴露任何参数，可以不需要values.yaml和config文件夹，如vanguard的chart包含三个版本，具体目录结构如下：

	vanguard
    ├── 0.1.0
    │   ├── Chart.yaml
    │   ├── config
    │   │   └── config.json
    │   ├── templates
    │   │   ├── clusterrolebinding.yaml
    │   │   ├── clusterrole.yaml
    │   │   ├── configmap.yaml
    │   │   ├── deployment.yaml
    │   │   ├── _helpers.tpl
    │   │   ├── serviceaccount.yaml
    │   │   └── service.yaml
    │   └── values.yaml
    ├── 0.1.1
    │   ├── Chart.yaml
    │   ├── config
    │   │   └── config.json
    │   ├── templates
    │   │   ├── clusterrolebinding.yaml
    │   │   ├── clusterrole.yaml
    │   │   ├── configmap.yaml
    │   │   ├── deployment.yaml
    │   │   ├── _helpers.tpl
    │   │   ├── serviceaccount.yaml
    │   │   └── service.yaml
    │   └── values.yaml
    └── 0.1.2
        ├── Chart.yaml
        └── templates
            └── vanguard_deploy.yaml
            
## Chart.yaml
chart的基本信息文件, 其中appVersion为使用的image版本，version为chart版本，例如vanguard的基础信息如下：

	apiVersion: v1
	appVersion: "0.1.0"
	description: Deploy a simple dns server
	home: https://github.com/zdnscloud/vanguard
	name: vanguard
	sources:
	- https://github.com/zdnscloud/vanguard
	version: 0.1.0

## _helpers.tpl
模版字段，也可以用来渲染templates下的模版文件，如在 _helpers.tpl定义

	{{- define "vanguard.name" -}}
	{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
	{{- end -}}
	
在模版文件中的使用方式如下：
	
	app: {{ template "vanguard.name" . }}
	
## Release对象
release为代码中定义的对象，包含所创建application的Name、Namespace和Service等信息，其中Service为zcloud，表示此application由zcloud创建，在模版文件中引用这些值格式如下：

	labels:
	  heritage: {{ .Release.Service }}
      release: {{ .Release.Name }}
     
	
## values.yaml
用来渲染templates中的模版文件，如在values.yaml文件中定义

	deployment：
	  image：
	    repository: zdnscloud/vanguard:0.1.0

如果想在模版文件中使用repository的值，需要在最前面加 .Values, 具体格式如下
	
	image: "{{ .Values.deloyment.image.repository }}"
	或者
	image: {{ .Values.deloyment.image.repository | quote }}
	
如果想使用values.yaml定义一个的一个对象如下：
	
	affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - kafka
          topologyKey: "kubernetes.io/hostname"
          
可以使用toYaml来使用values.yaml的affinity值
 
	affinity:
	{{ toYaml .Values.affinity | indent 8 }}
	
如果使用values.yaml定义的数组：

	master:
	  persistence:
	    accessModes:
	    - ReadWriteOnce
	    
可以使用range方式来使用values.yaml的accessModes值

	accessModes:
	{{- range .Values.master.persistence.accessModes }}
	  - {{ . | quote }}
	{{- end }}

## config.json
如果想把values.yaml的参数暴露出来供用户修改，可以在config/config.json中定义，config.json中为json对象的数组，每个对象定义一个字段，对此字段进行定义和描述，label为字段名字，供web使用，jsonKey对应values.yaml的字段名字，type为字段类型，required表示字段是否必填，如果字段required为false，且没有值，那么渲染模版文件时将使用values.yaml中的默认值

目前仅支持三种字段类型的检查：int、string和enum
 
  * int 类型支持min、max
  * string 类型支持minLen、maxLen
  * enum 类型必须在validValues范围内


		[
	    	{"label": "hdfsDataNodeReplicas", "jsonKey": "hdfs.dataNode.replicas", "type": "int", "required": false, "description": "hdfs datanode replicas",  "min": 1, "max": 10},
			{ "label": "dataNodeStorageClass", "jsonKey": "persistence.dataNode.storageClass", "type": "enum", "required": false, "description": "datanode storageclass", "validValues": ["cephfs", "lvm"]},
			{ "label": "zeppelinDriverMemory", "jsonKey": "zeppelin.spark.driverMemory", "type": "string", "required": false, "description": "zeppelin driverMemory", "minLen": 2, "maxLen": 3}
		]
 