# Migration and Upgrade Guide

This page is created to provide guidance on how to migrate and upgrade the following:

* migration from iApp ConfigMap using CCCL to AS3 ConfigMap
* upgrading from CIS 1.x to CIS 2.x using CCCL to AS3
* removing of _AS3 partition
* CIS and AS3 support schema versions

# Migration from iApp ConfigMap using CCCL to AS3 ConfigMap

When migrating from iApp ConfigMap using CCCL to AS3 ConfigMap there are some differences and gotchas. In CIS, configuration of ConfigMap are agent-specific, meaning that configuration in ConfigMap differs from agent to agent. CIS v2.0 the default agent changed from CCCL to AS3. Therefore when migrating from iApp ConfigMap to AS3 ConfigMap using CIS 2.x you do not need to specify the agent as a CIS argument. However if upgrade from CIS 1.x to CIS 2.x and wanting to continue using iApp ConfigMap, specify the CCCL agent type as –agent=CCCL.

Application Services 3 Extension (referred to as AS3 Extension or more often simply AS3) is a flexible, low-overhead mechanism for managing application-specific configurations on a BIG-IP system. The AS3 Extension is required to be installed on BIG-IP. Follow the following document to install AS3 Extension on BIG-IP:

## Install AS3 on BIGIP

https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

Another difference between iApp ConfigMap using CCCL to AS3 ConfigMap is the concept of tenants. iApp ConfigMap using CCCL the BIG-IP objects are created in BIG-IP using the CIS base partition. When using a AS3 ConfigMap the objects are create in the tenants defined in the JSON declaration. AS3 will create the tenant if the tenant does not exist on BIG-IP. In the example below from the AS3 ConfigMap the tenant is defined as k8s_app. The CIS base tenant cannot be used for the AS3 ConfigMap as in the iApp ConfigMap using CCCL.

Also since AS3 is declarative and declared to BIG-IP by CIS as a single JSON, all the applications need to be added to JSON declaration. CIS unlike Route, Ingress and CRDs wont maintain the state of the AS3 json from the AS3 ConfigMap. In the example below, one application myService_f5_http has been defined. When defining the second application it would can defined in the same tenant as myService_f5_https for example. It is best practice to define your AS3 ConfigMap tenant name the same as the K8S namespace.

Example of ConfigMap with iApp

    kind: ConfigMap
    apiVersion: v1
    metadata:
    name: k8s.http
    namespace: default
    labels:
        f5type: virtual-server
    data:
    # See the f5-schema table for schema-controller compatibility
    # https://clouddocs.f5.com/containers/latest/releases_and_versioning.html#f5-schema
    schema: "f5schemadb://bigip-virtual-server_v0.1.7.json"
    data: |
        {
        "virtualServer": {
            "backend": {
            "serviceName": "f5-hello-world",
            "servicePort": 8080
            },
            "frontend": {
            "partition": "k8s",
            "iapp": "/k8s/f5.http_custom",
            "iappPoolMemberTable": {
                "name": "pool__members",
                "columns": [
                    {"name": "addr", "kind": "IPAddress"},
                    {"name": "port", "kind": "Port"},
                    {"name": "connection_limit", "value": "0"}
                ]
            },
            "iappOptions": {
                "description": "myService_f5.http iApp"
            },
            "iappVariables": {
                "monitor__monitor": "/#create_new#",
                "monitor__response": "none",
                "monitor__uri": "/v1/welcome",
                "net__client_mode": "lan",
                "net__server_mode": "lan",
                "pool__addr": "172.17.32.172",
                "pool__pool_to_use": "/#create_new#",
                "pool__port": "80",
                "pool__hosts": "1.1.1.1"
            }
            }
        }
        }

Example of AS3 ConfigMap with AS3 for tenant k8s_app

    kind: ConfigMap
    apiVersion: v1
    metadata:
    name: k8s.http
    namespace: default
    labels:
        f5type: virtual-server
        as3: "true"
    data:
    template: |
        {
            "class": "AS3",
            "declaration": {
                "class": "ADC",
                "schemaVersion": "3.20.0",
                "id": "urn:uuid:33045210-3ab8-4636-9b2a-c98d22ab915d",
                "label": "http",
                "remark": "Simple HTTP application",
                "k8s_app": {
                    "class": "Tenant",
                    "myService_f5_http": {
                        "class": "Application",
                        "template": "generic",
                        "myService_f5_http": {
                            "class": "Service_HTTP",
                            "virtualAddresses": [
                                "10.192.75.111"
                            ],
                            "virtualPort": 8080,
                            "pool": "pool__members"
                        },
                        "pool__members": {
                            "class": "Pool",
                            "monitors": [
                                {
                                    "use": "http_monitor"
                                }
                            ],
                            "members": [
                                {
                                    "servicePort": 80,
                                    "serverAddresses": []
                                }
                            ]
                        },
                        "http_monitor": {
                            "class": "Monitor",
                            "monitorType": "http",
                            "send": "HTTP GET /v1/welcome",
                            "receive": "none",
                            "adaptive": false
                        }
                    }
                }
            }
        }

# Upgrading from CIS 1.x to CIS 2.x using CCCL to AS3

CIS has two API's to communicate with BIG=IP

* CCCL legacy python rest based API
* AS3 uses the F5 Automation tool chain framework

In CIS v2.x the default API agent was changed from CCCL to AS3. This guide is created to provide some guidance when migrating from CIS 1.x to CIS 2.x using CCCL to AS3

## Step 1

Upgrade CIS 1.x to CIS 2.0

When migrating from CIS 1.x to CIS 2.x using CCCL to AS3 is very important to upgrade to CIS 2.0 first with CCCL agent. Very important to add the CCCL agent to the CIS argument before creating the CIS deployment

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
  "--agent=cccl",
]
```

## Step 2

Convert CIS from agent CCCL to AS3. If BIG-IP has a self-signed certificate for management you need to add insecure=true or schema validation will fail. You also need to install the AS3 Extension. For CIS 2.0 install AS3-18. Follow the following document to install AS3 Extension on BIG-IP:

### Install AS3 on BIGIP

https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

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
  # Self-signed cert
  "--insecure=true",
]
```

CIS will create a tenant on the BIG-IP names base-tenant_AS3. This will be removed in Step 3 

## Step 3

Upgrade to CIS 2.1.x and remove the _AS3 partition. No change in the CIS arguments is required. Just a CIS version and AS3 update. Use CIS 2.1.1 with AS3.21

# Removing of _AS3 partition

As mentioned above uses using CIS 2.x with Ingress or Routes with agent AS3, CIS 2.1 will use the CIS base partition.

# CIS and AS3 support schema versions

CIS 2.1 support AS3-20 and below. If using AS3-21 please update CIS with the following patch specified in Github issue https://github.com/F5Networks/k8s-bigip-ctlr/issues/1433