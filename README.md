# Minimal OPA-Envoy Sidecar Example for Kubernetes

This repository shows how to run OPA and Envoy as sidecar containers inside an
app deployment to enforce HTTP API access control policies. Envoy is statically
configured to proxy traffic for the app with the [External
Authorization](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/security/ext_authz_filter#arch-overview-ext-authz)
filter enabled to use OPA.

## Try It Out

> These steps have been tested on Kubernetes v1.15.

### 1. Bootstrap Kubernetes cluster (optional)

Bootstrap a new Kubernetes cluster using [kind](https://github.com/kubernetes-sigs/kind):

```bash
kind create cluster
```

Set the `KUBECONFIG` environment variable to point `kubectl` at the cluster:

```bash
export KUBECONFIG=~/.kube/kind-config-kind
```

### 2. Create ConfigMaps containing proxy config and app policy

Load an Envoy proxy config into the cluster that passes traffic for the app
container and enables External Authorization.

```bash
kubectl create configmap proxy-config --from-file config/envoy.yaml
```

Load an example policy into the cluster that will ALLOW ALL incoming traffic to
the app:

```bash
kubectl create configmap app-policy --from-file policies/allow_all.rego
```

> In typical deployments the policy would either be built into the OPA container
> image or it would fetched dynamically via the [Bundle
> API](https://www.openpolicyagent.org/docs/latest/bundles/). ConfigMaps are
> used in this example for test purposes.

### 3. Create Deployment with sidecar containers

```bash
kubectl apply -f example-app-deployment.yaml
```

**example-app-deployment.yaml**:

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: example-app
  labels:
    app: example-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: example-app
  template:
    metadata:
      labels:
        app: example-app
    spec:
      initContainers:
        - name: proxy-init
          image: openpolicyagent/proxy_init:v2
          args: ["-p", "8000", "-u", "1111"]
          securityContext:
            capabilities:
              add:
              - NET_ADMIN
            runAsNonRoot: false
            runAsUser: 0
      containers:
        - name: app
          image: python
          args: ["python", "-m", "http.server", "8080"]
        - name: envoy
          image: envoyproxy/envoy:v1.10.0
          securityContext:
            runAsUser: 1111
          volumeMounts:
          - readOnly: true
            mountPath: /config
            name: proxy-config
          args:
          - "envoy"
          - "--config-path"
          - "/config/envoy.yaml"
        - name: opa
          image: openpolicyagent/opa:0.12.1-istio-13
          securityContext:
            runAsUser: 1111
          volumeMounts:
          - readOnly: true
            mountPath: /policies
            name: app-policy
          args:
          - "--plugin-dir=/app"
          - "run"
          - "--server"
          - "--set=plugins.envoy_ext_authz_grpc.addr=:9191"
          - "--set=plugins.envoy_ext_authz_grpc.query=data.system.main"
          - "--ignore=.*"
          - "/policies"
      volumes:
        - name: app-policy
          configMap:
            name: app-policy
        - name: proxy-config
          configMap:
            name: proxy-config
```

### 5. Create Service to expose HTTP server

```bash
kubectl apply -f example-app-service.yaml
```

**example-app-service.yaml**:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: example-app
spec:
  selector:
    app: example-app
  ports:
  - name: http
    protocol: TCP
    port: 8080
    nodePort: 30808
  type: NodePort
```

> Depending on your Kubernetes cluster you may not want to use a NodePort
> service. Any kind of service will work for this example.

Get the Service's IP/hostname:

```bash
SERVICE_IP=$(kubectl get pod -l app=example-app -o jsonpath='{.items[].status.hostIP}')
echo $SERVICE_IP
```


### 6. Test app HTTP endpoint (expect 200)

```bash
curl -i $SERVICE_IP:30808
```

### 7. Update app-policy ConfigMap

Replace the existing app policy with one that will DENY ALL traffic to the app.

```
package system

main = false
```

Delete the app pod to make the update take affect. This will take several
seconds while the pod is gracefully terminated.

### 8. Test app HTTP endpoint (expect 403)

```bash
curl -i $SERVICE_IP:30808
```

## How Do I Write Policies?

The input document provided to your policies is defined by the Envoy External
Authorization API. For an example of the input document see the
[open-policy-agent/opa-istio-plugin
README](https://github.com/open-policy-agent/opa-istio-plugin#example-input).