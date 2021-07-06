# OpenShift 4.7 and F5 Container Ingress Services (CIS) User-Guide for BIG-IP Cluster

This user guide is create to document OpenShift 4.7 integration of CIS and BIG-IP cluster. This user guide provides configuration for a BIG-IP cluster configuration using OpenShift SDN. BIG-IP cluster is pre-configured using **manual with incremental sync**

![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/cluster/diagram/2021-07-06_13-18-55.png)

Demo on YouTube [video]()

## Environment parameters

* OpenShift 4.7 on vSphere with installer-provisioned infrastructure
* CIS 2.5
* AS3: 3.28
* BIG-IP 16.0.1.1 cluster

## Prerequisite

Since CIS is using the AS3 declarative API we need the AS3 extension installed on BIG-IP. Follow the link to install AS3
 
* Install AS3 on BIG-IP
https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

### Step 1: Create a VXLAN profile and tunnel

On bigip-01 create the VXLAN profile and tunnel

```
(tmos)# create net tunnels vxlan vxlan-mp flooding-type multipoint
(tmos)# create net tunnels tunnel openshift_vxlan key 0 profile vxlan-mp local-address 10.192.125.62 secondary-address 10.192.125.60 traffic-group traffic-group-1
```
![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/cluster/diagram/2021-07-06_13-07-43.png)

On bigip-02 create the VXLAN tunnel

```
(tmos)# create net tunnels tunnel openshift_vxlan key 0 profile vxlan-mp local-address 10.192.125.62 secondary-address 10.192.125.61 traffic-group traffic-group-1
```
![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/cluster/diagram/2021-07-06_13-08-24.png)

### Step 2: Create a new OpenShift HostSubnet

Create a host subnet for the BIP-IP. This will provide the subnet for creating the tunnel self-IP

    oc create -f f5-openshift-hostsubnet-01.yaml
    oc create -f f5-openshift-hostsubnet-02.yaml
    oc create -f f5-openshift-hostsubnet-float.yaml

```
# oc get hostsubnets
NAME                        HOST                        HOST IP         SUBNET          EGRESS CIDRS   EGRESS IPS
f5-server-01                f5-server-01                10.192.125.60   10.129.4.0/23
f5-server-02                f5-server-02                10.192.125.61   10.130.4.0/23
f5-server-float             f5-server-float             10.192.125.62   10.131.4.0/23
ocp-pm-bwmmz-master-0       ocp-pm-bwmmz-master-0       10.192.75.229   10.130.0.0/23
ocp-pm-bwmmz-master-1       ocp-pm-bwmmz-master-1       10.192.75.231   10.129.0.0/23
ocp-pm-bwmmz-master-2       ocp-pm-bwmmz-master-2       10.192.75.230   10.128.0.0/23
ocp-pm-bwmmz-worker-9ch4b   ocp-pm-bwmmz-worker-9ch4b   10.192.75.234   10.129.2.0/23
ocp-pm-bwmmz-worker-lws6s   ocp-pm-bwmmz-worker-lws6s   10.192.75.235   10.131.0.0/23
ocp-pm-bwmmz-worker-qdhgx   ocp-pm-bwmmz-worker-qdhgx   10.192.75.233   10.128.2.0/23
```

f5-openshift-hostsubnet.yaml [repo](https://github.com/mdditt2000/openshift-4-7/tree/master/cluster/cis)

### Step 3: Create a self IP in the VXLAN

Create a self IP address in the VXLAN on each device. The subnet mask you assign to the self IP must match the one that the OpenShift SDN assigns to nodes. **Note** that is a /14 by default. Be sure to specify a floating traffic group (for example, traffic-group-1). Otherwise, the self IP will use the BIG-IP system’s default

On bigip-01 create the self IP from hostsubnets **f5-server-01**
```
(tmos)# create net self 10.129.4.60/14 allow-service all vlan openshift_vxlan
```
![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/cluster/diagram/2021-07-06_13-49-15.png)

On bigip-02 create the self IP from hostsubnets **f5-server-02**
```
(tmos)# create net self 10.130.4.61/14 allow-service all vlan openshift_vxlan
```
![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/cluster/diagram/2021-07-06_13-50-21.png)

On the active BIG-IP, create a floating IP address in the subnet assigned by the OpenShift SDN from from hostsubnets **f5-server-float**
```
(tmos)# create net self 10.131.4.62/14 allow-service default traffic-group traffic-group-1 vlan openshift_vxlan
```
![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/cluster/diagram/2021-07-06_14-12-10.png)

### Step 4: Create a new partition on your BIG-IP system

    (tmos)# create auth partition OpenShift

This needs to match the partition in the controller configuration created by the CIS Operator

## Installing the F5 Container Ingress Services Operator in OpenShift

In OpenShift, CIS can be installed manually using a a yaml deployment manifest or using the Operator in OpenShift. The CIS Operator is a packaged deployment of CIS and will use Helm Charts to create the deployment. This user-guide provide additional information and examples when using the CIS Operator in OpenShift

### Step 5:

Create BIG-IP login credentials for use with Operator Helm charts

    oc create secret generic bigip-login  -n kube-system --from-literal=username=admin  --from-literal=password=<secret>

### Step 6:

Locate the F5 Container Ingress Services Operator in OpenShift OperatorHub as shown in the diagram below. Recommend search for F5 

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-10_12-59-30.png)

