# OpenShift 3.11 and Container Ingress Controler Quick Start Guide

This page is created to document OCP 3.11 with intergration of CIS and BIGIP. Please open issues on my github page on contact me at m.dittmer@f5.com

# Note

Environment parameters

* OCP 3.11 - one master and two worker nodes
* CIS 1.11
* AS3: 3.13.1
* HA BIG-IP 14.1 - flannel overlay using floats

# OpenShift 3.11 Install

OCP is installed on RHEL 7.5 on ESXi using a mutli-nic deployment

* ose-3-11-master  
* ose-3-11-node1
* ose-3-11-node2

Innore----WORK IN PROGRESS

## Prerequisite 
 
* Install AS3 on BIGIP
https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

### AS3/CCCL arguments:

Add argument in the CIS controller deployment to use “as3” path with OpenShift. If using a configmap with AS3 you dont need this aurgument
 ```
 --agent=as3
 ```
To use “cccl” path then use the below argument
 ```
 --agent=cccl
```
If you using BIGIP self-signed certs please add or schema validation will fail
 ```
 --insecure=true
```
Example folder https://github.com/mdditt2000/openshift/tree/dev/cis-1-11

#Note: CCCL will be removed in the upcoming months. CCCL doesnt support BIG-IP v14.1

## Notes

New features in CIS 1.11

* OpenShift WAF policy support
```
virtual-server.f5.com/waf: /Common/WAF_Policy
```
Unable to use two different WAF Policy with two different routes (#110)
Modification/Removal of WAF Policy annotation doesn't get reflected on BIG IP (#109) 

* Alternative backend, blue/green support using weight
Look for the example f5-demo-app-route-ab

* Route status update in OpenShift Dashboard
The router name must not be router

## Create openshift BIGIP controller authentication and RBAC
```
oc create secret generic bigip-login --namespace kube-system --from-literal=username=admin --from-literal=password=admin
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