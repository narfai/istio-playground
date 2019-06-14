# Experimentations

## TODO put corresponding task & examples links


## Exposure


### K8S Minikube Dashboard

```

minikube dashboard &

```

### Kiali

http://localhost:20001/kiali

```

kubectl port-forward svc/kiali 20001:20001 -n istio-system &

```

### Kibana

```

kubectl -n logging port-forward $(kubectl -n logging get pod -l app=kibana -o jsonpath='{.items[0].metadata.name}') 5601:5601 &

```

### ELK

```

kubectl -n logging port-forward $(kubectl -n logging get pod -l app=elasticsearch -o jsonpath='{.items[0].metadata.name}') 9200:9200 &

```

## Dynamic values

```

export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
export INGRESS_HOST=$(minikube ip)
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})

```


## Working K8S [OK]

https://github.com/kubernetes/minikube/blob/master/docs/drivers.md#kvm2-driver

```

minikube destroy
minikube start --cpus 4 --memory 8192 --kubernetes-version v1.13.0 # latest istio-compatible stable release


```


## Working istio LTS way

wget https://github.com/istio/istio/releases/download/1.0.8/istio-1.0.8-linux.tar.gz
tar -xvzf istio-1.0.8-linux.tar.gz
rm istio-1.0.8-linux.tar.gz
mv istio-1.0.8 istio

helm template istio-lts/install/kubernetes/helm/istio --name istio --namespace istio-system --set kiali.enabled=true --set sidecarInjectorWebhook.enabled=true > istio-lts.yaml
kubectl apply -f istio-lts/install/kubernetes/helm/istio/templates/crds.yaml

kubectl create namespace istio-system
kubectl apply -f ./istio-lts.yaml

kubectl label namespace default istio-injection=enabled
kubectl apply -f ./kiali-secret.yaml

## Working istio > 1.1 [OK]

https://istio.io/docs/setup/kubernetes/install/helm/
https://istio.io/docs/reference/config/installation-options/

```

cd playground

wget https://github.com/istio/istio/releases/download/1.1.8/istio-1.1.8-linux.tar.gz
tar -xvzf istio*.tar.gz
rm istio*.tar.gz
mv istio-* istio

kubectl create namespace istio-system
kubectl label namespace default istio-injection=enabled

helm template istio/install/kubernetes/helm/istio-init --name istio-init --namespace istio-system | kubectl apply -f -

wait for the 53 CRDs ( 58 with SDS ) : kubectl get crds | grep 'istio.io\|certmanager.k8s.io'

helm template istio/install/kubernetes/helm/istio \
--name istio \
--namespace istio-system \
--values istio/install/kubernetes/helm/istio/values-istio-demo-auth.yaml > istio.yaml

kubectl apply -f ./istio.yaml
kubectl apply -f ./kiali-secret.yaml

kubectl delete -f ./istio.yaml
kubectl delete -f ./kiali-secret.yml
kubectl delete namespace istio-system

```

## With SDS and cert manager

https://istio.io/docs/examples/advanced-gateways/ingress-certmgr/

## Working book API samples [0K]

```

kubectl apply -f istio/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f istio/samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl apply -f istio/samples/bookinfo/networking/destination-rule-all.yaml


kubectl delete -f istio/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl delete -f istio/samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl delete -f istio/samples/bookinfo/networking/destination-rule-all.yaml


```

## Working FluentD + ELK [OK]

```

kubectl apply -f fluentd-elk

kubectl delete -f fluentd-elk


```

## Working inbound test with httpbin [OK]

```

kubectl apply -f istio/samples/httpbin/httpbin.yaml
kubectl apply -f inbound-simple

kubectl delete -f istio/samples/httpbin/httpbin.yaml
kubectl delete -f inbound-simple

Tricks to host : https://istio.io/docs/tasks/traffic-management/ingress/#accessing-ingress-services-using-a-browser


```

## Working outbound test with sleep [OK]

```

kubectl apply -f istio/samples/sleep/sleep.yaml
kubectl apply -f outbound-simple

kubectl delete -f istio/samples/sleep/sleep.yaml
kubectl delete -f outbound-simple


```

