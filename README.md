# k8s-study install

kubernetes x microservices sample project 설치 및 테스트 가이드

## Prerequisites

- Kubernetes 1.9 이상
- Docker for mac(18.05.0-ce edge)과 AWS(kops 1.9)에서 테스트하였음
- MacOS에서 테스트하였음

### /etc/hosts

```
127.0.0.1 pongpong.io www.pongpong.io api.pongpong.io
```

## Setup

### Istio

- Istio client를 설치함 (0.7.1 이상)

```
# move to temp directory
cd ~/Downloads
curl -L https://git.io/getLatestIstio | sh -
export ISTIO_VESRION=0.7.1
mv istio-${ISTIO_VESRION}/bin/istioctl /usr/local/bin
# test
istioctl version
# cleanup
rm -rf istio-${ISTIO_VESRION}
```

- Istio를 설치함 (pilot, mixer, istio-auth, ...)
- Kong Ingress를 사용할 것이므로 istio ingress controller는 제외함
- 1.9부터 지원하는 auto injection도 사용하지 않음 (study 목적)
- mTLS도 사용하지 않음
- `no matches for config.istio.io` 에러 발생시 한번더 `apply`

```
kubectl apply -f install/istio.yaml
# check
kubectl -n istio-system get svc
kubectl -n istio-system get po
```

### Istio addons

- zipkin, prometheus, grafana, service graph

```
kubectl apply -f install/addons/zipkin.yaml
kubectl apply -f install/addons/prometheus.yaml
kubectl apply -f install/addons/grafana.yaml
kubectl apply -f install/addons/servicegraph.yaml
# check
kubectl -n istio-system get svc
kubectl -n istio-system get po
# port forwarding
kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=zipkin -o jsonpath='{.items[0].metadata.name}') 9411:9411 & # zipkin
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090 & # prometheus
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 & # grafana
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=servicegraph -o jsonpath='{.items[0].metadata.name}') 8088:8088 & # service graph
```

- prometheus test

expression sample

```
istio_request_count
istio_request_count{response_code="200"}
```

### Kong

- kong + kong ingress 설치

설치하는데 시간이 조금 걸리고 에러가 발생하지만 조금 기다려야함

```
kubectl apply -f install/kong-all-in-one-postgres.yaml
# check
kubectl -n kong get svc
watch kubectl -n kong get po
```

### Kong Plugin

- key auth 활성화

현재 버전(0.0.3)에서는 재설치시 오류발생

```
kubectl apply -f install/kong-plugins.yaml
# check
kubectl proxy
curl http://localhost:8001/api/v1/namespaces/kong/services/http:kong-ingress-controller:8001/proxy/consumers
curl http://localhost:8001/api/v1/namespaces/kong/services/http:kong-ingress-controller:8001/proxy/key-auths
```

### User Service

```
kubectl apply -f service/user-service/0_user-service-db.yml
istioctl kube-inject -f service/user-service/1_user-service.yml | kubectl apply -f -
# check
kubectl get svc
kubectl get po
# test
kubectl proxy
curl http://localhost:8001/api/v1/namespaces/kong/services/http:kong-ingress-controller:8001/proxy/services
curl http://localhost:8001/api/v1/namespaces/kong/services/http:kong-ingress-controller:8001/proxy/routes
curl http://localhost:8001/api/v1/namespaces/kong/services/http:kong-ingress-controller:8001/proxy/plugins
```

user 만들기 ([httpie](https://httpie.org/))

```
http POST http://api.pongpong.io/user-service/v1/signup email=subicura@subicura.com password=1234 apiKey:anonymous
http POST http://api.pongpong.io/user-service/v1/login email=subicura@subicura.com password=1234 apiKey:anonymous
```