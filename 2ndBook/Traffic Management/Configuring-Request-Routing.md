## Configuring Request Routing

### 준비
- Istio 설치 : https://github.com/grepsean/study-istio/blob/master/setup.md
- Bookinfo Sample Application 배포 : https://github.com/grepsean/study-istio/blob/master/examples.md
- Traffic Management 컨셉 확인 : https://istio.io/docs/concepts/traffic-management/

## Bookinfo Sample
- 이전에 살펴봤듯이 4개의 microservices로 구성되었다.
- 이 microservices중에 `reviews`는 내용이 다른 3가지 버전(v1, v2, v3)이 배포되어 있다.
- `/productpage`를 브라우저에서 열어보면 접근할때 마다 reviews쪽이 다른 스타일로 보여지게될 것이다.
  - 이는 default service version을 명시하지 않아서, istio가 round robin 스타일로 라우팅을 처리하기 때문이다.
- 이번 섹션에서는 우선 모든 트래픽을 v1으로 보내보자. 그리고나서 HTTP request header에 따라 라우팅하는것도 해보자.

## Virtual service 
우선 모든 요청을 v1으로 라우팅해보자.
0. 시작사기 앞서 bookinfo application의 default destination rules가 설정되어 있어야한다.
```console
$ kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
```

1. 아래를 실행시켜서 virtual services를 배포하자.
```console
$ kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```
  - 설정이 적용되기까지는 몇초 정도 소요된다. 
  
2. `$ kubectl get virtualservices -o yaml`을 실행하면 아래와 같이 배포된 VirtualService 내용을 확인할 수 있다.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: details
  ...
spec:
  hosts:
  - details
  http:
  - route:
    - destination:
        host: details
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage
  ...
spec:
  gateways:
  - bookinfo-gateway
  - mesh
  hosts:
  - productpage
  http:
  - route:
    - destination:
        host: productpage
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
  ...
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  ...
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
---
```
  - 위 설정에서 볼 수 있듯이, 모든 service에 대해서 subset을 v1 하나만 설정한것을 확인할 수 있다.
  
3. 또한 아래 커맨드로 정의되어 있는 subset을 확인할 수 있음
```console
$ kubectl get destinationrules -o yaml
```

### 적용된 라우팅 테스트
브라우저로 열어둔 `/productpage` 페이지를 새로고침하면 바로 확인가능하다.
- 이때 접근하는 URL은 `http://$GATEWAY_URL/productpage`
  - `$GATEWAY_URL` 환경변수의 경우 [Bookinfo Sample](https://github.com/grepsean/study-istio/blob/master/examples.md#ingress-ip%EC%99%80-port)에서 먼저 설명했듯이, ingress를 이용해서 외부에서 접근가능한 External IP(혹은 Host Name)이다.
  - 해당 URL로 접근하면 version `reviews:v1`인 review service로 트래픽이 전달되기 때문에, rating starts는 아무리 새로고침해도 보이지 않을 것이다. 
  
  
### 사용자의 Context에 따른 라우팅 
이번에는 이름이 Json이라는 사람이 접속했을때는 reviews:v2로 라우팅되게 해보자.
istio에서는 user의 identity를 알 수 있는 것을 지원하고 있지 않기때문에, 본 섹션에서는 end-user라는 custom header 내용을 기준으로 다른 reviews service로 라우팅할 것이다. (사실, productpage service에서 end-user라는 custom header에 사용자의 ID를 넣어준다)


1. 아래 커맨드로 user-based 라우팅을 배포해보자.
```console
$ kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

2. 배포된 virtual service를 확인해보자.
```console
$ kubectl get virtualservice reviews -o yaml
```
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  ...
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```
  - `spec.http.match.headers`를 통해서 `end-user`라는 header가 `jason`이 정확히(`exact`) 매칭되는지 설정되었다.
  
3. `/productpage` 페이지를 브라우저로 열고, `jason`이라는 ID로 로그인해보자. (Password도 `jason`)
  - 그리고 새로고침해보면, Reviews 부분에 Black stars 확인할 수 있을 것이다.
  - Password 부분은 아무거나 가능하다. 그냥 `session`에 `user`의 ID를 저장할 뿐이다.
    - https://github.com/istio/istio/tree/master/samples/bookinfo/src/productpage#L219
  
4. 만약 `jason`이 아니 다른 사용자로 로그인했다면, Black stars를 절대 볼 수 없을 것이다.
  - 위의 match에 걸리지 않았으므로, 기본 라우팅 설정인 `reviews:v1`으로 라우팅될 것이다. 
  
### Wrap up
- 본 섹션에서는 최초에는 `v1`으로 무조건 라우팅하다가, end-user라는 custom header를 보고 jason이라는 사람으로 로그인했을때 `v2`로 라우팅되게 해봤다.
- 쿠버네티스 services는 istio의 L7 라우팅 기능을 제대로 사용하기 위해서는 몇가지 사항을 만족해야한다. ([Requirements for Pods and Services](https://istio.io/docs/setup/kubernetes/prepare/requirements/)참고)
- Traffic Shifting 섹션에서는 하나의 버전에서 다른 버전으로 점진적으로 트래픽을 전환하는(traffic shifting) 기본적인 패턴에 대해서 배울 것이다.

### Cleanup
1. 아래 커맨드를 이용해 본 섹션에서 사용한 virtual service를 삭제할 수 있다.
```console
$ kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

2. 만약 다음 섹션들을 보지 않을 것이라면, [Bookinfo cleanup](https://github.com/grepsean/study-istio/blob/master/examples.md#cleanup) 섹션을 참고해서 예제에서 생성한 모든 Bookinfo resource를 삭제할 수 있다.