## Working crossbound single host passthrough with ingress gw, route & vs [OK]

```

kubectl apply -f crossbound-simple

kubectl delete -f crossbound-simple


```

## Working crossbound https passthrough [OK]

```

kubectl apply -f crossbound-https-passthrough

kubectl delete -f crossbound-https-passthrough


```

## Working outbound with egress control [OK]

```

kubectl apply -f istio/samples/sleep/sleep.yaml
kubectl apply -f outbound-egress

kubectl delete -f istio/samples/sleep/sleep.yaml
kubectl delete -f outbound-egress


```

## Working outbound egress with SNI proxy

```

kubectl apply -f istio/samples/sleep/sleep.yaml

kubectl create configmap egress-sni-proxy-configmap -n istio-system --from-file=nginx.conf=./outbound-sni-proxy/sni-proxy.conf

cat outbound-sni-proxy/istio-egressgateway-with-sni-proxy-base.yaml | helm template istio/install/kubernetes/helm/istio/ \
--name istio-egressgateway-with-sni-proxy --namespace istio-system \
-x charts/gateways/templates/deployment.yaml \
-x charts/gateways/templates/service.yaml \
-x charts/gateways/templates/serviceaccount.yaml \
-x charts/gateways/templates/autoscale.yaml \
-x charts/gateways/templates/clusterrole.yaml \
-x charts/gateways/templates/clusterrolebindings.yaml \
--set global.istioNamespace=istio-system -f - > ./outbound-sni-proxy/istio-egressgateway-with-sni-proxy.yaml

kubectl apply -f ./outbound-sni-proxy/istio-egressgateway-with-sni-proxy.yaml
kubectl apply -f ./outbound-sni-proxy/static-entry-se.yaml
kubectl apply -f ./outbound-sni-proxy/wikipedia-tls-se.yaml
kubectl apply -f ./outbound-sni-proxy/egress-sni-proxy-mutual-tls-gw-dr-se.yaml

kubectl delete --ignore-not-found=true envoyfilter forward-downstream-sni egress-gateway-sni-verifier
kubectl delete -f ./istio-egressgateway-with-sni-proxy.yaml
kubectl delete configmap egress-sni-proxy-configmap -n istio-system

```

## Working tunnelled crossbound strongswan to egress to net HTTP

```


```

## Working tunnelled crossbound strongswan to egress to net TLS

```

```


# Useful

## Checks


### Check which namespace has sidecar injection

```

kubectl get namespace -L istio-injection

```

### Check for ingress gateway exposure

```

kubectl get svc istio-ingressgateway -n istio-system

```

### Check for egress gateway

```

kubectl get svc istio-egressgateway -n istio-system

```

### Sleep to Httpbin inside mesh query

```

kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name}) -c sleep -- curl -v httpbin:8000/status/418

```

### Book sample inside mesh query

```

kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"

```

### Inbound httpbin query

```

curl -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/status/200

```

### Outbound sleep to httpbin.org query

```

kubectl exec -it $SOURCE_POD -c sleep -- curl -v http://httpbin.org/status/200

```

### Crossbound httpbin.org query

```

curl -I -HHost:httpbin.org http://$INGRESS_HOST:$INGRESS_PORT/status/200

```

### Outbound tls sleep to httpbin.org

```

kubectl exec -it $SOURCE_POD -c sleep -- curl -v https://httpbin.org/status/200

```

### Crossbound TLS passthough ( notice there is no HTTP logging, only TCP )

```

curl -v --resolve httpbin.org:$SECURE_INGRESS_PORT:$INGRESS_HOST https://httpbin.org:$SECURE_INGRESS_PORT/status/404

```

### Outbound httpbin egress query

```

kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://httpbin.org/status/300

```

### Check egress logs

```

kubectl logs -l istio=egressgateway -c istio-proxy -n istio-system | tail

```

### Check egress SNI gateway availability

```

kubectl get pod -l istio=egressgateway-with-sni-proxy -n istio-system

```

### Check for istio deployed crd

```

kubectl get crds

```


# Prod ready

## Extending Istio Root CA

https://istio.io/blog/2019/root-transition/ => update : latest istio CA is 10 years now

## Service tagging

app & version
