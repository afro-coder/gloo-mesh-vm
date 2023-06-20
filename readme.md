# Gloo Mesh VM

This assume's Gloo Mesh is installed and workload clusters are registered

To check this please see
```bash
meshctl --kubecontext ${MGMT} check
```
Your output should look similar to

```
 meshctl check --kubeconfig kubeconfig

🟢 License status

 INFO  gloo-mesh trial license expiration is 30 Aug 23 13:48 IST
 INFO  Valid GraphQL license module found

🟢 CRD version check


🟢 Gloo Platform deployment status

Namespace | Name                  | Ready | Status
gloo-mesh | gloo-mesh-redis       | 1/1   | Healthy
gloo-mesh | gloo-mesh-mgmt-server | 1/1   | Healthy
gloo-mesh | gloo-mesh-ui          | 1/1   | Healthy
gloo-mesh | prometheus-server     | 1/1   | Healthy

🟢 Mgmt server connectivity to workload agents

Cluster  | Registered | Connected Pod
workload | true       | gloo-mesh/gloo-mesh-mgmt-server-ffc8496bf-gsm7z
```

Run this command to see the registered clusters.
```bash
pod=$(kubectl --context ${MGMT} -n gloo-mesh get pods -l app=gloo-mesh-mgmt-server -o jsonpath='{.items[0].metadata.name}')
kubectl --context ${MGMT} -n gloo-mesh debug -q -i ${pod} --image=curlimages/curl -- curl -s http://localhost:9091/metrics | grep relay_push_clients_connected
```

You should get an output similar to this:
```
# HELP relay_push_clients_connected Current number of connected Relay push clients (Relay Agents).
# TYPE relay_push_clients_connected gauge
relay_push_clients_connected{cluster="workload"} 1
```

If all is good here you can proceed.

### Environment variables 
```
# My Management Cluster is called mgmt and workload is called workload
export MGMT_CONTEXT="mgmt"

export REMOTE_CONTEXT="workload"
export GLOO_MESH_LICENSE_KEY=$GM

export MGMT="mgmt"

export CLUSTER1="workload"

# this has been added explicitly in the files below
export REPO="us-docker.pkg.dev/gloo-mesh/istio-a7d0a87b8a81"
export ISTIO_IMAGE="1.17.1-solo"
export REVISION="1-17-1"


export GLOO_MESH_VERSION=v2.3.1
#curl -sL https://run.solo.io/meshctl/install | sh -
export PATH=$HOME/.gloo-mesh/bin:$PATH

```

### Deploy Bookinfo
```
curl https://raw.githubusercontent.com/istio/istio/release-1.16/samples/bookinfo/platform/kube/bookinfo.yaml > bookinfo.yaml

kubectl --context ${CLUSTER1} create ns bookinfo-frontends
kubectl --context ${CLUSTER1} create ns bookinfo-backends
kubectl --context ${CLUSTER1} label namespace bookinfo-frontends istio.io/rev=1-17-1 --overwrite
kubectl --context ${CLUSTER1} label namespace bookinfo-backends istio.io/rev=1-17-1 --overwrite

# deploy the frontend bookinfo service in the bookinfo-frontends namespace
kubectl --context ${CLUSTER1} -n bookinfo-frontends apply -f bookinfo.yaml -l 'account in (productpage)'
kubectl --context ${CLUSTER1} -n bookinfo-frontends apply -f bookinfo.yaml -l 'app in (productpage)'
kubectl --context ${CLUSTER1} -n bookinfo-backends apply -f bookinfo.yaml -l 'account in (reviews,ratings,details)'
# deploy the backend bookinfo services in the bookinfo-backends namespace for all versions less than v3
kubectl --context ${CLUSTER1} -n bookinfo-backends apply -f bookinfo.yaml -l 'app in (reviews,ratings,details),version notin (v3)'
# Update the productpage deployment to set the environment variables to define where the backend services are running
kubectl --context ${CLUSTER1} -n bookinfo-frontends set env deploy/productpage-v1 DETAILS_HOSTNAME=details.bookinfo-backends.svc.cluster.local
kubectl --context ${CLUSTER1} -n bookinfo-frontends set env deploy/productpage-v1 REVIEWS_HOSTNAME=reviews.bookinfo-backends.svc.cluster.local
# Update the reviews service to display where it is coming from
kubectl --context ${CLUSTER1} -n bookinfo-backends set env deploy/reviews-v1 CLUSTER_NAME=${CLUSTER1}
kubectl --context ${CLUSTER1} -n bookinfo-backends set env deploy/reviews-v2 CLUSTER_NAME=${CLUSTER1}
```
Check the same
```
kubectl --context ${CLUSTER1} -n bookinfo-frontends get pods && kubectl --context ${CLUSTER1} -n bookinfo-backends get pods
```

