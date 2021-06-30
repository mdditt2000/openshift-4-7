# OpenShift 4.7 and Container Ingress Controller Quick User-Guide for Standalone BIG-IP 

This user guide is create to document OpenShift 4.7 integration of CIS and BIG-IP. This user guide provides configuration for a standalone BIG-IP configuration using OpenShift SDN

## Environment parameters

* OpenShift 4.7 on vSphere with installer-provisioned infrastructure
* CIS 2.5
* AS3: 3.28
* BIG-IP 16.0.1.1

## Prerequisite

Since CIS is using the AS3 declarative API we need the AS3 extension installed on BIG-IP. Follow the link to install AS3
 
* Install AS3 on BIG-IP
https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

## Create a BIG-IP VXLAN tunnel

```
(tmos)# create net tunnels vxlan vxlan-mp flooding-type multipoint
(tmos)# create net tunnels tunnel openshift_vxlan key 0 profile vxlan-mp local-address 192.168.200.60
```

![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone/diagram/2021-06-29_15-42-04.png)

Create a host subnet for the BIPIP. This will provide the subnet for creating the tunnel self-IP

```
oc create -f f5-openshift-hostsubnet.yaml
```

```
# oc get hostsubnet
NAME                        HOST                        HOST IP          SUBNET          EGRESS CIDRS   EGRESS IPS
f5-server                   f5-server                   192.168.200.60   10.131.2.0/23
ocp-pm-bwmmz-master-0       ocp-pm-bwmmz-master-0       10.192.75.229    10.130.0.0/23
ocp-pm-bwmmz-master-1       ocp-pm-bwmmz-master-1       10.192.75.231    10.129.0.0/23
ocp-pm-bwmmz-master-2       ocp-pm-bwmmz-master-2       10.192.75.230    10.128.0.0/23
ocp-pm-bwmmz-worker-9ch4b   ocp-pm-bwmmz-worker-9ch4b   10.192.75.234    10.129.2.0/23
ocp-pm-bwmmz-worker-lws6s   ocp-pm-bwmmz-worker-lws6s   10.192.75.235    10.131.0.0/23
ocp-pm-bwmmz-worker-qdhgx   ocp-pm-bwmmz-worker-qdhgx   10.192.75.233    10.128.2.0/23
```
f5-openshift-hostsubnet.yaml [repo](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone/cis/f5-openshift-hostsubnet.yaml)

## Add the BIG-IP device to the OpenShift overlay network 
```
(tmos)# create net self 10.131.2.60/14 allow-service all vlan openshift_vxlan
```
Subnet comes from the creating the hostsubnet. Used .60 to be consistent with BigIP internal interface

## Create a new partition on your BIG-IP system
```
(tmos)# create auth partition openshift
```
This needs to match the partition in the controller configuration

## Create CIS Controller, BIG-IP credentials and RBAC Authentication

```
args: [
        "--bigip-username=$(BIGIP_USERNAME)",
        "--bigip-password=$(BIGIP_PASSWORD)",
        # Replace with the IP address or hostname of your BIG-IP device
        "--bigip-url=192.168.200.83",
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
## Enable logging for AS3
```
oc get pod -n kube-system
oc log -f f5-server-### -n kube-system | grep -i 'as3'
```