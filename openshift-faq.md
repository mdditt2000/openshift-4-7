# OpenShift FAQ for Container Ingress Controller

## OpenShift node install with multiple interfaces

**Problem:** CIS takes the node ip information from kube api in cluster mode. It should actually take these details from flannel as CIS creates the fdb entries in BIG-IP using these details. This issue is only seen when the nodes are multiple interfaces. 

**Solution:** Use Flannel annotations to get the node ip addresses. Add a new annotation for mac address and public IP. This needs to get added to each node. Please find the required annotations below. MAC address of the vxlan interface of that particular node.

```
    flannel.alpha.coreos.com/backend-data: '{"VtepMAC":"<MAC>"}'
    flannel.alpha.coreos.com/public-ip: “<IP>”
```
In OpenShift edit the nodes yaml

```
oc get nodes -o yaml
and edit the yaml to add the annotations and apply
```
Github issue https://github.com/F5Networks/k8s-bigip-ctlr/issues/797

---

## Memory recommendation using CIS and AS3

**Solution:** restjavad recommendations and best practices when using CIS with AS3. The re-provisioning of memory will trigger a restart of the services

Increase memory assigned to the Linux host:
```
tmsh modify sys db provision.extramb value 1000
```
Allow restjavad to access the extra memory:
```
tmsh modify sys db restjavad.useextramb value true
```
Save the config:
```
tmsh save sys config
```
Wait until the unit is online again. Increase the restjavad maxMessageBodySize property using the following curl command:
```
[curl -s -f -u admin: -H "Content-Type: application/json" -d '{"maxMessageBodySize":134217728}' -X POST http://localhost:8100/mgmt/shared/server/messaging/settings/8100
```
.. image:: blob/master/images/resource.png
:width: 400

---