### Deploy httpbin not-in-mesh
```
kubectl --context ${CLUSTER1} create ns httpbin
kubectl apply --context ${CLUSTER1} -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: not-in-mesh
  namespace: httpbin
---
apiVersion: v1
kind: Service
metadata:
  name: not-in-mesh
  namespace: httpbin
  labels:
    app: not-in-mesh
    service: not-in-mesh
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: not-in-mesh
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: not-in-mesh
  namespace: httpbin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: not-in-mesh
      version: v1
  template:
    metadata:
      labels:
        app: not-in-mesh
        version: v1
    spec:
      serviceAccountName: not-in-mesh
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: not-in-mesh
        ports:
        - containerPort: 80
EOF
```

#### Deploy httpbin in-mesh

```bash
kubectl --context ${CLUSTER1} apply -n httpbin -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: in-mesh
---
apiVersion: v1
kind: Service
metadata:
  name: in-mesh
  labels:
    app: in-mesh
    service: in-mesh
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: in-mesh
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: in-mesh
spec:
  replicas: 1
  selector:
    matchLabels:
      app: in-mesh
      version: v1
  template:
    metadata:
      labels:
        app: in-mesh
        version: v1
        istio.io/rev: 1-17-1
    spec:
      serviceAccountName: in-mesh
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: in-mesh
        ports:
        - containerPort: 80

EOF
```

#### Deploy Gloo Mesh Addons

```bash
kubectl --context ${CLUSTER1} create namespace gloo-mesh-addons
kubectl --context ${CLUSTER1} label namespace gloo-mesh-addons istio.io/rev=1-17-1

helm upgrade --install gloo-mesh-agent-addons gloo-mesh-agent/gloo-mesh-agent \
  --namespace gloo-mesh-addons \
  --kube-context=${CLUSTER1} \
  --set glooMeshAgent.enabled=false \
  --set rate-limiter.enabled=true \
  --set ext-auth-service.enabled=true \
  --version 2.3.1

```

### This assumes you have setup Gloo Mesh in the Management Cluster
Also please deploy the Gloo Mesh addons along with Bookinfo and Httpbin both in-mesh and mesh.


### Create a workspace entry for cross network communication

```bash
kubectl apply --context ${MGMT} -f - <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: WorkspaceSettings
metadata:
  name: global
  namespace: gloo-mesh
spec:
  options:
    eastWestGateways:
      - selector:
          labels:
            istio: eastwestgateway
EOF
```


#### Deploy Istio using ILM (Please Compare it with the files you have)

```bash
kubectl --context ${CLUSTER1} create ns istio-gateways
kubectl --context ${CLUSTER1} label namespace istio-gateways istio.io/rev=1-17-1

cat << EOF | kubectl --context ${CLUSTER1} apply -f -
apiVersion: v1
kind: Service
metadata:
  labels:
    app: istio-ingressgateway
    istio: ingressgateway
    revision: 1-17-1
  name: istio-ingressgateway
  namespace: istio-gateways
spec:
  ports:
  - name: http2
    port: 80
    protocol: TCP
    targetPort: 8080
  - name: https
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
    revision: 1-17-1
  type: LoadBalancer

EOF

cat << EOF | kubectl --context ${CLUSTER1} apply -f -
apiVersion: v1
kind: Service
metadata:
  labels:
    app: istio-ingressgateway
    istio: eastwestgateway
    revision: 1-17-1
    topology.istio.io/network: workload
  name: istio-eastwestgateway
  namespace: istio-gateways
spec:
  ports:
  - name: status-port
    port: 15021
    protocol: TCP
    targetPort: 15021
  - name: tls
    port: 15443
    protocol: TCP
    targetPort: 15443
  - name: https
    port: 16443
    protocol: TCP
    targetPort: 16443
  - name: tcp-istiod
    port: 15012
    protocol: TCP
    targetPort: 15012
  - name: tcp-webhook
    port: 15017
    protocol: TCP
    targetPort: 15017
  selector:
    app: istio-ingressgateway
    istio: eastwestgateway
    revision: 1-17-1 # Check revision
    topology.istio.io/network: workload # Should match the network
  type: LoadBalancer

EOF
```


