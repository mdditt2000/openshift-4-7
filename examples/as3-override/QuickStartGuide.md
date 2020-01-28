# AS3 Override Example

**Note** CIS controller will need to be restart after applying update to AS3 override configMap. Enhancement to resolve this restart is planned

AS3 Override is a refer of existing configuration on BIGIP. CIS is attaching these profiles/polices to the virtual

## Dos profile with logging

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
              },
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

## Waf profile with logging

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
              },
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


## Certificate Override In BIG-IP Example

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: example-cert
  namespace: kube-system
data:
  template: |
    {
        "declaration": {
            "test_AS3": {
                "Shared": {
                    "osr_default_bar_com_cssl": {
                   "certificate": "-----BEGIN CERTIFICATE-----
                                    -----END CERTIFICATE-----",
                 "privateKey" :"-----BEGIN RSA PRIVATE KEY-----
                                -----END RSA PRIVATE KEY-----"
                   }
                }
            }
        }
    }
```



## Override Virtual Server

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: example-vs
  namespace: kube-system
data:
  template: |
    {
        "declaration": {
            "test_AS3": {
                "Shared": {
                    "ose_vserver": {
                        "virtualAddresses": [
                            "172.16.3.7"
                        ]
                    }
                }
            }
        }
    }
```


## Override Pool Members

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: example-pool
  namespace: kube-system
data:
  template: |
    {
        "declaration": {
            "test_AS3": {
                "Shared": {
                    "openshift_default_svc_dynamic_ratio_member": {
                        "members": [
                         {
                        "addressDiscovery": "static",
                        "serverAddresses": [
                            "172.16.1.8"
                        ],
                        "servicePort": 32165
                    },
                    {
                        "addressDiscovery": "static",
                        "serverAddresses": [
                            "172.16.1.13"
                        ],
                        "servicePort": 32165
                    },
                    {
                        "addressDiscovery": "static",
                        "serverAddresses": [
                            "172.16.1.17"
                        ],
                        "servicePort": 32165
                    }
                         ]
                    }
                }
            }
        }
    }
```