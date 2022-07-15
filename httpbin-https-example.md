```markdown
Reference:
https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/

## 0. Precondition: Setup k8s cluster on AWS and deploy Neuvector images

## 1. Deploy httpbin service and sleep app (sleep app works as a http client)
kubectl label namespace default istio-injection=enabled
kubectl apply -f samples/httpbin/httpbin.yaml
kubectl apply -f samples/sleep/sleep.yaml

root@cilium-master:~/istio-1.14.1# kubectl get pods,svc
NAME                           READY   STATUS    RESTARTS   AGE
pod/httpbin-74fb669cc6-xd6bt   2/2     Running   0          35m
pod/sleep-557747455f-lk4js     2/2     Running   0          8d

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/httpbin      ClusterIP   10.100.65.251    <none>        8000/TCP   35m
service/kubernetes   ClusterIP   10.100.0.1       <none>        443/TCP    8d
service/sleep        ClusterIP   10.100.167.251   <none>        80/TCP     8d


## 2. Determining the ingress IP and ports
root@cilium-master:~/istio-1.14.1# kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP                                                               PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.100.35.173   a80c9f6f3b84f42f2bd23d116fc2b435-1684764718.us-east-1.elb.amazonaws.com   15021:30911/TCP,80:30070/TCP,443:31508/TCP,31400:32078/TCP,15443:31750/TCP   8d
root@cilium-master:~/istio-1.14.1#

root@cilium-master:~/istio-1.14.1#
root@cilium-master:~/istio-1.14.1# export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
root@cilium-master:~/istio-1.14.1# export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
root@cilium-master:~/istio-1.14.1# echo $SECURE_INGRESS_PORT, $INGRESS_HOST
443, a80c9f6f3b84f42f2bd23d116fc2b435-1684764718.us-east-1.elb.amazonaws.com


## 3. Generate client and server certificates and keys
### 3.1 Create a root certificate and private key to sign the certificates for your services
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=*.us-east-1.elb.amazonaws Inc./CN=us-east-1.elb.amazonaws.com' -keyout us-east-1.elb.amazonaws.com.key -out us-east-1.elb.amazonaws.com.crt

### 3.2 Create a certificate and a private key for *.us-east-1.elb.amazonaws.com
openssl req -out *.us-east-1.elb.amazonaws.com.csr -newkey rsa:2048 -nodes -keyout *.us-east-1.elb.amazonaws.com.key -subj "/CN=*.us-east-1.elb.amazonaws.com/O=*.us-east-1.elb.amazonaws organization"

openssl x509 -req -sha256 -days 365 -CA us-east-1.elb.amazonaws.com.crt -CAkey us-east-1.elb.amazonaws.com.key -set_serial 0 -in *.us-east-1.elb.amazonaws.com.csr -out *.us-east-1.elb.amazonaws.crt


## 4. Create a secret for the ingress gateway
kubectl create -n istio-system secret tls httpbin-credential --key=*.us-east-1.elb.amazonaws.com.key --cert=*.us-east-1.elb.amazonaws.crt


## 5. Create Virtual service and gateway
root@cilium-master:~/istio-1.14.1# cat vs-httpsbin.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: mygateway
spec:
  selector:
    istio: ingressgateway # use istio default ingress gateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: httpbin-credential # must be the same as secret
    hosts:
    - '*'
---

apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  #- "a80c9f6f3b84f42f2bd23d116fc2b435-1684764718.us-east-1.elb.amazonaws.com"
  - "*.us-east-1.elb.amazonaws.com"
  gateways:
  - mesh
  - mygateway
  http:
  - match:
    - uri:
        prefix: /status
    - uri:
        prefix: /delay
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
root@cilium-master:~/istio-1.14.1# kubectl apply -f vs-httpsbin.yaml

root@cilium-master:~/istio-1.14.1# kubectl get vs,gw
NAME                                         GATEWAYS               HOSTS                               AGE
virtualservice.networking.istio.io/httpbin   ["mesh","mygateway"]   ["*.us-east-1.elb.amazonaws.com"]   58m

NAME                                    AGE
gateway.networking.istio.io/mygateway   58m


## 6. Create service entry
root@cilium-master:~/istio-1.14.1# cat se-httpbin-https.yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: https-route-httpbin
  namespace: istio-system
spec:
  hosts:
  - "*.us-east-1.elb.amazonaws.com"
  location: MESH_INTERNAL
  ports:
  - name: https
    number: 443
    protocol: TLS
  resolution: DNS
 #resolution: STATIC
  endpoints:
  - address: istio-ingressgateway.istio-system.svc.cluster.local

root@cilium-master:~/istio-1.14.1# kubectl apply -f se-httpbin-https.yaml

root@cilium-master:~/istio-1.14.1# kubectl get se -n istio-system
NAME                  HOSTS                                                                                                       LOCATION        RESOLUTION   AGE
https-route-httpbin   ["*.us-east-1.elb.amazonaws.com"]                                                                           MESH_INTERNAL   DNS          59m
root@cilium-master:~/istio-1.14.1#


## 7. Send an HTTPS request to access the httpbin service through HTTPS
### 7.1 copy certificate to container
kubectl cp us-east-1.elb.amazonaws.com.crt sleep-557747455f-lk4js:/tmp/

### 7.2 container send HTTPS request to httpbin service through HTTPS
root@cilium-master:~#  kubectl exec -ti sleep-557747455f-lk4js sh
/tmp $ ls
us-east-1.elb.amazonaws.com.crt
/tmp $
/tmp $ curl --cacert us-east-1.elb.amazonaws.com.crt "https://a80c9f6f3b84f42f2bd23d116fc2b435-1684764718.us-east-1.elb.amazonaws.com:443/status/418"

    -=[ teapot ]=-

       _...._
     .'  _ _ `.
    | ."` ^ `". _,
    \_;`"---"`|//
      |       ;/
      \_     _/
        `"""`
```