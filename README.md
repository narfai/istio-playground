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

# Working FluentD + ELK [OK]

kubectl apply -f fluentd-elk/logging-stack.yaml
kubectl apply -f fluentd-elk/fluentd-istio-crd.yaml

kubectl delete -f fluentd-elk/logging-stack.yaml
kubectl delete -f fluentd-elk/fluentd-istio-crd.yaml

# Working inbound test with httpbin [OK]

kubectl apply -f istio/samples/httpbin/httpbin.yaml
kubectl apply -f inbound-simple/httpbin-gateway.yaml
kubectl apply -f inbound-simple/httpbin-routes-vs.yaml

Tricks to host : https://istio.io/docs/tasks/traffic-management/ingress/#accessing-ingress-services-using-a-browser

# Working outbound test with sleep [OK]

kubectl apply -f istio/samples/sleep/sleep.yaml
kubectl apply -f outbound-simple/httpbin-service-entry.yaml


# Working crossbound single host passthrough with ingress gw, route & vs [OK]

kubectl apply -f crossbound-simple/httpbin-cross-gateway.yaml
kubectl apply -f crossbound-simple/httpbin-cross-routes-vs.yaml
kubectl apply -f crossbound-simple/httpbin-cross-service-entry.yaml

# Working crossbound passthrough only with istio gw & egress / ingress

# Working crossbound https passthrough

# Working outbound with strongswan in front of ingress

## Exposure

minikube dashboard &;
kubectl port-forward svc/kiali 20001:20001 -n istio-system &
kubectl -n logging port-forward $(kubectl -n logging get pod -l app=kibana -o jsonpath='{.items[0].metadata.name}') 5601:5601 &
kubectl -n logging port-forward $(kubectl -n logging get pod -l app=elasticsearch -o jsonpath='{.items[0].metadata.name}') 9200:9200 &


## Persisted

export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
export INGRESS_HOST=$(minikube ip)
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})

## Checks

kubectl get namespace -L istio-injection
kubectl get svc istio-ingressgateway -n istio-system
kubectl get gateway
kubectl get destinationrules -o yaml
Inside com 		# kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name}) -c sleep -- curl -v httpbin:8000/status/418
Book inbound 	# kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
Inbound 		# curl -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/status/200
Outbound 		# kubectl exec -it $SOURCE_POD -c sleep -- curl -v http://httpbin.org/status/200
Cross			# curl -I -HHost:httpbin.org http://$INGRESS_HOST:$INGRESS_PORT/status/200
