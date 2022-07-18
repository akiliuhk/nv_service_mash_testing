# Protecting Istio-managed Microservices with NeuVector



## Overview

This document describes two specific use cases of NeuVector to protect the ingress traffic to Istio Ingress Controller.



## Testing Environment

1. Deployed Rancher 2.6.5

2. Provisioned a RKE2-based downstream kubernetes (v1.23.6) cluster with the following cluster tools deployed.

   * Rancher Monitoring (Prometheus/Grafana)
   * Rancher Istio
   * NeuVector with the following custom built images
     * docker.io/neuvector/controller:eg.1
     * docker.io/neuvector/enforcer:eg.1
     * docker.io/neuvector/manager:eg.1

3. Testing Host:

   SLES 15 SP3 with kubectl installed and configured to connect to the above RKE2 cluster

   Check out the testing manifests into the testing host

   ```bash
   git clone https://github.com/dsohk/nv_service_mesh_testing
   
   cd nv_service_mesh_testing
   ```

   



## Test Scenarios

1. Wildcard test in service mesh
2. HTTPS test in service mesh



### 1. Wildcard test in Service Mesh 



Step 1 - Create `http-gateway` in the Istio system

```
kubectl create -f manifests/istio-http-gateway.yaml
```

Verify the http-gateway is deployed successfully

```bash
❯ kubectl get gateway -n istio-system
NAME           AGE
http-gateway   2m51s
```





Step 2 - Deploy the demo apps into namespace demox and demoy

```bash
kubectl create -f manifests/demox-one-gfeng.yaml
kubectl create -f manifests/demoy-one-gfeng.yaml
```

The output would be similar to as follows:

```bash
❯ kubectl create -f manifests/demox-one-gfeng.yml
namespace/demox created
deployment.apps/demox-app-1 created
deployment.apps/demox-app-2 created
service/demox-app-1-svc created
service/demox-app-2-svc created
virtualservice.networking.istio.io/demox-app-1-vs created
virtualservice.networking.istio.io/demox-app-2-vs created
❯ kubectl create -f manifests/demoy-one-gfeng.yml
namespace/demoy created
deployment.apps/demoy-app-1 created
deployment.apps/demoy-app-2 created
service/demoy-app-1-svc created
service/demoy-app-2-svc created
virtualservice.networking.istio.io/demoy-app-1-vs created
virtualservice.networking.istio.io/demoy-app-2-vs created
```

Let's verify the deployed resources

```bash
kubectl get pods,svc -n demox
```





Step 3 - Create Service Entry



```
❯ kubectl create -f istio-service-entry.yaml
serviceentry.networking.istio.io/http-route created

❯ kubectl get se -n istio-system
NAME         HOSTS                                                                                                       LOCATION        RESOLUTION   AGE
http-route   ["*.us-east-1.elb.amazonaws.com","demox-app-2-svc.demox","demoy-app-1-svc.demoy","demoy-app-2-svc.demoy"]   MESH_INTERNAL   DNS          76s
```



```bash
❯ kubectl get vs -n demox
NAME             GATEWAYS                        HOSTS                                                                         AGE
demox-app-1-vs   ["istio-system/http-gateway"]   ["a89aed554d8884c218b83a834a0787cb-1414116015.us-east-1.elb.amazonaws.com"]   9m11s
demox-app-2-vs   ["mesh"]                        ["demox-app-2-svc.demox"]
```



```bash
❯ kubectl get vs -n demoy
NAME             GATEWAYS                        HOSTS                       AGE
demoy-app-1-vs   ["istio-system/http-gateway"]   ["demoy-app-1-svc.demoy"]   8m42s
demoy-app-2-vs   ["mesh"]                        ["demoy-app-2-svc.demoy"]   8m42s
```



