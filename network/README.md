网络流程
=========
1：根据configMap中cni-conf.json，生成/etc/cni/net.d/10-flannel.conflist <br />  
2：根据configMap中net-conf.json，生成/run/flannel/subnet.env <br />  
3：根据/etc/cni/net.d/10-flannel.conflist和 /run/flannel/subnet.env，分别调用/opt/cni/bin/flannel和/opt/cni/bin/portmap命令<br />  
4：bridge完成网卡的建立和绑定<br />  
5：host-local完成ip的分配<br />  
6：flanneld进程负责监听subnet事件，创建和删除ARP、FDB、路由表等，以达到Pod网络互通<br />  
