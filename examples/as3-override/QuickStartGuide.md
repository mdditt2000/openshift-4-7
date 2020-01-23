# AS3 Override Example

AS3 Override is a refer of existing configuration on BIGIP. CIS is attaching these profiles/polices to the virtual

## Dos profile

```
AS3 Override:
kind: ConfigMap
apiVersion: v1
metadata:
  name: override
  namespace: default
data:
  template: |
    {
        "declaration": {
            "test_AS3": {
                "Shared": {
                    "https_ose_vserver": {
                        "profileDOS": {
                             "bigip": "/Common/dos-defense"
                        }
                    }
                }
            }
        }
    }
```

## Waf profile

A WAF route annotation virtual-server.f5.com/waf: /Common/WAF_Policy is available or you can use the AS3 override feature. Example of route annotation: [route](https://github.com/mdditt2000/openshift-3-11/blob/master/examples/routes/f5-demo-app-route-waf.yaml)

```
AS3 Override:
kind: ConfigMap
apiVersion: v1
metadata:
  name: override
  namespace: default
data:
  template: |
    {
        "declaration": {
            "test_AS3": {
                "Shared": {
                    "https_ose_vserver": {
                        "policyWAF": {
                             "bigip": "/Common/WAF_Policy"
                        }
                    }
                }
            }
        }
    }
```

## Bot-Defense profile

**bot-defense** requires AS3-17. Please download the AS3 version here: https://github.com/mdditt2000/openshift-3-11/blob/master/yosef/f5-appsvcs-3.17.0-2.noarch.rpm
 
```
AS3 Override:
kind: ConfigMap
apiVersion: v1
metadata:
  name: override
  namespace: default
data:
  template: |
    {
        "declaration": {
            "test_AS3": {
                "Shared": {
                    "https_ose_vserver": {
                        "profileBotDefense": {
                             "bigip": "/Common/bot-defense"
                        }
                    }
                }
            }
        }
    }
```

**Note** CIS controller will need to be restart after applying update to AS3 override configMap. Enhancement to resolve this restart is planned
