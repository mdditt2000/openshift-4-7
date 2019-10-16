# OpenShift 3.11 and Container Ingress Controler Quick Start Guide

This page is created to document OCP 3.11 with intergration of CIS and BIGIP. Please open issues on my github page on contact me at m.dittmer@f5.com

# Note

Environment parameters

* OCP 3.11 - one master and two worker nodes
* CIS 1.11
* AS3: 3.13.1 LTS
* BIG-IP 14.1

# OpenShift 3.11 Install

OCP is installed on RHEL 7.5 on ESXi using a mutli-nic deployment

* ose-3-11-master  
* ose-3-11-node1
* ose-3-11-node2

## Prerequisite

Since CIS is using the AS3 declarative API we need the AS3 extention installed on BIGIP. Follow the link to install AS3
 
* Install AS3 on BIGIP
https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

## Create a new OpenShift HostSubnet

Create a host subnet for the BIPIP. This will provide the subnet for creting the tunnel self-IP

```
oc create -f f5-kctlr-openshift-hostsubnet.yaml
```
```
[root@ose-3-11-master openshift-3-11]# oc get hostsubnets
NAME                               HOST                               HOST IP          SUBNET          EGRESS CIDRS   EGRESS IPS
f5-server                          f5-server                          192.168.200.82   10.128.2.0/23   []     []
ose-3-11-master.example.com        ose-3-11-master.example.com        192.168.200.84   10.128.0.0/23   []     []
ose-3-11-node1.example.com         ose-3-11-node1.example.com         192.168.200.85   10.130.0.0/23   []     []
ose-3-11-node2.lexample.com        ose-3-11-node2.example.com         192.168.200.86   10.129.0.0/23   []     []
```
## Create a BIG-IP VXLAN tunnel

## create net tunnels vxlan vxlan-mp flooding-type multipoint
```
(tmos)# create net tunnels vxlan vxlan-mp flooding-type multipoint
(tmos)# create net tunnels tunnel openshift_vxlan key 0 profile vxlan-mp local-address 192.168.200.82
```
## Add the BIG-IP device to the OpenShift overlay network
```
(tmos)# create net self 10.128.2.82/14 allow-service all vlan openshift_vxlan
```
Subnet comes from the creating the hostsubnets. Used .82 to be consisten with BigIP internal interface

## Create a new partition on your BIG-IP system
```
(tmos)# create auth partition openshift
```
This needs to match the partition in the controller contiguration

## Create CIS Controller, BIG-IP credentials and RBAC Authentication

```
args: [
        "--bigip-username=$(BIGIP_USERNAME)",
        "--bigip-password=$(BIGIP_PASSWORD)",
        # Replace with the IP address or hostname of your BIG-IP device
        "--bigip-url=192.168.200.82",
        # Replace with the name of the BIG-IP partition you want to manage
        "--bigip-partition=openshift",
        "--pool-member-type=cluster",
        # Replace with the path to the BIG-IP VXLAN connected to the
        # OpenShift HostSubnet
        "--openshift-sdn-name=/Common/openshift_vxlan",
        "--manage-routes=true",
        "--namespace=f5demo",
        "--route-vserver-addr=10.192.75.107",
        "--log-level=DEBUG",
        # Self-signed cert
        "--insecure=true",
        "--agent=as3"
       ]
```
```
oc create secret generic bigip-login --namespace kube-system --from-literal=username=admin --from-literal=password=f5PME123
oc create serviceaccount bigip-ctlr -n kube-system
oc create -f f5-kctlr-openshift-clusterrole.yaml
oc create -f f5-k8s-bigip-ctlr-openshift.yaml
oc adm policy add-cluster-role-to-user cluster-admin -z bigip-ctlr -n kube-system
```

## Notes

New features in CIS 1.11

* OpenShift WAF policy support
```
virtual-server.f5.com/waf: /Common/WAF_Policy
``` 
Note: Currently path / doesnt work. Please add a path such as /foo. Issue will be fixed in CIS 1.11.1

* Alternative backend, blue/green support using weight
Look for the example f5-demo-app-route-ab

* Route status update in OpenShift Dashboard
The router name must not be router

## Create openshift BIGIP controller authentication and RBAC
```
oc create secret generic bigip-login --namespace kube-system --from-literal=username=admin --from-literal=password=f5PME123
oc create serviceaccount bigip-ctlr -n kube-system
oc create -f f5-kctlr-openshift-clusterrole.yaml
oc create -f f5-k8s-bigip-ctlr-openshift.yaml
```
## Delete kubernetes bigip container connecter, authentication and RBAC
```
oc delete serviceaccount bigip-ctlr -n kube-system
oc delete -f f5-kctlr-openshift-clusterrole.yaml
oc delete -f f5-k8s-bigip-ctlr-openshift.yaml
oc delete secret bigip-login -n kube-system
```
## Create container f5-demo-app-route
```
oc create -f f5-demo-app-route-deployment.yaml -n f5demo
oc create -f f5-demo-app-route-service.yaml -n f5demo
oc create -f f5-demo-app-route-basic.yaml -n f5demo
oc create -f f5-demo-app-route-balance.yaml -n f5demo
oc create -f f5-demo-app-route-edge-ssl.yaml -n f5demo
oc create -f f5-demo-app-route-reencrypt-ssl.yaml -n f5demo
oc create -f f5-demo-app-route-passthrough-ssl.yaml -n f5demo
oc create -f f5-demo-app-route-waf.yaml -n f5demo
oc create -f f5-demo-app-route-ab.yaml -n f5demo
```
Please look for example files in my repo

## Delete container f5-demo-app-route
```
oc delete -f f5-demo-app-route-deployment.yaml -n f5demo
oc delete -f f5-demo-app-route-service.yaml -n f5demo
oc delete -f f5-demo-app-route-basic.yaml -n f5demo
oc delete -f f5-demo-app-route-balance.yaml -n f5demo
oc delete -f f5-demo-app-route-edge-ssl.yaml -n f5demo
oc delete -f f5-demo-app-route-reencrypt-ssl.yaml -n f5demo
oc delete -f f5-demo-app-route-passthrough-ssl.yaml -n f5demo
oc delete -f f5-demo-app-route-waf.yaml -n f5demo
oc delete -f f5-demo-app-route-ab.yaml -n f5demo
```