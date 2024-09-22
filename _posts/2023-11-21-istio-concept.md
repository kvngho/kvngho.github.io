---
title: Istio 개념
date: 2023-11-21
categories: ["NETWORK", "ISTIO"]
tags: ['kubenetes', 'network', 'istio']     # TAG names should always be lowercase
toc: true
---

# Istio

## Goal
1. Istio 개념이해
2. 로컬환경에서 테스트해보기
   
## 필요성
마이크로 서비스 아키텍쳐는 여러 장점을 가지고 있긴하지만, 그에 따른 많은 단점또한 가지고 있다. 
예를 들어 2개의 내부 서비스와 1개의 외부 API로 구성된 시스템을 생각해보자.
```
Service A -> Service B -> External API 
```
이런식으로 호출이 이루어졌다고 가정하고, External API에 장애가 생겼다고 생각해보자. External API를 호출하고 있는 Service B 안에 쓰레드는 응답을 기다리기 위해 대기 상태가 된다.
이 상태로 클라이언트에서 Service A에 대한 호출이 계속 되면, 결국 Service A 안에 다른 쓰레드들도 응답을 받기 위하여 대기 상태가 된다. 

이러한 상황이 반복되면 결국 Service A 내부에 가용할 수 있는 쓰레드는 없어지고 Service A도 문제가 발생하게 된다. 이러한 현상을 ***장애 전파*** 라고 한다.

이러한 현상이 반복되고, 반복되는 문제를 해결하기 위하여 여러 디자인 패턴이 생기게 된다. 예를 들어 위의 문제는 Circuit Breack 라는 패턴으로 해결할 수 있다.
```
// Circuit Breaker
Service A - Circuit Breaker - Service B - External API
```
이런식으로 서비스 사이에 Circuit breaker라는 개념을 적용해서 네트워크 트래픽을 감시하고, Service B에 장애가 발생하게 되면 해당 연결을 끊어서 Service A를 대기상태에 놓이게 하는것이 아니고 바로 에러를 전달하는 것이다. 이렇게하면 Service A까지 장애가 전파되는 것을 방지할 수 있다.

Circuit breaker 말고도 여러가지 패턴들로 문제를 해결할 수 있는데, 몇가지 문제점이 존재한다.
1. 패턴들을 모두 공부해야하고, 공개된 오픈소스들의 사용법이 복잡하다.
2. 애플리케이션 계층에서 동작하게 되면 모든 서비스에 해당 기능을 직접 추가하여야한다.

이러한 문제점들을 해결하고자 등장한 것이 ***서비스 메쉬***라는 아키텍쳐 컨셉이다.

## Service Mesh?

> 서비스 메쉬는 MSA 환경에서 마이크로 서비스간 통신을 제어하고 관리할 수 있는 ***인프라계층***

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*qzMhV4O6AJBSyTrMoO91rg.png)

위의 이미지 처럼 마이크로서비스 옆에 Sidecar형식으로 프록시를 붙여서(***!서비스 내부에 붙이는게 아님***) 마이크로서비스간의 통신을 제어한다.
 앞에 언급한 Circuit breaker 또한 Sidecar proxy에서 연결을 관리함으로써 구현이 가능해진다.
### Service Mesh를 왜 써야할까?

> 관찰가능성, 트래픽 관리, 보안과 같은 기능을 ***자체 코드에 추가하지 않고 추가할 수있다.***
>
-> 기존 마이크로서비스에 코드를 추가하지 않고, 계층을 추가하여서 모니터링이 가능해진다.


## Istio ?
Kubernets 클러스터에서 컨테이너를 연결, 모니터링, 보호하는 구성 가능한 오픈서비스-메시 계층
> 서비스 메쉬를 구현할 수 있는 오픈소스

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*pIg02RVBOzOeRoLWkHlAbg.png)


## Istio의 구조 및 구성 요소 ([공식문서](https://istio.io/latest/docs/ops/deployment/architecture/))
![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FekDnSh%2FbtqH48PFImk%2F6LqtDdyX1Ii3Hr7l4J7HuK%2Fimg.png)

Istio는 크게 두가지의 요소로 구성되어 있다.
### 1. Data Plane
Data Plane은 ***서비스와 사이드카로 배포된 Envoy proxy***의 집합이다. Envoy를 통해서 서비스에 들어오고 나가는 모든 트래픽을 통제하고 관리한다.
### 2. Control Plane
Data Plane(Envoy)를 컨트롤하는 부분이다. 

