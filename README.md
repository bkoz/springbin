# Deploy a Spring Boot binary (jar) to OpenShift 

### Using the odo client

Grab ```odo``` from https://mirror.openshift.com/pub/openshift-v4/clients/odo/

```
odo project create myproj
odo create java backend --binary targets/ola.jar
odo push
odo log
odo url create --port 8080
```

Cleanup from ```odo```

Delete from server.
```
odo delete backend
```
Delete from server and local.
```
odo delete --app backend --all
``` 
Delete the project.
```
odo delete project myproj
```
### Using the oc client
This procedure uses OpenShift's source to image workflow and the [openjdk18](https://access.redhat.com/containers/?tab=images&platform=openshift#/registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift) builder image to push a local jar file into the OpenShift build configuration.

If you don't have the openjdk18 builder image, create the ```image-stream``` object I obtained from Thomas's [blog post](https://developers.redhat.com/blog/2017/02/23/getting-started-with-openshift-java-s2i/) and import it into your project.

```
oc create -f openjdk-s2i-imagestream.json
```

If you need an example Spring Boot jar, clone, compile and package the following example.
```
git clone https://github.com/redhat-helloworld-msa/ola.git
cd ola
mvn clean package
```

Now create a build config with a binary build strategy and push the jar. If you want to push your own jar, make sure it is located in the ```target``` directory.
```
oc create -f openjdk-s2i-imagestream.json
oc new-build --binary=true --name=ola --image-stream=redhat-openjdk18-openshift
```
Next, create a deployment config, a service and a route.
```
oc start-build ola --from-dir=target --follow
oc new-app ola -l app=ola,hystrix.enabled=true
oc expose service ola
```

Wait for the pod to become ready then test it.

```
host=`oc get route ola --template={{.spec.host}}`
curl $host/api/health
```

Optionally, add a readiness probe.
```
oc env dc/ola AB_ENABLED=jolokia; oc patch dc/ola -p '{"spec":{"template":{"spec":{"containers":[{"name":"ola","ports":[{"containerPort": 8778,"name":"jolokia"}]}]}}}}'
oc set probe dc/ola --readiness --get-url=http://:8080/api/health
```
[Reference](https://github.com/redhat-helloworld-msa/helloworld-msa/blob/master/ola.adoc)