Select the Operator to Install. In this example I am installing the latest Operator 1.7.0. Select the Install tab as shown in the diagram

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-10_13-20-27.png)

### Step 7:

Install the Operator and provide the installation mode, installed namespaces and approval strategy. In this user-guide and demo I am using the defaults

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-10_13-47-45.png)

Operator will take a few minutes to install

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-10_13-50-10.png)

Once installed select the View Operator tab

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-10_13-51-02.png)

Now that the operator is installed you can create an instance of CIS. This will deploy CIS in OpenShift

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-14_14-07-36.png)

### Step 8: Create instance for f5-server-01

Note that currently some fields may not be represented in form so its best to use the "YAML View" for full control of object creation. Select the "YAML View"

![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/cluster/diagram/2021-07-06_14-52-52.png)

Enter requirement objects in the YAML View. Please add the recommended setting below:

* Change the f5-server name to **f5-server-01**
* Remove **agent as3** as this is default
* Change repo image to **f5networks/cntr-ingress-svcs**. By default OpenShift will pull the image from Docker. 
* Change the user to **registry.connect.redhat.com** so OpenShift will be pull the published image from the RedHat Ecosystem Catalog [repo](https://catalog.redhat.com/software/containers/f5networks/cntr-ingress-svcs/5ec7ad05ecb5246c0903f4cf)

Instance f5-server-01

```
apiVersion: cis.f5.com/v1
kind: F5BigIpCtlr
metadata:
  name: f5-server-01
  namespace: openshift-operators
spec:
  args:
    log_as3_response: true
    manage_routes: true
    log_level: DEBUG
    route_vserver_addr: 10.192.125.65
    bigip_partition: OpenShift
    openshift_sdn_name: /Common/openshift_vxlan
    bigip_url: 10.192.125.60
    insecure: true
    pool-member-type: cluster
  bigip_login_secret: bigip-login
  image:
    pullPolicy: Always
    repo: f5networks/cntr-ingress-svcs
    user: registry.connect.redhat.com
  namespace: kube-system
  rbac:
    create: true
  resources: {}
  serviceAccount:
    create: true
  version: latest
```

Instance f5-server-02

```
apiVersion: cis.f5.com/v1
kind: F5BigIpCtlr
metadata:
  name: f5-server-02
  namespace: openshift-operators
spec:
  args:
    log_as3_response: true
    manage_routes: true
    log_level: DEBUG
    route_vserver_addr: 10.192.125.65
    bigip_partition: OpenShift
    openshift_sdn_name: /Common/openshift_vxlan
    bigip_url: 10.192.125.61
    insecure: true
    pool-member-type: cluster
  bigip_login_secret: bigip-login
  image:
    pullPolicy: Always
    repo: f5networks/cntr-ingress-svcs
    user: registry.connect.redhat.com
  namespace: kube-system
  rbac:
    create: true
  resources: {}
  serviceAccount:
    create: true
  version: latest
```

Select the Create tab

### Step 9: Validate CIS deployment. Select Workloads/Deployments 

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-14_14-42-54.png)

Select the **f5-bigip-ctlr-operator** to see more details on the CIS deployment. Also validate the CIS deployment image

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-14_14-45-08.png)

### Step 10: Installing the Demo App in OpenShift

Deploy demo app in OpenShift. This could be done using the OpenShift UI or CLI. In this guide i use the CLI. Demo app repo available below 

```
# oc create -f demo-app/
deployment.apps/f5-demo created
service/f5-demo created
```
demo-app [repo](https://github.com/mdditt2000/openshift-4-7/tree/master/standalone/demo-app)

You can validate the demo app install via the OpenShift UI

![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone/diagram/2021-06-30_11-39-52.png)

## Create Route for Ingress traffic to Demo App

### Step 11:

Create basic route for Ingress traffic from BIG-IP to Demo App 

```
# oc create -f f5-demo-route-basic.yaml
route.route.openshift.io/f5-demo-route-basic created
```

f5-demo-route-basic [repo](https://github.com/mdditt2000/openshift-4-7/tree/master/standalone/route)

Validate the route via the OpenShift UI under the Networking/Routes

![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone/diagram/2021-06-30_13-59-43.png)

Validate the route via the BIG-IP

![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone/diagram/2021-06-30_14-00-53.png)