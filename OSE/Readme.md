### Implementation of this NodeJs and Redis Apps on Openshift

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

Through Openshift UI console - go to the poc project and create a redis-cache application and a redis-database application.


For the redis-database you need to create a persistent volume. 
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

Using ConfigMap to externalize the redis database connection. 
more connect.properties
service=redis-cache

oc create configmap config --from-file=connect.properties
oc set volumes dc/hof-example-form --add -m /etc/node-app/ --configmap-name=config

inside DC
oc get dc/hof-example-form  -o json

YAML 
volumes:
   - configMap:
       defaultMode: 420
       items:
       - key: connect.properties
         path: node-app.config
       name: config
     name: volume-kqkow

JSON
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




oc export all --as-template=poc >poc.yaml
