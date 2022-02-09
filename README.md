# 1. Pre-requisites

* Kublr 1.21.2+
* jq
* kubectl 1.21+

# 2. Deploy k8s clusters

Create `aws` and `azure` secret credentials in `default` space using Kublr UI or API.

Create clusters `demo-hybrid-1-aws` and `demo-hybrid-2-azure` in `default` space via Kublr UI or API using cluster specificactions
from the files `clusters/demo-hybrid-1-aws.yaml` and `clusters/demo-hybrid-2-azure.yaml` in this project.

Wait for all statuses become green, including 2 packages in the end of the status page - `submariner-additional-configuration`, `submariner-broker`.

# 3. Download clusters' kubeconfig files

Download the clusters' kubeconfig files from Kublr UI or API and save them in your current work directory as `config1aws` and `config2az`.

Verify that the clusters are available for CLI tools:

```
export KUBECONFIG=config1aws
kubectl get nodes
export KUBECONFIG=config2az
kubectl get nodes
```

# 4. Connect clusters with submariner

## 4.1. Install subctl

Submariner provides a CLI tool `subctl` that simplifies some operations.

```
wget https://github.com/submariner-io/submariner-operator/releases/download/subctl-release-0.11/subctl-release-0.11-linux-amd64.tar.xz
tar xfv subctl-release-0.11-linux-amd64.tar.xz
mv subctl-release-0.11/subctl-release-0.11-linux-amd64 subctl
```

## 4.2. Prepare common submariner parameters

```
export KUBECONFIG=config1aws

export BROKER_NS=submariner-k8s-broker
export SUBMARINER_NS=submariner-operator
export SUBMARINER_PSK=$(LC_CTYPE=C tr -dc 'a-zA-Z0-9' < /dev/urandom | fold -w 64 | head -n 1)

export SUBMARINER_BROKER_CA=$(kubectl -n "${BROKER_NS}" get secrets \
    -o jsonpath="{.items[?(@.metadata.annotations['kubernetes\.io/service-account\.name']=='${BROKER_NS}-client')].data['ca\.crt']}")

export SUBMARINER_BROKER_TOKEN=$(kubectl -n "${BROKER_NS}" get secrets \
    -o jsonpath="{.items[?(@.metadata.annotations['kubernetes\.io/service-account\.name']=='${BROKER_NS}-client')].data.token}" \
       | base64 --decode)

export SUBMARINER_BROKER_URL="$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}' | grep -E -o -i '([^/]+):[0-9]+')"

echo
echo "KUBECONFIG=${KUBECONFIG}" ; echo
echo "BROKER_NS=${BROKER_NS}" ; echo
echo "SUBMARINER_NS=${SUBMARINER_NS}" ; echo
echo "SUBMARINER_PSK=${SUBMARINER_PSK:+***}" ; echo
echo "SUBMARINER_BROKER_CA=${SUBMARINER_BROKER_CA}" ; echo
echo "SUBMARINER_BROKER_TOKEN=${SUBMARINER_BROKER_TOKEN:+***}" ; echo
echo "SUBMARINER_BROKER_URL=${SUBMARINER_BROKER_URL}" ; echo
```

## 4.3. Join AWS cluster

Deploy submariner operator to the AWS cluster

```
export KUBECONFIG=config1aws
export CLUSTER_ID=demo-hybrid-1-aws
export CLUSTER_CIDR=100.96.0.0/11
export SERVICE_CIDR=100.64.0.0/13

echo
echo "KUBECONFIG=${KUBECONFIG}" ; echo
echo "CLUSTER_ID=${CLUSTER_ID}" ; echo
echo "CLUSTER_CIDR=${CLUSTER_CIDR}" ; echo
echo "SERVICE_CIDR=${SERVICE_CIDR}" ; echo

helm upgrade -i submariner-operator https://submariner-io.github.io/submariner-charts/charts/submariner-operator-0.11.0.tgz \
        --create-namespace \
        --namespace "${SUBMARINER_NS}" \
        --set ipsec.psk="${SUBMARINER_PSK}" \
        --set broker.server="${SUBMARINER_BROKER_URL}" \
        --set broker.token="${SUBMARINER_BROKER_TOKEN}" \
        --set broker.namespace="${BROKER_NS}" \
        --set broker.ca="${SUBMARINER_BROKER_CA}" \
        --set submariner.cableDriver=libreswan \
        --set submariner.clusterId="${CLUSTER_ID}" \
        --set submariner.clusterCidr="${CLUSTER_CIDR}" \
        --set submariner.serviceCidr="${SERVICE_CIDR}" \
        --set submariner.globalCidr="${GLOBAL_CIDR}" \
        --set serviceAccounts.globalnet.create="" \
        --set submariner.natEnabled="true" \
        --set brokercrd.create=false
```

## 4.4. Join Azure cluster

Deploy submariner operator to the Azure cluster

