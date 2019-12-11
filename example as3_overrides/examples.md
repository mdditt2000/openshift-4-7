# AS3 Override examples 

This example is used to log all request. Good for WAF
```
kind: ConfigMap
apiVersion: v1
metadata:
  name: f5-route-vs-override
  namespace: f5demo
data:
  template: |
    {
      "declaration": {
        "openshift_AS3": {
          "Shared": {
            "https_ose_vserver": {
              "securityLogProfiles": [
                {
                  "bigip": "/Common/Log all requests"
                }
              ]
            },
            "ose_vserver": {
              "securityLogProfiles": [
                {
                  "bigip": "/Common/Log all requests"
                }
              ]
            }
          }
        }
      }
    }
```
This example is used to log all request. Good for WAF
```
"snat": {
   "bigip": "/Common/SNATPOOL"
 }
 ```