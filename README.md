# Deploy a spring-boot binary (jar) to OpenShift 
## Overview
This procedure uses the OpenShift's source to image workflow and the redhat-openjdk18-openshift builder image.

If you don't have the openjdk18 builder image, create the ```image-stream``` object I obtained from Thomas's [blog post](https://developers.redhat.com/blog/2017/02/23/getting-started-with-openshift-java-s2i/) and import it into your project.

```
oc create -f openjdk-s2i-imagestream.json
```

If you need an example SpringBoot jar, clone, compile and package the following example.
```
git clone https://github.com/redhat-helloworld-msa/ola.git
cd ola
mvn compile
mvn package
```

Now create a build config with a binary build strategy and push the jar. If you use your own jar, make sure it is located in the ```target``` directory.
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

Wait for the pod to run and become ready then test it.

```
host=`oc get route ola --template={{.spec.host}}`
curl $host/api/health
```

Optionally, add a readiness probe.
```
oc env dc/ola AB_ENABLED=jolokia; oc patch dc/ola -p '{"spec":{"template":{"spec":{"containers":[{"name":"ola","ports":[{"containerPort": 8778,"name":"jolokia"}]}]}}}}'
oc set probe dc/ola --readiness --get-url=http://:8080/api/health
```