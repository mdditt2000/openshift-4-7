# OpenShift 4.8 and F5 Container Ingress Services (CIS) User-Guide for Standalone BIG-IP using OVN-Kubernetes Advanced Networking

This user guide is create to document OpenShift 4.8 integration of CIS and standalone BIG-IP using OVN-Kubernetes advanced networking. This user guide provides configuration for a standalone BIG-IP with **OVN-Kubernetes hybrid overlay feature(VxLAN)**. OVN-Kubernetes hybrid overlay uses the GENEVE protocol for EAST/WEST traffic within the OpenShift Cluster and VxLAN tunnels to network BIG-IP devices.

![diagram]()

Demo on YouTube [video]()

RedHat documents the installation of **OVN-K8S advanced networking** in the [specifying advanced network configuration sections](https://docs.openshift.com/container-platform/4.8/installing/installing_vsphere/installing-vsphere-installer-provisioned-network-customizations.html#modifying-nwoperator-config-startup_installing-vsphere-installer-provisioned-network-customizations) of the install process. Based on the following note from RedHat, its very important to follow the installation of OVN-Kubernetes Hybrid Overlay Feature when installing OpenShift. Modification, migration cannot be applied once OpenShift is already installed.

![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone-ovn/diagram/2021-08-03_13-12-08.png)

### Prerequisites

You have created the **install-config.yaml** file with the required modifications. When creating the install-config.yaml, change the default networkType: **OpenShiftSDN** to networkType: **OVNKubernetes**

### Procedure

**Step 1:** Create install-config.yaml

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
install-config.yaml.yaml [repo](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone-ovn/openshift/install-config.yaml)

**Step 2:** Create manifests

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

**Step 3:** Copy cluster-network-03-config.yaml to manifests directory

RedHat documentation for [configuring hybrid networking with OVN-Kubernetes](https://docs.openshift.com/container-platform/4.8/networking/ovn_kubernetes_network_provider/configuring-hybrid-networking.html#configuring-hybrid-ovnkubernetes_configuring-hybrid-networking)

Create a stub manifest file for the advanced network configuration that is named cluster-network-03-config.yml in the <installation_directory>/manifests/ directory. The defaultNetwork: hybridOverlayConfig: {} is required

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
```
cluster-network-03-config.yaml [repo](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone-ovn/openshift/cluster-network-03-config.yaml)

**Step 4:** Create Cluster

You ready to create the OpenShift cluster

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

## Environment Parameters

* OpenShift 4.8 on vSphere with Installer-Provisioned Infrastructure (IPI)
* CIS 2.5
* AS3: 3.28
* BIG-IP 16.0.1.1

Since CIS is using the AS3 declarative API we need the AS3 extension installed on BIG-IP. Follow the link to install AS3
 
* Install AS3 on BIG-IP
https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

## Create a BIG-IP VXLAN tunnel for OVN-Kubernetes Advanced Networking

### Procedure

**Step 1:** Create tunnel profile

    (tmos)# create net tunnels vxlan vxlan-mp flooding-type multipoint

![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone-ovn/diagram/2021-08-03_14-18-36.png)

    (tmos)# create net tunnels tunnel openshift_vxlan key 4097 profile vxlan-mp local-address 10.192.125.60

**Note:** OpenShift uses 4097(VNI) for VxLAN communication

![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone-ovn/diagram/2021-08-03_14-20-34.png)

### Step 2: Create self-ip for CNI

    (tmos)# create net self 10.142.2.60/12 allow-service all vlan openshift_vxlan

**Note:** Use self IP range (10.142.2.60/12) which supernets the OpenShift cluster network i.e 10.128.0.0/14 to differentiate the VxLAN and GENEVE communication.

![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone-ovn/diagram/2021-08-03_14-39-20.png)

Diagram of all the BIG-IP self-ip addresses

![diagram](https://github.com/mdditt2000/openshift-4-7/blob/master/standalone-ovn/diagram/2021-08-03_14-40-06.png)

## Create a partition on BIG-IP for CIS to manage

### Step 4:

    (tmos)# create auth partition OpenShift

This needs to match the partition in the controller configuration created by the CIS Operator

## Create CIS Controller, BIG-IP credentials and RBAC Authentication

### Procedure

Create f5-bigip-deployment manifests

```
# oc create secret generic bigip-login --namespace kube-system --from-literal=username=admin --from-literal=password=<secret>
# oc create serviceaccount bigip-ctlr -n kube-system
# oc create -f f5-openshift-clusterrole.yaml
# oc create -f f5-bigip-deployment.yaml
# oc adm policy add-cluster-role-to-user cluster-admin -z bigip-ctlr -n kube-system
```

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