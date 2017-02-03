## Implementation of this NodeJs and Redis Apps on Openshift

#### Thanks to [Petro Algaba](https://github.com/antipodas)

Create the Proof of Concept project

```
oc new-project poc
```

Import the image redis database into Openshift

```
oc create -f https://raw.githubusercontent.com/getupcloud/origin-templates/master/imagestreams/redis.yaml -n poc
```

Import the Redis-cache (ephemeral) template into Openshift

```
oc create -f https://raw.githubusercontent.com/antipodas/origin-templates/master/templates/redis-cache-template.yaml
```

Import the Redis-database (persistant) template into Openshift
```
oc create -f https://raw.githubusercontent.com/antipodas/origin-templates/master/templates/redis-database-template.yaml
```


Create the application into the poc project. We are using a image called antipodas/mynode to build the nodejs apps. The [Dockerfile](https://raw.githubusercontent.com/antipodas/mynode/master/Dockerfile) of the nodejs builder image is available inside this github [project](https://github.com/antipodas/mynode)
 

```
oc new-app antipodas/mynode~https://github.com/dwojciec/hof-example-form.git -n poc
```

Create a route to access the application

```
oc expose svc hof-example-form
```


Through Openshift UI console - go to the poc project and create a redis-cache application and/or a redis-database application.

![redis-cache](https://raw.githubusercontent.com/dwojciec/hof-example-form/master/OSE/images/redis-cache.png)

and/or redis-database

![redis-database](https://raw.githubusercontent.com/dwojciec/hof-example-form/master/OSE/images/redis-database.png)

For the **redis-database** you need to create a persistent volume (PV).
 
On my local 'cluster up' environment (I'm using the fantastic [cluster up wrapper](https://github.com/openshift-evangelists/oc-cluster-wrapper) created by Jorge Morales to facilitate this task) I created it [like](https://github.com/openshift-evangelists/oc-cluster-wrapper#oc-cluster-create-volume) :

```
oc-cluster create-volume redis-database 5Gi
``` 

To be able to use correctly the Redis database we are creating a service Account and we will give Security Context privileges

```
oc create serviceaccount poc
oadm policy add-scc-to-user anyuid system:serviceaccount:poc:poc
oadm policy add-scc-to-group anyuid system:authenticated
```
On a single Openshift Installation replace oadm by

```
oc adm policy add-scc-to-user anyuid system:serviceaccount:poc:poc
oc adm policy add-scc-to-group anyuid system:authenticated

```

Using ConfigMap to externalize the redis database connection. The external file is called connect.properties. its content is :


```
$ cat connect.properties
service=redis-cache
```

Create the ConfigMap base on the connect.properties file

```
oc create configmap config --from-file=connect.properties
```

List and verify that the **ConfigMap** is created successfully containing connection to redis service. 

```
$ oc get configmap
NAME      DATA      AGE
config    1         22h
$ oc describe configmap config
Name:		config
Namespace:	poc
Labels:		<none>
Annotations:	<none>

Data
====
connect.properties:	23 bytes

$ oc get configmap config -o yaml
apiVersion: v1
data:
  connect.properties: |
    service=redis-database
kind: ConfigMap
metadata:
  creationTimestamp: 2017-02-02T12:02:47Z
  name: config
  namespace: poc
  resourceVersion: "679110"
  selfLink: /api/v1/namespaces/poc/configmaps/config
  uid: 82fe39e1-e93f-11e6-bfa6-3e516d87f083
```



The below command makes the content of the config **ConfigMap**, which you created from a file, called connect.properties, available in the /etc/node-app directory. The hof-example-form **DeploymentConfiguration** detects the configuration change, and automatically deploys the Pod with the new configuration. 


```
oc set volumes dc/hof-example-form --add -m /etc/node-app/ --configmap-name=config
```

inside the **DeploymentConfiguration**  we can see something like 

```
$ oc get dc/hof-example-form  -o json

......
"spec": {
                "volumes": [
                    {
                        "name": "volume-kqkow",
                        "configMap": {
                            "name": "config",
                            "items": [
                                {
                                    "key": "connect.properties",
                                    "path": "node-app.config"
                                }
                            ],
                            "defaultMode": 420
                        }
                    }
                ],
                ......
                  "volumeMounts": [
                            {
                                "name": "volume-kqkow",
                                "mountPath": "/etc/node-app"
                            }
                        ],
                
```

To consume the value inside the ConfigMap definition we have to update the **DeploymentConfig**
 
from 

```
 {
                        "name": "volume-kqkow",
                        "configMap": {
                            "name": "config",
                            "defaultMode": 420
                        }
                    }
```
to

```
 {
                        "name": "volume-kqkow",
                        "configMap": {
                            "name": "config",
                            "items": [
                                {
                                    "key": "connect.properties",
                                    "path": "node-app.config"
                                }
                            ],
                            "defaultMode": 420
                        }
                    }
```

The ConfigMap contents can be managed from UI 

![redis-database](https://raw.githubusercontent.com/dwojciec/hof-example-form/master/OSE/images/configmap.png)

We can edit it directly from UI

![redis-database](https://raw.githubusercontent.com/dwojciec/hof-example-form/master/OSE/images/configmap-edit.png)

The full application inside Openshift is
 

![redis-database](https://raw.githubusercontent.com/dwojciec/hof-example-form/master/OSE/images/beis-apps.png)

By having a ConfigMap we can switch the service from redis-cache (ephemeral) or redis-database (persistent). We are offering more flexibility. see here what we changed inside the original code to offer this flexibility.


To access the application we are using the route created. 
In my example
 
http://hof-example-form-poc.127.0.0.1.xip.io/my-awesome-form 


![redis-database](https://raw.githubusercontent.com/dwojciec/hof-example-form/master/OSE/images/apps-running.png)


I'm exporting all components into a template

```
oc export all --as-template=poc >poc.yaml
or in JSON format
oc export all --as-template=poc -o json >poc.json
```


### To Do

1. script to create the PV needed for redis database
2. based on the exported file I have to update value with [parameters](https://docs.openshift.org/latest/dev_guide/templates.html#writing-parameters) to be able to redeploy the full apps based on the template.
