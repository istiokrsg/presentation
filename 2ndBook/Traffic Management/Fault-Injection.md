## Fault Injection

### 준비
- Istio 설치 : https://github.com/grepsean/study-istio/blob/master/setup.md
- Bookinfo Sample Application 배포 : https://github.com/grepsean/study-istio/blob/master/examples.md
- Traffic Management 컨셉 확인 : https://istio.io/docs/concepts/traffic-management/
- [Traffic Routing 섹션](https://github.com/grepsean/study-istio/blob/master/Traffic%20Management/Configuring-Request-Routing.md#configuring-request-routing) 에서 살펴본 아래 실행해 보기
```console
$ kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
$ kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```
  - 이전 섹션에서의 라우팅 flow에 대한 설정 이해하기
    - `productpage` → `reviews:v2` → `ratings` (jason이라는 사용자만)
    - `productpage` → `reviews:v1` (이외의 모든 사용자)

### Injecting an HTTP delay fault
Bookinfo microservice에 대한 resiliency를 테스트하기 위해서 `7초` 정도의 딜레이가 생기게 해보자.
  - 단, `reviews:v2`와 `ratings` service 사이의 `json`이라는 사용자에 대해서만
이 테스트에서는 Bookinfo 내부에서 의도적으로 발생시킨 버그에 대해서는 다루지 않는다.

`reviews:v2` service에서는 `ratings` service를 호출시 `10초`의 timeout 설정이 하드코딩되어 있다.
따라서 `7초`의 딜레이를 주더라도, 우선 에러없이 정상적으로 서비스되어야 한다.

1. 우선 `json`이라는 사용자에 대해서 dealy를 발생시키는 fault injection rule을 생성해보자.
```console
$ kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
```

2. 생성된 rule을 확인해보자.
```console
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
  ...
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        fixedDelay: 7s
        percent: 100
    match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```
  - spec.http.fault를 통해서 delay를 생성했으며, `100%` 확률로 `7초`의 고정된 delay를 발생시킨다.
  - 사용가능한 옵션에 대해서는 공식문서의 [HTTPFaultInjection.Delay](https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/#HTTPFaultInjection-Delay) 섹션을 참고하자.

### 적용된 delay 설정을 테스트 해보자
1. [Bookinfo application](https://github.com/grepsean/study-istio/blob/master/examples.md)을 브라우저에서 실행시키자.

2. `/productpage`에서 `jason`으로 로그인해보면, 
  - 이 경우 `7초`의 delay가 발생되더라도, timeout 설정(`10초`)를 넘지 않으므로 에러없이 정상적으로 페이지가 로딩될 것으로 예상하겠지만,
  - 결국 아래와 같은 에러 메시지를 보게될 것이다.
  ```
  Error fetching product reviews!
  Sorry, product reviews are currently unavailable for this book.
  ```

3. 실제 웹페이지가 로딩된 시간을 확인해보자.
  - 우선 웹브라우저의 _Develop Tools_ 을 열어보자.
  - _Network_ 탭을 열어서
  - `productpage` 페이지를 리로드해보자. 
  - _Network_ 탭을 통해서 실제 로드하는데 `6초`가 걸린것을 확인할 수 있을 것이다.
  

### 왜 에러 메시지가 보이는 것인가?
정답은 바로 `reviews`라는 서비스에서 timeout설정에 의해 발생된 실패때문이다.

`productpage`-`reviews` 서비스 사이에는 `6초`의 timeout설정이 있는데, 실제로는 `3초` + `1재시도`에 의해 총 `6초` 처럼 보인다.
  - 이때 retry는 로직에서 for문 두번으로 처리한다.
    - https://github.com/istio/istio/blob/master/samples/bookinfo/src/productpage/productpage.py#L318
(`reviews`-`ratings` 서비스 사이의 timeout은 `10초`이지만) _fault inection_ 설정에 의해서 `ratings`를 호출하는데 7초의 delay가 발생하므로, `/productpage`를 호출하는 곳에서 timeout이 발생해버린다. 

Enterpise 환경에서는 서로 다른 팀에서 다른 microservices를 각각 개발하다보니 이러한 버그는 언제든지 발생할 수 있다. istio의 fualt injection은 이런 변칙적인 부분을 end user에게 영향없이 확인할 수 있게 도와준다.

`본 섹션에서는 jason이라는 사용자에게만 fault injection이 동작하므로, 다른 사용자로 로그인 한다면 전혀 영향이 없을 것이다.`


### 그럼 오류를 고쳐보자.
아마도 위의 문제를 아래의 방법으로 수정할 수 있을 것이다.
1. `productpage`의 timeout을 증가시키던지, `reviews`-`ratings` 서비스 사이의 timeout을 감소시키던지 한다.
2. 수정된 microservice를 중지 후 재시작한다.
3. `/productpage`에서 error가 발생되지 않는지 확인한다.

이러한 수정은 이미 `reviews:v3`버전에 적용되어 있으므로, 모든 traffic을 `reviews:v3`으로 흘려보낸다면, 간단하게 해결할 수 있다. 관련 내용은 [Traffic Shifting](https://istio.io/docs/tasks/traffic-management/traffic-shifting/)에서 살펴볼 수 있다.

`임의저긴 delay를 7s => 5s로 바꿔도 여전히 에러가 발생한다.`
왜냐면 productpage => reviews로 timeout은 `3초`이다. retry가 `1번`일뿐, 각 try마다 `5초`의 delay가 발생하므로 총 `3초`동안 `2번` 시도해서 총 `6초`가 걸리지만 에러는 여전할 것이다.

### Injecting an HTTP abort fault
이번에는 HTTP abort fault를 발생시켜보자. 본 섹션에서는 `ratings`라는 microservice에서 `jason`이라는 테스트 사용자가 로그인할 경우에 HTTP abort를 발생시켜 볼 것이다.
이 경우는 페이지를 로드하자마자 `Ratings service is currently unavailable`와 같은 오류 메시지가 나올 것이라고 예상할 수 있다.

1. 일단 샘플내에 미리 만들어놓은 설정을 apply하여 `jason`이라는 사용자에게 HTTP abort를 발생시키게 해보자.
```console
$ kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-abort.yaml
```

2. 생성된 `virtualservice`를 확인해보면
```console
$ kubectl get virtualservice ratings -o yaml
```
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
  ...
spec:
  hosts:
  - ratings
  http:
  - fault:
      abort:
        httpStatus: 500
        percent: 100
    match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```
  - spec.http.fault.abort를 통해서 100% 확률로 HTTP 500응답을 발생시킨다. 
  - 더 자세한 옵션에 대한 설명은 [HTTPFaultInjection.Abort](https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/#HTTPFaultInjection-Abort) 섹션을 참고하자.

### 적용된 abort 설정을 테스트해보자
1. [Bookinfo application](https://github.com/grepsean/study-istio/blob/master/examples.md)을 브라우저에서 실행시키자.

2. `/productpage`에서 `jason`으로 로그인해보면, (위의 abort 설정이 모든 pods에게 전파된 상태라면) 즉시 [`Ratings service is currently unavailable`](https://github.com/istio/istio/blob/fd77ab302dd72d9b687e81fbdc56af901838035a/samples/bookinfo/src/reviews/reviews-application/src/main/java/application/rest/LibertyRestEndpoint.java#L61)라는 에러 메시지를 확인할 수 있다.

3. `jason` 사용자를 로그아웃하거나, 다른 사용자로 로그인했다면, `/productpage`를 브라우저에서 열면 여전히 `reviews:v1`이 호출될 것이다. 따라서 위의 에러 메시지를 구경할 수 없을 것이다.

### Cleanup
1. 위에서 적용했던 routing rules를 삭제해보자. 
```console
$ kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

2. 만약 다음 섹션들을 보지 않을 것이라면, [Bookinfo cleanup](https://github.com/grepsean/study-istio/blob/master/examples.md#cleanup) 섹션을 참고해서 예제에서 생성한 모든 Bookinfo resource를 삭제할 수 있다.
