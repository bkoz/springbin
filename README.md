# Deploy a spring-boot jar to OpenShift 
## Overview
This procedure uses the OpenShift's source to image workflow and the redhat-openjdk18-openshift builder image.

If you don't have the openjdk18 builder image, create an ```image-stream``` object and import it into your project.

```
cat<<EOF>openjdk-s2i-imagestream.json
{
    "kind": "ImageStream",
    "apiVersion": "v1",
    "metadata": {
        "name": "redhat-openjdk18-openshift"
    },
    "spec": {
        "dockerImageRepository": "registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift",
        "tags": [
            {
                "name": "1.0",
                "annotations": {
                    "description": "OpenJDK S2I images.",
                    "iconClass": "icon-jboss",
                    "tags": "builder,java,xpaas",
                    "supports":"java:8,xpaas:1.0",
                    "sampleRepo": "https://github.com/jboss-openshift/openshift-quickstarts",
                    "sampleContextDir": "undertow-servlet",
                    "version": "1.0"
                }
            }
        ]
    }
}
EOF
```

Create the image-stream.

```
oc create -f openjdk-s2i-imagestream.json
```

If you need a SpringBoot jar, clone, compile and package the following example.
```
git clone https://github.com/redhat-helloworld-msa/ola.git
cd ola
mvn compile
mvn package
```

Create a build config with a binary build strategy and push the jar. 
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