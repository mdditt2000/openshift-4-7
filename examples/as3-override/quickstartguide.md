# AS3 Override Example

# Bot-Defense profiles

bot-defense requires AS3-17. Please download the AS3 version here: https://github.com/mdditt2000/openshift-3-11/blob/master/yosef/f5-appsvcs-3.17.0-2.noarch.rpm
 
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
