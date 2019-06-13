# Working K8S [OK]

minikube destroy
minikube start --cpus 4 --memory 8192


# Working istio [OK]

tar -xvzf https://github.com/istio/istio/releases/download/1.0.8/istio-1.0.8-linux.tar.gz istio

helm template istio/install/kubernetes/helm/istio --name istio --namespace istio-system --set kiali.enabled=true --set sidecarInjectorWebhook.enabled=true > istio.yaml
kubectl apply -f istio/install/kubernetes/helm/istio/templates/crds.yaml

kubectl create namespace istio-system
kubectl apply -f ./istio.yaml

kubectl label namespace default istio-injection=enabled
kubectl apply -f ./kiali-secret.yaml


# Working book [0K] [WITH ELK]

kubectl apply -f istio/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f istio/samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl apply -f istio/samples/bookinfo/networking/destination-rule-all.yaml


kubectl delete -f istio/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl delete -f istio/samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl delete -f istio/samples/bookinfo/networking/destination-rule-all.yaml


# Working FluentD + ELK [OK]

kubectl apply -f fluentd-elk

kubectl delete -f fluentd-elk


# Working inbound test with httpbin [OK]

kubectl apply -f istio/samples/httpbin/httpbin.yaml
kubectl apply -f inbound-simple

kubectl delete -f istio/samples/httpbin/httpbin.yaml
kubectl delete -f inbound-simple

Tricks to host : https://istio.io/docs/tasks/traffic-management/ingress/#accessing-ingress-services-using-a-browser


# Working outbound test with sleep [OK]

kubectl apply -f istio/samples/sleep/sleep.yaml
kubectl apply -f outbound-simple

kubectl delete -f istio/samples/sleep/sleep.yaml
kubectl delete -f outbound-simple


# Working crossbound single host passthrough with ingress gw, route & vs [OK]

kubectl apply -f crossbound-simple

kubectl delete -f crossbound-simple


# Working crossbound https passthrough

# Working outbound with strongswan in front of ingress

# Useful


## Exposure


### K8S Minikube Dashboard

minikube dashboard &


### Kiali

kubectl port-forward svc/kiali 20001:20001 -n istio-system &


### Kibana

kubectl -n logging port-forward $(kubectl -n logging get pod -l app=kibana -o jsonpath='{.items[0].metadata.name}') 5601:5601 &


### ELK

kubectl -n logging port-forward $(kubectl -n logging get pod -l app=elasticsearch -o jsonpath='{.items[0].metadata.name}') 9200:9200 &


## Persisted

export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
export INGRESS_HOST=$(minikube ip)
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})


## Checks


### Check which namespace has sidecar injection

kubectl get namespace -L istio-injection


### Check for ingress gateway exposure

kubectl get svc istio-ingressgateway -n istio-system


### Check for egress gateway

kubectl get svc istio-egressgateway -n istio-system


### Sleep to Httpbin inside mesh query

kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name}) -c sleep -- curl -v httpbin:8000/status/418


### Book sample inside mesh query

kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"


### Inbound httpbin query

curl -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/status/200


### Outbound sleep to httpbin.org query

kubectl exec -it $SOURCE_POD -c sleep -- curl -v http://httpbin.org/status/200


### Crossbound httpbin.org query

curl -I -HHost:httpbin.org http://$INGRESS_HOST:$INGRESS_PORT/status/200


### Outbound tls sleep to httpbin.org

kubectl exec -it $SOURCE_POD -c sleep -- curl -v https://httpbin.org/status/200


### Crossbound TLS passthough ( notice there is no HTTP logging, only TCP )

curl -v --resolve httpbin.org:$SECURE_INGRESS_PORT:$INGRESS_HOST https://httpbin.org:$SECURE_INGRESS_PORT/status/404

### Outbound httpbin egress query

kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://httpbin.org/status/300

### Check egress logs

kubectl logs -l istio=egressgateway -c istio-proxy -n istio-system | tail