#### Deploy ILM, GLM 

```bash
cat << EOF | kubectl --context ${MGMT} apply -f -

apiVersion: admin.gloo.solo.io/v2
kind: IstioLifecycleManager
metadata:
  name: workload-installation
  namespace: gloo-mesh
spec:
  installations:
    - clusters:
      - name: workload
        defaultRevision: true
      revision: 1-17-1
      istioOperatorSpec:
        profile: minimal
        hub: us-docker.pkg.dev/gloo-mesh/istio-a7d0a87b8a81
        tag: 1.17.1-solo
        namespace: istio-system
        values:
          global:
            meshID: mesh1
            multiCluster:
              clusterName: workload
            network: workload
        meshConfig:
          accessLogFile: /dev/stdout
          defaultConfig:
            proxyMetadata:
              ISTIO_META_DNS_CAPTURE: "true"
              ISTIO_META_DNS_AUTO_ALLOCATE: "true"
        components:
          pilot:
            k8s:
              env:
                - name: PILOT_ENABLE_K8S_SELECT_WORKLOAD_ENTRIES
                  value: "false"
          ingressGateways:
          - name: istio-ingressgateway
            enabled: false
EOF

cat << EOF | kubectl --context ${MGMT} apply -f -

apiVersion: admin.gloo.solo.io/v2
kind: GatewayLifecycleManager
metadata:
  name: workload-ingress
  namespace: gloo-mesh
spec:
  installations:
    - clusters:
      - name: workload
        activeGateway: false
      gatewayRevision: 1-17-1
      istioOperatorSpec:
        profile: empty
        hub: us-docker.pkg.dev/gloo-mesh/istio-a7d0a87b8a81
        tag: 1.17.1-solo
        values:
          gateways:
            istio-ingressgateway:
              customService: true
        components:
          ingressGateways:
            - name: istio-ingressgateway
              namespace: istio-gateways
              enabled: true
              label:
                istio: ingressgateway
---
apiVersion: admin.gloo.solo.io/v2
kind: GatewayLifecycleManager
metadata:
  name: workload-eastwest
  namespace: gloo-mesh
spec:
  installations:
    - clusters:
      - name: workload
        activeGateway: false
      gatewayRevision: 1-17-1
      istioOperatorSpec:
        profile: empty
        hub: us-docker.pkg.dev/gloo-mesh/istio-a7d0a87b8a81
        tag: 1.17.1-solo
        values:
          gateways:
            istio-ingressgateway:
              customService: true
        components:
          ingressGateways:
            - name: istio-eastwestgateway
              namespace: istio-gateways
              enabled: true
              label:
                istio: eastwestgateway
                topology.istio.io/network: workload
              k8s:
                env:
                  - name: ISTIO_META_ROUTER_MODE
                    value: "sni-dnat"
                  - name: ISTIO_META_REQUESTED_NETWORK_VIEW
                    value: workload
EOF
```

### VM Integration

Environment variables

```bash
export VM_APP="vm1"
export VM_NAMESPACE="virtualmachines"
export WORK_DIR="vm1"
export SERVICE_ACCOUNT="vm1-sa"
export CLUSTER_NETWORK=$(kubectl --context ${CLUSTER1} -n istio-gateways get svc -l istio=eastwestgateway -ojson | jq -r '.items[0].metadata.labels["topology.istio.io/network"]')
export VM_NETWORK="vm-network"
export CLUSTER="${CLUSTER1}"
```

Then, we need to create a directory where we'll store all the files that need to be used in our VM:

```bash
rm -rf ${WORK_DIR}
mkdir -p ${WORK_DIR}
```

