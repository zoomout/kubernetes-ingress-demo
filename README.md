# Kubernetes ingress demo
## Prerequisites
- minikube
- kubectl
- helm

# Deploy Ingress

```
kubectl create namespace ingress-debug
kubectl config set-context $(kubectl config current-context) --namespace=ingress-debug
helm install stable/nginx-ingress --name my-ingress-debug
```

# Check ingress is deployed
```
K8S_CLUSTER_IP=`minikube ip`
curl $K8S_CLUSTER_IP
# should return "default backend - 404" - means the ingress with default backend is deployed
```

# Check NGINX ingress version
```
POD_NAME=$(kubectl get pods -l app=nginx-ingress -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD_NAME -- /nginx-ingress-controller --version
```

# Deploy demo applications
```
kubectl create -f cafe.yaml
kubectl create -f cafe-ingress.yaml
```

# Test deployed application
```
kubectl get services my-ingress-debug-nginx-ingress-controller
NAME                                        TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
my-ingress-debug-nginx-ingress-controller   LoadBalancer   XX.YY.ZZ.AA      <pending>     80:30867/TCP,443:30879/TCP   1m
APP_PORT=30867
curl --resolve cafe.example.com:$APP_PORT:$K8S_CLUSTER_IP http://cafe.example.com:$APP_PORT/coffee
curl --resolve cafe.example.com:$APP_PORT:$K8S_CLUSTER_IP http://cafe.example.com:$APP_PORT/tea
```

# Deploy two ingresses
It's possible to install multiple ingress controllers
```
#kubectl delete ns ingress-debug --grace-period=0 --force # to delete namespace
kubectl create namespace ingress-debug
kubectl config set-context $(kubectl config current-context) --namespace=ingress-debug
helm init
helm install stable/nginx-ingress --name ingress-coffee --set controller.ingressClass=ingress-coffee
helm install stable/nginx-ingress --name ingress-tea --set controller.ingressClass=ingress-tea

kubectl create -f cafe.yaml
kubectl create -f separate-ingresses.yaml

kubectl get services 
NAME                                           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-coffee-nginx-ingress-controller        LoadBalancer   XX.YY.ZZ.AA     <pending>     80:32454/TCP,443:30926/TCP   31s
ingress-tea-nginx-ingress-controller           LoadBalancer   XX.YY.ZZ.AA    <pending>     80:31339/TCP,443:31107/TCP   26s
APP_PORT_COFFEE=32454
APP_PORT_TEA=31339
K8S_CLUSTER_IP=`minikube ip`

alias curlIt="\
echo COFFEE:
curl --resolve coffee.example.com:$APP_PORT_COFFEE:$K8S_CLUSTER_IP http://coffee.example.com:$APP_PORT_COFFEE/coffee; echo;\
echo TEA:
curl --resolve tea.example.com:$APP_PORT_TEA:$K8S_CLUSTER_IP http://tea.example.com:$APP_PORT_TEA/tea; echo"

# Dump nginx config before and after scaling the cluster
kubectl get pods
NAME                                                           READY   STATUS    RESTARTS   AGE
ingress-coffee-nginx-ingress-controller-68945bf64d-xxmw4       1/1     Running   0          16m
COFFEE_INGRESS_POD=ingress-coffee-nginx-ingress-controller-68945bf64d-xxmw4

kubectl scale deployment/coffee --replicas=1
kubectl exec -it ${COFFEE_INGRESS_POD} cat /etc/nginx/nginx.conf > nginx1.conf
kubectl scale deployment/coffee --replicas=2
kubectl exec -it ${COFFEE_INGRESS_POD} cat /etc/nginx/nginx.conf > nginx2.conf