```
export KUBECONFIG=config2az
export CLUSTER_ID=demo-hybrid-2-azure
export CLUSTER_CIDR=100.160.0.0/11
export SERVICE_CIDR=100.128.0.0/13

echo
echo "KUBECONFIG=${KUBECONFIG}" ; echo
echo "CLUSTER_ID=${CLUSTER_ID}" ; echo
echo "CLUSTER_CIDR=${CLUSTER_CIDR}" ; echo
echo "SERVICE_CIDR=${SERVICE_CIDR}" ; echo

helm upgrade -i submariner-operator https://submariner-io.github.io/submariner-charts/charts/submariner-operator-0.11.0.tgz \
        --create-namespace \
        --namespace "${SUBMARINER_NS}" \
        --set ipsec.psk="${SUBMARINER_PSK}" \
        --set broker.server="${SUBMARINER_BROKER_URL}" \
        --set broker.token="${SUBMARINER_BROKER_TOKEN}" \
        --set broker.namespace="${BROKER_NS}" \
        --set broker.ca="${SUBMARINER_BROKER_CA}" \
        --set submariner.cableDriver=libreswan \
        --set submariner.clusterId="${CLUSTER_ID}" \
        --set submariner.clusterCidr="${CLUSTER_CIDR}" \
        --set submariner.serviceCidr="${SERVICE_CIDR}" \
        --set submariner.globalCidr="${GLOBAL_CIDR}" \
        --set serviceAccounts.globalnet.create="" \
        --set submariner.natEnabled="true" \
        --set brokercrd.create=false
```

## 4.5. Wait for VPN connection to be set up

```
./subctl show all
```

# 5. Test inter-pod/service connectivity

## 5.1. Deploy test server pods

```
for C in config1aws config2az ; do
kubectl --kubeconfig=$C apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: test-srv-pod
  labels: { app: test-srv-pod }
spec:
  selector: { app: test-srv-pod }
  ports:
  - name: http
    port: 5000
    targetPort: 5000
    protocol: TCP
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: test-srv-pod
  labels: { app: test-srv-pod }
spec:
  selector: { matchLabels: { app: test-srv-pod } }
  template:
    metadata: { labels: { app: test-srv-pod } }
    spec:
      containers:
        - name: srv
          image: quay.io/gravitational/netbox:latest
      terminationGracePeriodSeconds: 1
      tolerations:
        - operator: Exists
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: test-srv-host
  labels: { app: test-srv-host }
spec:
  selector: { matchLabels: { app: test-srv-host } }
  template:
    metadata: { labels: { app: test-srv-host } }
    spec:
      hostNetwork: true
      containers:
        - name: srv
          image: quay.io/gravitational/netbox:latest
      terminationGracePeriodSeconds: 1
      tolerations:
        - operator: Exists
EOF
done
```

Wait for all pods to start on all nodes

```
kubectl --kubeconfig=config1aws get pods -l app=test-srv-pod
kubectl --kubeconfig=config1aws get pods -l app=test-srv-host
kubectl --kubeconfig=config2az get pods -l app=test-srv-pod
kubectl --kubeconfig=config2az get pods -l app=test-srv-host
```

## 5.2. Test that inter-pod connections work by IP

```
for C1 in config1aws config2az ; do
  for P1 in $(kubectl --kubeconfig=$C1 get pods -l app=test-srv-pod -o name); do
    for C2 in config1aws config2az ; do
      for P2 in $(kubectl --kubeconfig=$C2 get pods -l app=test-srv-pod -o name); do
        A2="$(kubectl --kubeconfig=$C2 get $P2 -o jsonpath='{.status.podIP}')"
        echo -n -e "$C1\t$P1\t-> $C2\t$P2\t$A2:\t"
        kubectl --kubeconfig=$C1 exec $P1 -- curl -q http://$A2:5000 1>/dev/null 2>&1 && echo OK || echo NOK
      done
    done
  done
done
```

## 5.3. Test pod-to-service-ip connections

```
for C2 in config1aws config2az ; do
  A2="$(kubectl --kubeconfig=$C2 get svc test-srv-pod -o jsonpath='{.spec.clusterIP}')"
  for C1 in config1aws config2az ; do
    for P1 in $(kubectl --kubeconfig=$C1 get pods -l app=test-srv-pod -o name); do
      echo -n -e "$C1\t$P1\t-> $C2\t$A2:\t"
      kubectl --kubeconfig=$C1 exec $P1 -- curl -q http://$A2:5000 1>/dev/null 2>&1 && echo OK || echo NOK
    done
  done
done
```

## 5.4. Test pod-to-service-name connections

```
for C2 in cluster1.local cluster2.local ; do
  A2="test-srv-pod.default.svc.$C2"
  for C1 in config1aws config2az ; do
    for P1 in $(kubectl --kubeconfig=$C1 get pods -l app=test-srv-pod -o name); do
      echo -n -e "$C1\t$P1\t-> $C2\t$A2:\t"
      kubectl --kubeconfig=$C1 exec $P1 -- curl -q http://$A2:5000 1>/dev/null 2>&1 && echo OK || echo NOK
    done
  done
done
```

## 5.5. Test Submariner service exporting

```
export KUBECONFIG=config1aws

kubectl apply -f - <<EOF
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: test-srv-pod
EOF

# Alternatively:
subctl export service --namespace default test-srv-pod-1-aws
```

Test:

```
A2=test-srv-pod.default.svc.clusterset.local
for C1 in config1aws config2az ; do
  for P1 in $(kubectl --kubeconfig=$C1 get pods -l app=test-srv-pod -o name); do
    echo -n -e "$C1\t$P1\t-> $C2\t$A2:\t"
    kubectl --kubeconfig=$C1 exec $P1 -- curl -q http://$A2:5000 1>/dev/null 2>&1 && echo OK || echo NOK
  done
done
```

# 6. Reference

* https://github.com/submariner-io/
* https://github.com/submariner-io/submariner
* https://github.com/submariner-io/submariner-operator
* https://github.com/submariner-io/submariner-charts
* https://github.com/kubernetes-sigs/mcs-api
* https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/1645-multi-cluster-services-api