Expose the port 15012 and 15017 of istiod through the Istio Ingress Gateway:

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: istiod-gateway
  namespace: istio-gateways
spec:
  selector:
    istio: eastwestgateway
  servers:
    - port:
        name: tcp-istiod
        number: 15012
        protocol: TCP
      hosts:
        - "*"
    - port:
        name: tcp-istiodwebhook
        number: 15017
        protocol: TCP
      hosts:
        - "*"
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: istiod-vs
  namespace: istio-gateways
spec:
  hosts:
  - istiod-1-17-1.istio-system.svc.cluster.local
  gateways:
  - istiod-gateway
  tcp:
  - match:
    - port: 15012
    route:
    - destination:
        host: istiod-1-17-1.istio-system.svc.cluster.local
        port:
          number: 15012
  - match:
    - port: 15017
    route:
    - destination:
        host: istiod-1-17-1.istio-system.svc.cluster.local
        port:
          number: 443
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: istiod-dr
  namespace: istio-gateways
spec:
  host: istiod-1-17-1.istio-system.svc.cluster.local
  trafficPolicy:
    portLevelSettings:
    - port:
        number: 15012
      tls:
        mode: DISABLE
    - port:
        number: 15017
      tls:
        mode: DISABLE

EOF
```

Create a Gateway resource that allows application traffic from the VMs to route correctly:

```bash
# Gateway to route traffic correctly
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: cross-network-gateway
  namespace: istio-gateways
spec:
  selector:
    istio: eastwestgateway
  servers:
  - port:
      number: 15443
      name: tls
      protocol: TLS
    tls:
      mode: AUTO_PASSTHROUGH
    hosts:
    - "*.local"
EOF
```

Create the namespace that will host the virtual machine:

```bash
kubectl --context ${CLUSTER1} create namespace "${VM_NAMESPACE}"
```

Create a serviceaccount for the virtual machine:

```bash
kubectl --context ${CLUSTER1} create serviceaccount "${SERVICE_ACCOUNT}" -n "${VM_NAMESPACE}"
```

Create a the WorkloadGroup yaml for the VM:

```bash
cat <<EOF > workloadgroup.yaml
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadGroup
metadata:
  name: "${VM_APP}"
  namespace: "${VM_NAMESPACE}"
spec:
  metadata:
    labels:
      app: "${VM_APP}"
  template:
    serviceAccount: "${SERVICE_ACCOUNT}"
    network: "${VM_NETWORK}"
EOF
```

```
kubectl apply  --context ${CLUSTER1} -f ./workloadgroup.yaml
```

Download istio 1.17.1, preferablly on the VM

```bash
curl -LO https://storage.googleapis.com/istio-release/releases/1.17.1/rpm/istio-sidecar.rpm
```

Use the istioctl x workload entry command to generate:

- cluster.env: Contains metadata that identifies what namespace, service account, network CIDR and (optionally) what inbound ports to capture.
- istio-token: A Kubernetes token used to get certs from the CA.
- mesh.yaml: Provides additional Istio metadata including, network name, trust domain and other values.
- root-cert.pem: The root certificate used to authenticate.
- hosts: An addendum to /etc/hosts that the proxy will use to reach istiod for xDS.*

```bash
istioctl --context ${CLUSTER1} x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" -r 1-17-1
```

Add an entry in the hosts file to resolve the address of istiod by the IP address of the Istio Ingress Gateway:

```bash
echo "$(kubectl --context ${CLUSTER1} -n istio-gateways get svc istio-eastwestgateway -o jsonpath='{.status.loadBalancer.ingress[0].*}') istiod.istio-system.svc istiod-1-17-1.istio-system.svc" > "${WORK_DIR}"/hosts
```
Output:
```
192.168.31.169 istiod.istio-system.svc istiod-1-17-1.istio-system.svc

```

*Copy the files stored in ${WORK_DIR} over to your VM - Via SSH*


Install the root certificate at /var/run/secrets/istio:
```
sudo mkdir -p /etc/certs
sudo cp root-cert.pem /etc/certs/root-cert.pem
```

Install the istio token at /var/run/secrets/tokens:
```
mkdir -p /var/run/secrets/tokens
sudo mkdir -p /var/run/secrets/tokens
sudo cp istio-token /var/run/secrets/tokens/istio-token
```

Install the sidecar
```
curl -LO https://storage.googleapis.com/istio-release/releases/1.17.1/rpm/istio-sidecar.rpm

