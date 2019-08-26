# Container failure
1. RestartPolicy in PodSpec
    - Always (default)
    - OnFailure
    - Never
2. Never can only be speicified in Pod 
> The StatefulSet "counter-test" is invalid:      
> spec.template.spec.restartPolicy: Unsupported value: "Never": supported values: "Always"`

3. Docker restart policy     
`docker run --restart=always redis`
    - no (default)
    - on-failure
    - always
    - unless-stopped

4. Kubelet handle restart by itself
![""](pleg.png)
    - docker retart policy is set to empty
    - PLEG(pod lifecycle event generator) 
    - PLEG poll docker periodically(1s) to /relist all container 
    - kubelet will restart the container
```
ben   0s    Warning   BackOff   Pod   Back-off restarting failed container
ben   0s    Normal   Pulled   Pod   Container image "bikecn81/counter" already present on machine
ben   0s    Normal   Created   Pod   Created container
ben   0s    Normal   Started   Pod   Started container
```

5. Liveness/Readiness probe  
    1. Liveness probe failure will cause the pod to be killed
    1. Readiness probe failure only lead to traffic redirection     

``` yaml
subsets:
- addresses:
  - ip: 10.42.0.16
    nodeName: worker1
    targetRef:
      kind: Pod
      name: coredns-7b598b6596-85lhp
      namespace: kube-system
      resourceVersion: "3776"
      uid: 2913f008-be6c-11e9-980b-0251ce620080
  notReadyAddresses:
  - ip: 10.42.0.17
    nodeName: worker1
    targetRef:
      kind: Pod
      name: coredns-7b598b6596-b9rng
      namespace: kube-system
      resourceVersion: "8167"
      uid: 291298bf-be6c-11e9-980b-0251ce620080
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
```

# Node failure
## Node crash/power off
```
default   0s    Normal   NodeNotReady   Node   Node worker2 status is now: NodeNotReady
```
- node-controller will monitor nodes and generate the events
- taint-controller will add taint to the node
- taint-controller will mark the pod status to terminating
- replicaset will reschedule the pod to new node
- statefulset won't reschedule the pod (force deletion)

## Node error
```
ben   0s    Warning   NetworkNotReady   Pod   network is not ready: [runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized]
```
- kubelet will report node isn't ready and change the node status
- same with node power/off


# Workload
- Deployment > ReplicaSet > Pod
    - lightweight
    - local temporary storage or shared pvc-pv (support ReadWriteMany or ReadOnlyMany)
- StatefulSet > Pod
    - ordered operations
    - stable,unique network id/name across restart
    - stable pv, attach to same pv even reschedule
    - headless let client direct connect to backend pod
- DaemonSet > Pod
    - node level functionality
