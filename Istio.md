Refer to [Istio Installation](https://istio.io/docs/setup/kubernetes/quick-start/).

# Prerequisites
## Enviormemnt
AWS EC2 instances

## Install Kubernetes
Refer to [Kubernetes.md](https://github.com/alvenwong/docs/blob/master/Kubernetes.md)

# Install core components
```bash
kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml
kubectl apply -f install/kubernetes/istio-demo.yaml
```

## Verify the installation
```bash
kubectl get svc -n istio-system
kubectl get pods -n istio-system
```

# Deploy an application Bookinfo
## Install 
```bash
kubectl label namespace default istio-injection=enabled
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

## Confirm
```bash
kubectl get services
NAME                       CLUSTER-IP   EXTERNAL-IP   PORT(S)              AGE
details                    10.0.0.31    <none>        9080/TCP             6m
kubernetes                 10.0.0.1     <none>        443/TCP              7d
productpage                10.0.0.120   <none>        9080/TCP             6m
ratings                    10.0.0.15    <none>        9080/TCP             6m
reviews                    10.0.0.170   <none>        9080/TCP             6m
```

```bash
kubectl get pods
NAME                                        READY     STATUS    RESTARTS   AGE
details-v1-1520924117-48z17                 2/2       Running   0          6m
productpage-v1-560495357-jk1lz              2/2       Running   0          6m
ratings-v1-734492171-rnr5l                  2/2       Running   0          6m
reviews-v1-874083890-f0qf0                  2/2       Running   0          6m
reviews-v2-1343845940-b34q5                 2/2       Running   0          6m
reviews-v3-1813607990-8ch52                 2/2       Running   0          6m
```

## Set the ingress gateway
```bash
kubectl get gateway
NAME               AGE
bookinfo-gateway   32s
```

## Set ingress host and port
```bash
kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP              PORT(S)                         AGE
istio-ingressgateway   LoadBalancer   100.67.15.233   aa9140...amazonaws.com   80:31380/TCP,443:31390/TCPTCP   2h
```
If the EXTERNAL-IP is in the form of IP, like 130.211.10.121, your environment has an external load balancer that you can use for the ingress gateway. Set the ingress IP with 
```bash
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath=‘{.status.loadBalancer.ingress[0].ip}')
```
If the EXTERNAL-IP is in the form of a host name, such as aa9140...amazonaws.com, instead of an IP address, set the ingress IP with 
```bash
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```
Then set the ingress port with 
```bash
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
```

## Set the gateway url
```bash
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

## Confirm the app is running
```bash
curl -o /dev/null -s -w "%{http_code}\n" http://${GATEWAY_URL}/productpage
200
```
Or you can visit this product page from your browser with the url:
```bash
http://${GATEWAY_URL}/productpage
```

## Uninstall
```bash
samples/bookinfo/platform/kube/cleanup.sh
```

# Uninstall Istio core components
```bash
kubectl delete -f install/kubernetes/istio-demo.yaml
kubectl delete -f install/kubernetes/helm/istio/templates/crds.yaml -n istio-system
```

# Get into Kubernetes nodes

1. SSH to the EC2 instance on which you deploy Istio and Bookinfo. <br>
2. SSH to nodes. Two examples are shown as follows.
```bash
ssh -i ~/.ssh/id_rsa admin@api.brown.wangzhuang93.com
ssh -i ~/.ssh/id_rsa admin@ec2-13-58-36-99.us-east-2.compute.amazonaws.com
```

# Get container IP
```bash
sudo docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' Container_NAME_OR_ID
```
