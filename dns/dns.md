# pod DNSPolicy
1. ClusterFirst
  * use CoreDNS
``` resolv.conf
nameserver 10.43.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
# if label of domain is shorter than 5, the domain will be append 
# with search list before first try, if all search list failed
# the orginal name will be sent at last
options ndots:5 
```
2. Default
  * resolv.conf in host 
  * if resolv.conf is empty, use Google public DNS (8.8.8.8, 8.8.4.4) 
3. ClusterFirstWithHostNet
  * When hostNetwork is set to true, ClusterFirst will be overwrite to Default
  * ClusterFirstWithHostNet means even hostNetwork is set, still use CoreDNS
4. None

# pod DNSConfig 
``` yaml
dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster.local
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```
  * configuration will be merged with configure generated based on DNSPolicy

# Service Discovery
  * New domain is introduced to DNS system by creating new Service
  * Headless service (ClusterIP == "None") also create the domain for the Pod it related
   + Pod created by StatefulSet 
     * [pod-name].[svc-name].[namespace-name].svc.[cluster-dns]
   + Pod with hostName and subdomain, and subdomain is equal to svc-name
     * [host-name].[svc-name].[namespace-name].svc.[cluster-dns]


# Service DNS record
1. FQDN
  * [svc-name].[namespace-name].svc.[cluster-dns]
2. A
  * [svc-FQDN] A TTL ClusterIP
  * for headless service (ClusterIP is None), all the pod in ready state will be
    included in A record
3. PTR
  * [reverse-ip].in-addr.arpa. ttl IN PTR [svc-FQDN]
4. SRV
  * named port defined in service Spec
  * [port-name].[port-protocol].[svc-name].[namespace-name].svc.[cluster-dns]
  * Domain in Rdata part is the Endpoint FQDN
5. CNAME
  * external type service
  * [svc-FQDN] CNAME [svc-externalName]

# outer service performance
when query outside service which isn't subdomain of the cluster domain, affected by ndots
in resolv.conf, if the query name has shorter dot, it will go through the search list which
make the dns query very slow.

CoreDNS support autopath, it will return cname to avoid the next try through search list
```
www.google.com.<namespace>.svc.cluster.local cname www.google.com
www.google.com A 1.1.1.1
```
But autopath may always query the upstream dns first, and it won't cache the result, which may
cause problem when upstream server has query rate limit.
