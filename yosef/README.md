ReadMe for AS3-17.2 pre-build that provides support for referencing a BotPolicy

We have tested that Bot-Defense profiles get attached with virtual server with User Defined configmap as well as with AS3 overwrite functionality.
 
Please find the following examples for same
 
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
I added the bot-policy configuration to the as3 override configuration here f5-override-as3-declaration.yaml. Please download the AS3 version here https://github.com/mdditt2000/openshift-3-11/blob/master/yosef/f5-appsvcs-3.17.0-2.noarch.rpm
