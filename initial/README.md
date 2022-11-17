![Gloo Mesh Enterprise](images/gloo-mesh-enterprise.png)
# <center>Gloo Mesh Setup</center>


## Table of Contents
* [Introduction](#introduction)
* [Lab 1 - Configure for Kubernetes cluster](#Lab-1)
* [Lab 2 - Deploy and register Gloo Mesh](#Lab-2)
* [Lab 3 - Deploy Istio using Gloo Mesh Lifecycle Manager](#Lab-3)

## Introduction <a name="introduction"></a>

[Gloo Mesh Enterprise](https://www.solo.io/products/gloo-mesh/) is a management plane which makes it easy to operate [Istio](https://istio.io) on one or many Kubernetes clusters deployed anywhere (any platform, anywhere).

### Istio support

The Gloo Mesh Enterprise subscription includes end to end Istio support:

- Upstream first
- Specialty builds available (FIPS, ARM, etc)
- Long Term Support (LTS) N-4 
- Critical security patches
- Production break-fix
- One hour SLA Severity 1
- Install / upgrade
- Architecture and operational guidance, best practices

### Gloo Mesh overview

Gloo Mesh provides many unique features, including:

- multi-tenancy based on global workspaces
- zero trust enforcement
- global observability (centralized metrics and access logging)
- simplified cross cluster communications (using virtual destinations)
- advanced gateway capabilities (oauth, jwt, transformations, rate limiting, web application firewall, ...)

![Gloo Mesh graph](images/gloo-mesh-graph.png)

### Want to learn more about Gloo Mesh

You can find more information about Gloo Mesh in the official documentation:

[https://docs.solo.io/gloo-mesh/latest/](https://docs.solo.io/gloo-mesh/latest/)



## Lab 1 - Configure for Kubernetes cluster<a name="Lab-1"></a>

Clone this repository and go to the directory where this `README.md` file is.

Make sure your Kubernetes context is set the Kubernetes cluster you will be using:

```
kubectl config get-contexts
```

Set the *MGMT* environment variable to the name of Kubernetes context you will be using:

```bash
export MGMT=<name-of-kubernetes-context>
```

Set the *CLUSTER1* environment variable to the *MGMT* value:

```bash
export CLUSTER1=$MGMT
```

## Lab 2 - Deploy and register Gloo Mesh <a name="Lab-2"></a>

First of all, let's install the `meshctl` CLI:

```bash
GLOO_MESH_VERSION=v2.1.0
curl -sL https://run.solo.io/meshctl/install | sh -
PATH=$HOME/.gloo-mesh/bin:$PATH
```

Set your the environment variable 
export GLOO_MESH_LICENSE_KEY to your Gloo Mesh License Key:

```
export GLOO_MESH_LICENSE_KEY=<set-to-your-license-key>
```

Run the following commands to deploy the Gloo Mesh management plane:

```bash
helm repo add gloo-mesh-enterprise https://storage.googleapis.com/gloo-mesh-enterprise/gloo-mesh-enterprise 
helm repo update
kubectl --context ${MGMT} create ns gloo-mesh 
helm upgrade --install gloo-mesh-enterprise gloo-mesh-enterprise/gloo-mesh-enterprise \
--namespace gloo-mesh --kube-context ${MGMT} \
--version=2.1.0 \
--set glooMeshMgmtServer.ports.healthcheck=8091 \
--set glooMeshUi.serviceType=LoadBalancer \
--set mgmtClusterName=${MGMT} \
--set global.cluster=${MGMT} \
--set licenseKey=${GLOO_MESH_LICENSE_KEY}
kubectl --context ${MGMT} -n gloo-mesh rollout status deploy/gloo-mesh-mgmt-server
```
<!--bash
kubectl --context ${MGMT} scale --replicas=0 -n gloo-mesh deploy/gloo-mesh-ui
-->
<!--bash
kubectl wait --context ${MGMT} --for=condition=Ready -n gloo-mesh --all pod
until [[ $(kubectl --context ${MGMT} -n gloo-mesh get svc gloo-mesh-mgmt-server -o json | jq '.status.loadBalancer | length') -gt 0 ]]; do
  sleep 1
done
-->
Then, you need to set the environment variable to tell the Gloo Mesh agents how to communicate with the management plane:

```bash
export ENDPOINT_GLOO_MESH=gloo-mesh-mgmt-server:9900
export HOST_GLOO_MESH=$(echo ${ENDPOINT_GLOO_MESH} | cut -d: -f1)
```

Check that the variables have correct values:
```
echo $HOST_GLOO_MESH
echo $ENDPOINT_GLOO_MESH
```

Finally, you need to register the cluster(s).

Here is how you register the first one:

```bash
helm repo add gloo-mesh-agent https://storage.googleapis.com/gloo-mesh-enterprise/gloo-mesh-agent
helm repo update

kubectl apply --context ${MGMT} -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: KubernetesCluster
metadata:
  name: cluster1
  namespace: gloo-mesh
spec:
  clusterDomain: cluster.local
EOF

kubectl --context ${CLUSTER1} create ns gloo-mesh

helm upgrade --install gloo-mesh-agent gloo-mesh-agent/gloo-mesh-agent \
  --namespace gloo-mesh \
  --kube-context=${CLUSTER1} \
  --set relay.serverAddress=${ENDPOINT_GLOO_MESH} \
  --set relay.authority=gloo-mesh-mgmt-server.gloo-mesh \
  --set rate-limiter.enabled=false \
  --set ext-auth-service.enabled=false \
  --set cluster=cluster1 \
  --version 2.1.0
```

Note that the registration can also be performed using `meshctl cluster register`.

Wait a minute and then you can check the cluster(s) have been registered correctly using the following commands:

```
pod=$(kubectl --context ${MGMT} -n gloo-mesh get pods -l app=gloo-mesh-mgmt-server -o jsonpath='{.items[0].metadata.name}')
kubectl --context ${MGMT} -n gloo-mesh debug -q -i ${pod} --image=curlimages/curl -- curl -s http://localhost:9091/metrics | grep relay_push_clients_connected
```

You should get an output similar to this:

```
# HELP relay_push_clients_connected Current number of connected Relay push clients (Relay Agents).
# TYPE relay_push_clients_connected gauge
relay_push_clients_connected{cluster="cluster1"} 1
```

You can try the command a few times until the output is similar to the above.

## Lab 3 - Deploy Istio <a name="Lab-3"></a>

We are going to deploy Istio using Helm, but there are several other options. You can find more information in the [Istio documentation](https://istio.io/latest/docs/setup/install/).

First of all, let's Download the Istio release 1.15.1:

```bash
export ISTIO_VERSION=1.15.1
curl -L https://istio.io/downloadIstio | sh -
```

Then, you need to create the `istio-system` and the `istio-gateways` namespaces on the first cluster.

```bash
kubectl --context ${CLUSTER1} create ns istio-system
kubectl --context ${CLUSTER1} create ns istio-gateways
```

Now, let's deploy the Istio control plane on the first cluster:

```bash
helm --kube-context=${CLUSTER1} upgrade --install istio-base ./istio-1.15.1/manifests/charts/base -n istio-system --set defaultRevision=1-15

helm --kube-context=${CLUSTER1} upgrade --install istio-1.15.1 ./istio-1.15.1/manifests/charts/istio-control/istio-discovery -n istio-system --values - <<EOF
revision: 1-15
global:
  meshID: mesh1
  multiCluster:
    clusterName: cluster1
  network: network1
  hub: us-docker.pkg.dev/gloo-mesh/istio-workshops
  tag: 1.15.1-solo
meshConfig:
  trustDomain: cluster1
  accessLogFile: /dev/stdout
  enableAutoMtls: true
  defaultConfig:
    envoyMetricsService:
      address: gloo-mesh-agent.gloo-mesh:9977
    envoyAccessLogService:
      address: gloo-mesh-agent.gloo-mesh:9977
    proxyMetadata:
      ISTIO_META_DNS_CAPTURE: "true"
      ISTIO_META_DNS_AUTO_ALLOCATE: "true"
pilot:
  env:
    PILOT_ENABLE_K8S_SELECT_WORKLOAD_ENTRIES: "false"
    PILOT_SKIP_VALIDATE_TRUST_DOMAIN: "true"

EOF
```

Note that we set the `trust domain` to be the same as the cluster name and we configure the sidecars to send their metrics and access logs to the Gloo Mesh agent.

After that, you can deploy the gateway(s):

```bash
kubectl --context ${CLUSTER1} label namespace istio-gateways istio.io/rev=1-15

helm --kube-context=${CLUSTER1} upgrade --install istio-ingressgateway ./istio-1.15.1/manifests/charts/gateways/istio-ingress -n istio-gateways --values - <<EOF
global:
  hub: us-docker.pkg.dev/gloo-mesh/istio-workshops
  tag: 1.15.1-solo
gateways:
  istio-ingressgateway:
    name: istio-ingressgateway
    namespace: istio-gateways
    labels:
      istio: ingressgateway
    injectionTemplate: gateway
    ports:
    - name: http2
      port: 80
      targetPort: 8080
    - name: https
      port: 443
      targetPort: 8443
EOF
```

As you can see, we deploy the control plane (istiod) in the `istio-system` and gateway(s) in the `istio-gateways` namespace.

Run the following command until all the Istio Pods are ready:

```bash
kubectl --context ${CLUSTER1} get pods -n istio-system && kubectl --context ${CLUSTER1} get pods -n istio-gateways
```

When they are ready, you should get this output:

```
NAME                      READY   STATUS    RESTARTS   AGE
istiod-5c669bcf6f-2hn6c   1/1     Running   0          3m7s
NAME                                     READY   STATUS    RESTARTS   AGE
istio-ingressgateway-744fcf4fb-5dc7q     1/1     Running   0          2m44s
```
Set the environment variable for the service corresponding to the Istio Ingress Gateway of the cluster(s):

```bash
export ENDPOINT_HTTP_GW_CLUSTER1=$(kubectl --context ${CLUSTER1} -n istio-gateways get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].*}'):80
export ENDPOINT_HTTPS_GW_CLUSTER1=$(kubectl --context ${CLUSTER1} -n istio-gateways get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].*}'):443
export HOST_GW_CLUSTER1=$(echo ${ENDPOINT_HTTP_GW_CLUSTER1} | cut -d: -f1)
```