version 1.4까지는 pilot, mixer, citadel, galley로 구성되어있었고, 1.5부터는 istiod라는 하나의 모듈로 통합되었다. 
istiod는 크게 ***서비스 디스커버리, 설정 관리, 인증 관리 등을 수행한다.***

## [Istio 설치](https://istio.io/latest/docs/setup/install/istioctl/)
```shell
# 다운로드
> curl -L https://istio.io/downloadIstio | sh -
# 폴더 이동
> cd istio-1.20.0
# 바이너리 파일 위치 환경 변수 저장
# 그냥 ~/.zshrc에 추가하는게 편함
> export PATH=$PWD/bin:$PATH
```

### 1. Install Istio
istio의 demo [configuration profile](https://istio.io/latest/docs/setup/additional-setup/config-profiles/)을 다운로드 받고 설치한다.
> Istio는 기본적으로 몇가지의 configuration profile을 지원하는데, 여기서 설치하는 demo profile은 `istio-egressgateway`, ` istio-ingressgateway`, `istiod` 등의 컴포넌트를 포함한다.
```shell
❯ istioctl install --set profile=demo -y
✔ Istio core installed
✔ Istiod installed
✔ Ingress gateways installed
✔ Egress gateways installed
✔ Installation complete                                                                                                 
Made this installation the default for injection and validation
```

Istio의 기능을 이용하기 위해서는 서비스 메시안에 있는 Pod이 사이드카 프록시를 실행해야한다. 크게 [두가지 방법](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/)이 있다:
1. Pod가 존재하는 namespace에서 sidecar injection을 활성화
```shell
❯ kubectl label namespace default istio-injection=enabled
namespace/default labele
```
```shell
❯ kubectl label namespace default --list
istio-injection=enabled
kubernetes.io/metadata.name=default
```
namespace 말고도 pod에도 레이블을 지정함으로써 sidecar injection이 가능하다.
(sidecar.istio.io/inject:"true")

정상적으로 default 네임스페이스에 레이블링이 되었다.
2. istioctl을 이용한 수동 sidecar injection
```shell
❯ istioctl kube-inject -f samples/sleep/sleep.yaml | kubectl apply -f -
serviceaccount/sleep created
service/sleep created
deployment.apps/sleep created
```

### 2. 샘플 앱 배포
![](https://istio.io/latest/docs/examples/bookinfo/noistio.svg)
```shell
❯ kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/bookinfo/platform/kube/bookinfo.yaml
service/details created
...중략...
deployment.apps/productpage-v1 created
```
위와 같은 어플리케이션을 배포하게 되면, 아까 sidecar injection을 활성화했으므로 각 서비스에 sidecar 형식으로 envoy proxy가 활성화 된다.
![](https://istio.io/latest/docs/examples/bookinfo/withistio.svg)
(각 서비스의 코드 수정없이 envoy 프록시가 부착되었다.)

```shell
// 정상적으로 작동하는지 확인
❯ kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"

<title>Simple Bookstore App</title>
```
> ratings 서비스의 Pod에서 ProductPage로 http get 요청을 보내고, 해당 페이지의 제목을 추출한다.

### 3. [Istio Ingress Gateway](https://istio.io/latest/docs/concepts/traffic-management/#gateways)를 통한 메쉬와의 통신
해당 어플리케이션은 아직 외부와의 통신이 불가능하다. Istio Ingress Gateway을 통하여 외부와 통신을 할 수 있게 해보자
```shell
❯ kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/bookinfo/networking/bookinfo-gateway.yaml
gateway.networking.istio.io/bookinfo-gateway created
virtualservice.networking.istio.io/bookinfo created
❯ istioctl analyze
✔ No validation issues found when analyzing namespace: default.
```
(참고: 해당 yaml 내용)
```shell
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  # The selector matches the ingress gateway pod labels.
  # If you installed Istio using Helm following the standard documentation, this would be "istio=ingress"
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 8080
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```

### 4. Ingress 확인
```shell
❯ export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
❯ echo $INGRESS_HOST
127.0.0.1
❯ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
❯ export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
❯ export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
❯ echo "$GATEWAY_URL"
127.0.0.1:80
```
브라우저로 http://127.0.0.1:80/productpage 에 접속하면, 정상적으로 ingress가 작동하는것을 알수있다.

### 5. 대시보드 확인 (Kiali, Prometheus, Grafana, Jaegar)
```shell
❯ kubectl apply -f samples/addons
❯ kubectl rollout status deployment/kiali -n istio-system
❯ istioctl dashboard kiali
```
![](/assets/blogimg/monitor.png)

