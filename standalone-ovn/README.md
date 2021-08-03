# OpenShift 4.8 and F5 Container Ingress Services (CIS) User-Guide for Standalone BIG-IP using OVN-Kubernetes Advanced Networking

This user guide is create to document OpenShift 4.8 integration of CIS and standalone BIG-IP using OVN-Kubernetes advanced networking. This user guide provides configuration for a standalone BIG-IP with **OVN-Kubernetes hybrid overlay feature(VxLAN)**. OVN-Kubernetes hybrid overlay uses the GENEVE protocol for EAST/WEST traffic within the OpenShift Cluster and VxLAN tunnels to network BIG-IP devices. 

RedHat documents the installation of **OVN-K8S advanced networking** in the [specifying advanced network configuration sections](https://docs.openshift.com/container-platform/4.8/installing/installing_vsphere/installing-vsphere-installer-provisioned-network-customizations.html#modifying-nwoperator-config-startup_installing-vsphere-installer-provisioned-network-customizations) of the install process. Based on the following note from RedHat, its very important to follow the installation of OVN-Kubernetes Hybrid Overlay Feature when installing OpenShift. Modification, migration cannot be applied once OpenShift is already installed.

![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone-ovn/diagram/2021-08-03_13-12-08.png)

### Prerequisites

You have created the **install-config.yaml** file with the required modifications. When creating the install-config.yaml, modify the default networkType: **OpenShiftSDN** to networkType: **OVNKubernetes**

install-config.yaml.yaml [repo](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone-ovn/openshift/install-config.yaml)

### Procedure

* **Step 1:** Create install-config

```
# ./openshift-install create install-config --dir=ipi
? Platform vsphere
? vCenter vcsa7-pme.f5demo.com
? Username administrator@f5demo.com
? Password [? for help] *********
INFO Connecting to vCenter vcsa7-pme.f5demo.com
? Datacenter PME-LAB
? Cluster OCP-PM
? Default Datastore datastore1 (3)
? Network VM Network
? Virtual IP Address for API 10.192.125.101
? Virtual IP Address for Ingress 10.192.125.102
? Base Domain f5demo.com
? Cluster Name ocp-pm
? Pull Secret [? for help] ......
INFO Install-Config created in: ipi
```

* Step 2: Create manifests

```
# ./openshift-install create manifests --dir=ipi
INFO Consuming Install Config from target directory
INFO Manifests created in: ipi/manifests and ipi/openshift

# ls
04-openshift-machine-config-operator.yaml  cluster-infrastructure-02-config.yml  cluster-proxy-01-config.yaml     kube-system-configmap-root-ca.yaml
cloud-provider-config.yaml                 cluster-ingress-02-config.yml         cluster-scheduler-02-config.yml  machine-config-server-tls-secret.yaml
cluster-config.yaml                        cluster-network-01-crd.yml            cvo-overrides.yaml               openshift-config-secret-pull-secret.yaml
cluster-dns-02-config.yml                  cluster-network-02-config.yml         kube-cloud-config.yaml           openshift-kubevirt-infra-namespace.yaml
```

* Step 4: Copy cluster-network-03-config.yaml to manifests directory

```
# cat cluster-network-03-config.yaml
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  name: cluster
spec:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16
  defaultNetwork:
    ovnKubernetesConfig:
      hybridOverlayConfig: {}
    type: OVNKubernetes

# cp cluster-network-03-config.yaml /openshift/ipi/manifests/
# cd /openshift/ipi/manifests/
# ls
04-openshift-machine-config-operator.yaml  cluster-infrastructure-02-config.yml  cluster-network-03-config.yaml   kube-cloud-config.yaml                    openshift-kubevirt-infra-namespace.yaml
cloud-provider-config.yaml                 cluster-ingress-02-config.yml         cluster-proxy-01-config.yaml     kube-system-configmap-root-ca.yaml
cluster-config.yaml                        cluster-network-01-crd.yml            cluster-scheduler-02-config.yml  machine-config-server-tls-secret.yaml
cluster-dns-02-config.yml                  cluster-network-02-config.yml         cvo-overrides.yaml               openshift-config-secret-pull-secret.yaml
```

* Step 4: Create Cluster

```
# ./openshift-install create cluster --dir=ipi
INFO Consuming Worker Machines from target directory
INFO Consuming OpenShift Install (Manifests) from target directory
INFO Consuming Openshift Manifests from target directory
INFO Consuming Master Machines from target directory
INFO Consuming Common Manifests from target directory
INFO Creating infrastructure resources...
INFO Waiting up to 20m0s for the Kubernetes API at https://api.ocp-pm.f5demo.com:6443...
INFO API v1.21.1+8268f88 up
INFO Waiting up to 30m0s for bootstrapping to complete...
INFO Destroying the bootstrap resources...
INFO Waiting up to 40m0s for the cluster at https://api.ocp-pm.f5demo.com:6443 to initialize...
INFO Waiting up to 10m0s for the openshift-console route to be created...
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/openshift/ipi/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ocp-pm.f5demo.com
INFO Login to the console with user: "kubeadmin", and password: "secret"
INFO Time elapsed: 26m50s
#
```

![diagram]()

Demo on YouTube [video]()

## Environment parameters

* OpenShift 4.7 on vSphere with installer-provisioned infrastructure
* CIS 2.5
* AS3: 3.28
* BIG-IP 16.0.1.1

## Prerequisite before creating the OCP 4.7 Cluster

* Modify the **networkType** to **OVNKubernetes** in the install-config.yaml [manifests](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone-ovn/openshift/install-config.yaml)
* Create a stub manifest file cluster-network-03-config.yml for the advanced network configuration with **hybridOverlayConfig** objects [manifests](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone-ovn/openshift/cluster-network-03-config.yaml)

More information on configuring hybrid networking, review the OpenShift documentation [docs](https://docs.openshift.com/container-platform/4.7/networking/ovn_kubernetes_network_provider/configuring-hybrid-networking.html)

Since CIS is using the AS3 declarative API we need the AS3 extension installed on BIG-IP. Follow the link to install AS3
 
* Install AS3 on BIG-IP
https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

## Create a BIG-IP VXLAN tunnel for OpenShift SDN

### Step 1:

```
(tmos)# create net tunnels vxlan vxlan-mp flooding-type multipoint
(tmos)# create net tunnels tunnel openshift_vxlan key 0 profile vxlan-mp local-address 10.192.125.60
```

![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone/diagram/2021-06-29_15-42-04.png)

### Step 2:

Create a host subnet for the BIP-IP. This will provide the subnet for creating the tunnel self-IP

    oc create -f f5-openshift-hostsubnet.yaml

```
# oc get hostsubnet
NAME                        HOST                        HOST IP          SUBNET          EGRESS CIDRS   EGRESS IPS
f5-server                   f5-server                   10.192.125.60   10.128.4.0/23
ocp-pm-bwmmz-master-0       ocp-pm-bwmmz-master-0       10.192.75.229   10.130.0.0/23
ocp-pm-bwmmz-master-1       ocp-pm-bwmmz-master-1       10.192.75.231   10.129.0.0/23
ocp-pm-bwmmz-master-2       ocp-pm-bwmmz-master-2       10.192.75.230   10.128.0.0/23
ocp-pm-bwmmz-worker-9ch4b   ocp-pm-bwmmz-worker-9ch4b   10.192.75.234   10.129.2.0/23
ocp-pm-bwmmz-worker-lws6s   ocp-pm-bwmmz-worker-lws6s   10.192.75.235   10.131.0.0/23
ocp-pm-bwmmz-worker-qdhgx   ocp-pm-bwmmz-worker-qdhgx   10.192.75.233   10.128.2.0/23

```
f5-openshift-hostsubnet.yaml [repo](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone/cis/f5-openshift-hostsubnet.yaml)

### Step 3:

    (tmos)# create net self 10.128.4.60/14 allow-service all vlan openshift_vxlan

Subnet from the **f5-server** hostsubnet create above. Used .60 to be consistent with Big-IP internal interface

![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone/diagram/2021-06-30_09-42-15.png)

## Create a new partition on your BIG-IP system

### Step 4:

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

### Step 8:

Note that currently some fields may not be represented in form so its best to use the "YAML View" for full control of object creation. Select the "YAML View"

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-14_14-14-41.png)

Enter requirement objects in the YAML View. Please add the recommended setting below:

* Remove **agent as3** as this is default
* Change repo image to **f5networks/cntr-ingress-svcs**. By default OpenShift will pull the image from Docker. 
* Change the user to **registry.connect.redhat.com** so OpenShift will be pull the published image from the RedHat Ecosystem Catalog [repo](https://catalog.redhat.com/software/containers/f5networks/cntr-ingress-svcs/5ec7ad05ecb5246c0903f4cf)

```
apiVersion: cis.f5.com/v1
kind: F5BigIpCtlr
metadata:
  name: f5-server
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

Select the Create tab

### Step 9:

Validate CIS deployment. Select Workloads/Deployments 

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-14_14-42-54.png)

Select the **f5-bigip-ctlr-operator** to see more details on the CIS deployment. Also validate the CIS deployment image

![diagram](https://github.com/mdditt2000/k8s-bigip-ctlr/blob/main/user_guides/operator/diagrams/2021-06-14_14-45-08.png)

## Installing the Demo App in OpenShift

### Step 10:

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