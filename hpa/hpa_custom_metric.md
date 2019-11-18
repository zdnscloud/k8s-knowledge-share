# HPA custom metric
## 概要
HPA通过custom.metrics.k8s.io API server 获取custom metric信息，然后根据custom metric对workload的pods做自动扩缩，本文以zdnscloud/vanguard案例，以vanguard的zdns_vanguard_qps指标做custom metric参数，来演示HPA对pods做扩缩的过程

## 版本说明
* zcloud版本：hpa-custom_metric
* prometheus-adapter chart version: 1.4.0
* vanguard chart version: 0.2.0

## 安装和测试
* 使用hpa-custom_metric版本的zcloud安装集群
* 安装lvm存储
* 安装监控prometheus
* 查看监控service的名字，monitor-ktzxfuoikmjm-prome-prometheus
* 查看prometheus-adapter chart的configmap文件，其中包含zdns_vanguard_.*的名字匹配规则，来匹配以zdns_vanguard_开头的custom metric

		chart/prometheus-adapter/1.4.0/templates/custom-metrics-configmap.yaml
		
		- seriesQuery: '{__name__=~"^zdns_vanguard_.*",kubernetes_pod_name!="",kubernetes_namespace!=""}'
		  seriesFilters: []
		  resources:
		    overrides:
		      kubernetes_namespace:
		        resource: namespace
		      kubernetes_pod_name:
		        resource: pod
		  name:
		    matches: ^zdns_vanguard_.*$
		    as: ""
		  metricsQuery: sum(<<.Series>>{<<.LabelMatchers>>}) by (<<.GroupBy>>)
	
* 在应用商店安装prometheus-adapter，需要参数为prometheusServiceName，使用prometheus的service名字来组成url，即http://monitor-ktzxfuoikmjm-prome-prometheus.zcloud.svc
* 在应用商店中安装vanguard

		kubectl get deploy
		NAME       READY   UP-TO-DATE   AVAILABLE   AGE
		vanguard   1/1     1            1           2m

* 创建HPA

		{	
			"name": "vg1",
    		"scaleTargetKind": "deployment",
    		"scaleTargetName": "vanguard",
    		"minReplicas": 1,
    		"maxReplicas": 10,
    		"customMetrics":[
    			{
    				"metricName": "zdns_vanguard_qps",
    				"averageValue": "2000"
    			}
    		]
    	}
* 使用dnsperf向vanguard打压力, 如vanguard service的ip为10.43.220.18，qps为5000

		./dnsperf -s 10.43.220.18 -d a.txt -l 600 -Q 5000 -c 10

* 查看vanguard deployment, pod的个数变成了3个

		kubectl get deploy
		NAME       READY   UP-TO-DATE   AVAILABLE   AGE
		vanguard   3/3     3            3           5m

* 通过metric api直接获取pods的qps值

		kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/zdns_vanguard_qps"
		{"kind":"MetricValueList","apiVersion":"custom.metrics.k8s.io/v1beta1","metadata":{"selfLink":"/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/%2A/zdns_vanguard_qps"},"items":[{"describedObject":{"kind":"Pod","namespace":"default","name":"vanguard-ddbb69fc8-6cbhc","apiVersion":"/v1"},"metricName":"zdns_vanguard_qps","timestamp":"2019-11-18T10:50:38Z","value":"3007"},{"describedObject":{"kind":"Pod","namespace":"default","name":"vanguard-ddbb69fc8-8lz5w","apiVersion":"/v1"},"metricName":"zdns_vanguard_qps","timestamp":"2019-11-18T10:50:38Z","value":"1500"},{"describedObject":{"kind":"Pod","namespace":"default","name":"vanguard-ddbb69fc8-blr6p","apiVersion":"/v1"},"metricName":"zdns_vanguard_qps","timestamp":"2019-11-18T10:50:38Z","value":"498"}]}
		
可以看到三个pod的qps分别为3007、1500、498，即 （3007+1500+498）/ 3 = 1668

* 查看hpa和hpa.status，平均值为三个pods的qps平均值

		kubectl get hpa
		NAME   REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
		vg1    deployment/vanguard   1668/2k   1         10        3          5m51s
		
		"status": {
			"currentReplicas": 2,
			"desiredReplicas": 2,
			"currentMetrics": {
				"customMetrics": [
					{
						"metricName": "zdns_vanguard_qps",
						"averageValue": "1668"
                    }
                ]
           }
        }

* 停止dns压力， 由于HPA在减少pods数值时有延时5min，即5min后，vanguard pods个数会变成minReplicas值，hps.status如下：

		kubectl get deploy
		NAME       READY   UP-TO-DATE   AVAILABLE   AGE
		vanguard   1/1     1            1           15m

		kubectl get hpa
		NAME   REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
		vg1    deployment/vanguard   0/2k      1         10        1          11m

		"status": {
			"currentReplicas": 1,
			"desiredReplicas": 1,
			"currentMetrics": {
				"customMetrics": [
					{
						"metricName": "zdns_vanguard_qps",
						"averageValue": "0"
                    }
                ]
           }
        }