sudo yum install -y istio-sidecar.rpm
```

Install cluster.env within the directory /var/lib/istio/envoy/:
```
sudo cp cluster.env /var/lib/istio/envoy/cluster
```

Install the Mesh Config to /etc/istio/config/mesh:
```
sudo cp mesh.yaml /etc/istio/config/mesh
```

Install the Mesh Config to /etc/istio/config/mesh:
```
sudo bash -c 'cat hosts >> /etc/hosts'
cat /etc/hosts
```
Transfer ownership to the Istio proxy:

```bash
sudo mkdir -p /etc/istio/proxy
chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem
sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem
```

Start the Istio SideCar
```
systemctl status istio.service
systemctl cat istio.service
```
Note: This doesn't fail at all on a normal centos8 vm with Firewall disabled and selinux disabled.

### Back to the Cluster

Let's update the bookinfo `Workspace` to include the `virtualmachines` namespace of the first cluster:

```bash
kubectl apply --context ${MGMT} -f - <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: Workspace
metadata:
  name: bookinfo
  namespace: gloo-mesh
  labels:
    allow_ingress: "true"
spec:
  workloadClusters:
  - name: workload
    namespaces:
    - name: bookinfo-frontends
    - name: bookinfo-backends
    - name: virtualmachines
EOF
```

#### On the VM
```
curl -v localhost:15000/clusters | grep productpage.bookinfo-frontends.svc.cluster.local
```
Output
```
* Connection #0 to host localhost left intact
outbound|9080||productpage.bookinfo-frontends.svc.cluster.local::observability_name::outbound|9080||productpage.bookinfo-frontends.svc.cluster.local
outbound|9080||productpage.bookinfo-frontends.svc.cluster.local::default_priority::max_connections::4294967295
outbound|9080||productpage.bookinfo-frontends.svc.cluster.local::default_priority::max_pending_requests::4294967295
outbound|9080||productpage.bookinfo-frontends.svc.cluster.local::default_priority::max_requests::4294967295
outbound|9080||productpage.bookinfo-frontends.svc.cluster.local::default_priority::max_retries::4294967295
outbound|9080||productpage.bookinfo-frontends.svc.cluster.local::high_priority::max_connections::1024
```

### On the VM

```
curl -I productpage.bookinfo-frontends.svc.cluster.local:9080/productpage
```
Output
```
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 5337
server: envoy
date: Tue, 20 Jun 2023 08:09:00 GMT
x-envoy-upstream-service-time: 553
```

Let's run the http server on the VM

```
python3 -m http.server  9999
```

Create the following on the K8s side
```
VM_IP=<YOUR VM IP>
```
We will now create a service entry to enable connection to the VM
```
cat <<EOF | kubectl --context ${CLUSTER1} apply -f -
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: ${VM_APP}
  namespace: virtualmachines
spec:
  hosts:
  - ${VM_APP}.virtualmachines.svc.cluster.local
  location: MESH_INTERNAL
  ports:
  - number: 9999
    name: http-vm
    protocol: TCP
    targetPort: 9999
  resolution: STATIC
  workloadSelector:
    labels:
      app: ${VM_APP}
EOF

cat <<EOF | kubectl --context ${CLUSTER1} apply -f -
apiVersion: networking.istio.io/v1beta1
kind: WorkloadEntry
metadata:
  name: ${VM_APP}
  namespace: virtualmachines
spec:
  network: ${CLUSTER_NETWORK}
  address: ${VM_IP}
  labels:
    app: ${VM_APP}
  serviceAccount: ${SERVICE_ACCOUNT}
EOF
```

Try to access the app from the `productpage` Pod:

```bash
kubectl --context ${CLUSTER1} -n bookinfo-frontends exec $(kubectl --context ${CLUSTER1} -n bookinfo-frontends get pods -l app=productpage -o jsonpath='{.items[0].metadata.name}') -- python -c "import requests; r = requests.get('http://${VM_APP}.virtualmachines.svc.cluster.local:9999'); print(r.text)"

```

Output
```
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
```
