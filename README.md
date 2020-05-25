# Unofficial AppD Java Agent Autoinjector

This is a sample of how to use an instance of the [k8s-auto-injector](https://github.com/tumblr/k8s-sidecar-injector/) with a configmap and annotations to automatically inject the AppD Java APM Agent into a Kubernetes pod via init container. Note that JAVA_OPTS or another similar way of setting parameters passed to the JVM is needed for this example.

## Getting Started

First, if it does not already exist, create the appdynamics namespace on Kubernetes:

```
kubectl create namespace appdynamics
```
Then, create the TLS certificates needed by the admission webhook:

```
kubectl apply -f appd-tls-gen.yaml
```

After the self-signed certificates have been created, the value for the CA public key must be set directly in the webhook configuration:

```
CABUNDLE_BASE64="$(kubectl get secret appd-tls --namespace appdynamics -o json | jq -r '.data["ca.crt"]')"
sed -i '' -e "s|__CA_BUNDLE_BASE64__|$CABUNDLE_BASE64|g" k8s-sidecar-injector/mutating-webhook-configuration.yaml
```

Now, you'll need to update the k8s-sidecar-injector/appd-java-agent-configmap.yaml file with appropriate values for your controller (and, if needed, adjust JAVA_OPTS to match the approach appropriate for your application container). 

You'll also need to base64-encode your AppDynamics controller key

```
echo -n '<your controller key>' | base64
```

and paste the value into the appd-key field in the appd-secret.yaml file. Then you can run

```
kubectl apply -f appd-secret.yaml --namespace <your application namespace>
```

Install the sidecar injector:

```
kubectl apply -f k8s-sidecar-injector
```

You should now be ready to modify the deployment for your application container(s). Add annotations similar to those below (and, if necessary, make adjustments to how you are passing JAVA_OPTS or similar). Note that you will want to set the tier-name and reuse-node-name-prefix values to something meaningful to your setup:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-sample-deployment
  labels:
    app: tomcat-sample
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat

      # injector annotations begin
      annotations:
        injector.tumblr.com/request: appd-java
        k8s.appdynamics.com/tier-name: <your tier name>
        k8s.appdynamics.com/reuse-node-name: "true"
        k8s.appdynamics.com/reuse-node-name-prefix: "<your node name prefix>"
      # injector annotations end

    spec:
      containers:
        - name: tomcat-sample
          image: <your app image here>
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          securityContext: {}
```

You should now be ready to deploy your application container and the AppDynamics agent will be auto-injected via init container.