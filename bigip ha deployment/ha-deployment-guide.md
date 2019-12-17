# OpenShift 3.11 and Container Ingress Controller with BIGIP HA Quick Start Guide 

This page is created to document OCP 3.11 with integration of CIS and BIGIP in HA. This guide documents dual CIS controller with BIGIPs configured in HA. Please open issues on my github page on contact me at m.dittmer@f5.com

# Note

Environment parameters

* OCP 3.11 - one master and two worker nodes
* CIS 1.12
* AS3: 3.16
* 2 BIG-IP 14.1.2.2

# OpenShift 3.11 Install

OCP is installed on RHEL 7.5 on ESXi

* ose-3-11-master  
* ose-3-11-node1
* ose-3-11-node2

## Prerequisite

Since CIS is using the AS3 declarative API we need the AS3 extension installed on BIGIP. Follow the link to install AS3
 
* Install AS3 on BIGIP
https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

## Step 1: Create a new OpenShift HostSubnet

Create one HostSubnet for each BIG-IP device. These will handle health monitor traffic. Also create one HostSubnet to pass data traffic using floating IP address. **Note** all files can be located under the bigip ha deployment folder. Recommend cloning this folder and making the modifications

```
oc create -f f5-kctlr-openshift-hostsubnet-ose-bigip-01.yaml
oc create -f f5-kctlr-openshift-hostsubnet-ose-bigip-02.yaml
oc create -f f5-kctlr-openshift-hostsubnet-float.yaml
```
```
[root@ose-3-11-master bigip ha deployment]# oc get hostsubnets
NAME                               HOST                               HOST IP          SUBNET          EGRESS CIDRS   EGRESS IPS
f5-ose-bigip-01                    f5-ose-bigip-01                    192.168.200.82   10.131.0.0/23   []             []
f5-ose-bigip-02                    f5-ose-bigip-02                    192.168.200.83   10.128.2.0/23   []             []
f5-ose-float                       f5-ose-float                       192.168.200.81   10.129.2.0/23   []             []
ose-3-11-master.lab.fp.f5net.com   ose-3-11-master.example.com        192.168.200.84   10.128.0.0/23   []             []
ose-3-11-node1.lab.fp.f5net.com    ose-3-11-node1.example.com         192.168.200.85   10.130.0.0/23   []             []
ose-3-11-node2.lab.fp.f5net.com    ose-3-11-node2.example.com         192.168.200.86   10.129.0.0/23   []             []
```
## Step 2: Create a VXLAN profile

Create a VXLAN profile that uses multi-cast flooding on each BIGIP
```
(tmos)# create net tunnels vxlan vxlan-mp flooding-type multipoint

```
## Step 3: Create a VXLAN tunnel

Set the key to 0 to grant the BIG-IP device access to all OpenShift projects and subnets

```
on ose-bigip-01 create the VXLAN tunnel
(tmos)# create /net tunnels tunnel openshift_vxlan key 0 profile vxlan-mp local-address 192.168.200.81 secondary-address 192.168.200.82 traffic-group traffic-group-1
```
```
On ose-bigip-02 create the VXLAN tunnel
(tmos)# create /net tunnels tunnel openshift_vxlan key 0 profile vxlan-mp local-address 192.168.200.81 secondary-address 192.168.200.83 traffic-group traffic-group-1
```
## Step 4: Create a self IP in the VXLAN

Create a self IP address in the VXLAN on each device. The subnet mask you assign to the self IP must match the one that the OpenShift SDN assigns to nodes. **Note** that is a /14 by default. Be sure to specify a floating traffic group (for example, traffic-group-1). Otherwise, the self IP will use the BIG-IP system’s default

```
[root@ose-3-11-master bigip ha deployment]# oc get hostsubnets
NAME                               HOST                               HOST IP          SUBNET          EGRESS CIDRS   EGRESS IPS
f5-ose-bigip-01                    f5-ose-bigip-01                    192.168.200.82   10.131.0.0/23   []             []
f5-ose-bigip-02                    f5-ose-bigip-02                    192.168.200.83   10.128.2.0/23   []             []
f5-ose-float                       f5-ose-float                       192.168.200.81   10.129.2.0/23   []             []
```
On ose-bigip-01 create the self IP
```
(tmos)# create /net self 10.131.0.82/14 allow-service none vlan openshift_vxlan
```
On ose-bigip-02 create the self IP
```
(tmos)# create /net self 10.128.2.83/14 allow-service none vlan openshift_vxlan
```
On the active BIGIP, create a floating IP address in the subnet assigned by the OpenShift SDN
```
(tmos)# create /net self 10.129.2.81/14 allow-service none traffic-group traffic-group-1 vlan openshift_vxlan
```
## Step 5: Create a new partition on your BIG-IP system

Create a new partition on your BIG-IP system
```
(tmos)# create auth partition openshift
```
## Create CIS Controller, BIG-IP credentials and RBAC Authentication

Deploy both ose-bigip-01 and ose-bigip-02 controller for the associated BIGIPs. Example below from oc delete ose-bigip-01

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
        # Logging level
        "--log-level=DEBUG",
        "--log-as3-response=true",
        # AS3 override functionality
        "--override-as3-declaration=f5demo/f5-route-vs-override",
        # Self-signed cert
        "--insecure=true",
        "--agent=as3"
       ]
```
```
oc create secret generic bigip-login --namespace kube-system --from-literal=username=admin --from-literal=password=f5PME123
oc create serviceaccount bigip-ctlr -n kube-system
oc create -f f5-kctlr-openshift-clusterrole.yaml
oc create -f f5-k8s-bigip-ctlr-openshift-ose-bigip-01.yaml
oc create -f f5-k8s-bigip-ctlr-openshift-ose-bigip-02.yaml
oc adm policy add-cluster-role-to-user cluster-admin -z bigip-ctlr -n kube-system
```
## Delete kubernetes bigip container connecter, authentication and RBAC
```
oc delete serviceaccount bigip-ctlr -n kube-system
oc delete -f f5-kctlr-openshift-clusterrole.yaml
oc delete -f f5-k8s-bigip-ctlr-openshift-ose-bigip-01.yaml
oc delete -f f5-k8s-bigip-ctlr-openshift-ose-bigip-02.yaml
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