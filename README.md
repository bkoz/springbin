# Deploy a spring-boot jar to OpenShift 
## Overview
This procedure uses the OpenShift's source to image workflow and the redhat-openjdk18-openshift builder image.

If you don't have the builder image, import the image stream into your project.

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

```
oc create -f openjdk-s2i-imagestream.json
```

```
oc new-build --binary=true --name=spring --image-stream=redhat-openjdk18-openshift
oc env bc/spring -e JAVA_APP_DIR=/deployments -e JAVA_APP_JAR=ola.jar
oc start-build spring --from-dir=. --follow
oc start-build bc/spring --from-file=ola.jar --follow
oc new-app spring
```

Building the ola sample swagger springboot app

```
git clone https://github.com/redhat-helloworld-msa/ola.git
cd ola/
mvn compile
mvn package
cd target/
java -jar ola.jar
```

Try: 
``` oc start-build spring --from-file=ola.jar --follow```


Clone, compile and package the code.
```
git clone https://github.com/redhat-helloworld-msa/ola.git
cd ola
mvn compile
mvn package
```

Create image stream, build config, deployment config, service and route.
```
oc create -f openjdk-s2i-imagestream.json
oc new-build --binary=true --name=ola --image-stream=redhat-openjdk18-openshift
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