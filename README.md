# Ambient Performance Test Setup

## Deploy cluster
Example using the `gcloud` CLI in the /gke-deploy directory. The 5 namespace application example below was run using 3x `n2-standard-8` instance types

## Application Notes
- 5 namespace isolated applications
- using [nicholasjackson/fake-service](https://github.com/nicholasjackson/fake-service) application
- 3-tier application, 4 deployments per namespace, 1 replicas per deployment 
  - A > B1,B2 > C
  - CPU requests: 700m // CPU limits: 700m (guaranteed QoS)
  - MEM requests: 500Mi // MEM limits: 500Mi (guaranteed QoS)

## add ambient helm repo
```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

## install istio-base
```bash
helm upgrade --install istio-base istio/base -n istio-system --version 1.21.0 --create-namespace
```

## install Kubernetes Gateway CRDs
```bash
echo "installing Kubernetes Gateway CRDs"
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.0.0" | kubectl apply -f -; }
```


## install istio-cni
```bash
helm upgrade --install istio-cni istio/cni \
-n kube-system \
--version=1.21.0 \
-f -<<EOF
profile: ambient
# uncomment below if using GKE
cni:
  cniBinDir: /home/kubernetes/bin
EOF
```

## install istiod
```bash
helm upgrade --install istiod istio/istiod \
-n istio-system \
--version=1.21.0 \
-f -<<EOF
profile: ambient
EOF
```

## install ztunnel

For GKE, ztunnel is expected to be deployed in `kube-system`
```bash
helm upgrade --install ztunnel istio/ztunnel \
-n kube-system \
--version=1.21.0 \
-f -<<EOF
hub: docker.io/istio
tag: 1.21.0
resources:
  requests:
      cpu: 500m
      memory: 2048Mi
istioNamespace: istio-system
# Additional volumeMounts to the ztunnel container
volumeMounts:
  - name: tmp
    mountPath: /tmp
# Additional volumes to the ztunnel pod
volumes:
  - name: tmp
    emptyDir:
      medium: Memory
EOF
```

# Configure an App

## deploy client into ambient mesh
```bash
kubectl apply -k client/ambient
```

## deploy httpbin into ambient mesh
```bash
kubectl apply -k httpbin/ambient
```

## exec into sleep client and curl httpbin /get endpoint to verify mTLS
```bash
kubectl exec -it deploy/sleep -n client -c sleep sh

curl httpbin.httpbin.svc.cluster.local:8000/get
```

## remove httpbin
```bash
kubectl delete -k httpbin/ambient
```


# Set up the Performance Test

## deploy 5 namespace tiered-app into ambient mesh
```bash
kubectl apply -k tiered-app/5-namespace-app/ambient
```

## exec into sleep client and curl tiered-app
```bash
kubectl exec -it deploy/sleep -n client sh

curl http://tier-1-app-a.ns-1.svc.cluster.local:8080
```

## watch logs of ztunnel for traffic interception
```bash
kubectl logs -n kube-system ds/ztunnel -f
```

Output should look similar to below, note you can see the spiffe ID of client sleep
```
2024-03-07T00:45:13.205154Z  INFO inbound{id=75f92cf47e739b015f76405a976d0359 peer_ip=10.32.1.7 peer_id=spiffe://cluster.local/ns/client/sa/sleep}: ztunnel::proxy::inbound: got CONNECT request to 10.32.3.6:80
2024-03-07T00:45:13.839633Z  INFO inbound{id=e737b49726c8c0a5b92e20c0ae6b7872 peer_ip=10.32.1.7 peer_id=spiffe://cluster.local/ns/client/sa/sleep}: ztunnel::proxy::inbound: got CONNECT request to 10.32.3.6:80
2024-03-07T00:45:14.377121Z  INFO inbound{id=1c8da768ba34c2eba072911c6a17b892 peer_ip=10.32.1.7 peer_id=spiffe://cluster.local/ns/client/sa/sleep}: ztunnel::proxy::inbound: got CONNECT request to 10.32.3.6:80
```

## deploy 5 vegeta loadgenerators
```bash
kubectl apply -k loadgenerators/5-loadgenerators
```

## watch logs of vegeta loadgenerator
```bash
kubectl logs -l app=vegeta -f -n ns-1
```

## watch top pods
```bash
watch kubectl top pods -n ns-1
watch kubectl top pods -n kube-system --sort-by cpu
```


## example exec into vegeta to run your own test (optional)
```bash
kubectl --namespace ns-1 exec -it deploy/vegeta -c vegeta -- /bin/sh
```

test run:
```bash
echo "GET http://tier-1-app-a.ns-1.svc.cluster.local:8080" | vegeta attack -dns-ttl=0 -rate 500/1s -duration=2s | tee results.bin | vegeta report -type=text
```

## uninstall
```bash
helm uninstall ztunnel -n kube-system
helm uninstall istiod -n istio-system
helm uninstall istio-cni -n kube-system
helm uninstall istio-base -n istio-system
kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.0.0" | kubectl delete -f -;
```