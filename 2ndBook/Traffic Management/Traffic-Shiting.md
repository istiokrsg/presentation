## Traffic Shifting
이번 섹션에서는 traffic을 old versio에서 new version으로 점진적으로 전환하는 방법을 살펴볼 것이다.
istio에서는 이런 방법을 route rules에서 traffic의 percent를 설정을 통해서 가능하다. 
본 섹션에서는 우선 `reviews:v1`에서 `reviews:v3`로의 트래픽을 50%만 설정했다가, 100%로 변경하여 `reviews:v3`로 트래픽을 완전히 전환하는 방법을 살펴보도록하자.

### 준비
- Istio 설치 : https://github.com/grepsean/study-istio/blob/master/setup.md
- Bookinfo Sample Application 배포 : https://github.com/grepsean/study-istio/blob/master/examples.md
- Traffic Management 컨셉 확인 : https://istio.io/docs/concepts/traffic-management/

### Weight-based routing
Bookinfo에서의 destination rule을 설정한 적이 없다면, [Apply Default Destination Rules](https://istio.io/docs/examples/bookinfo/#apply-default-destination-rules)를 통해서 Default Destination rule을 먼저 설정해보자.

1. 우선 아래 커맨드를 통해서 모든 트래픽을 `v1`으로 향하게 하자.
```console
$ kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

2. 브라우저에서 `/productpage` 페이지를 열어보자.
- 이때 접근하는 URL은 `http://$GATEWAY_URL/productpage`
  - `$GATEWAY_URL` 환경변수의 경우 [Bookinfo Sample](https://github.com/grepsean/study-istio/blob/master/examples.md#ingress-ip%EC%99%80-port)에서 먼저 설명했듯이, ingress를 이용해서 외부에서 접근가능한 External IP(혹은 Host Name)이다.
  - 해당 URL로 접근하면 version `reviews:v1`인 review service로 트래픽이 전달되기 때문에, rating starts는 아무리 새로고침해도 보이지 않을 것이다. 
  
3. 이제 `reviews:v1`에서 `reviews:v3`fh 50%의 트래픽을 전환해보자.
```console
$ kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
```
  - rules가 전파되는데 몇초 정도 기다리자. 

4. 적용된 virtualservice를 확인해보자.
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
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v3
      weight: 50
```
  - spec.http.route.desitination.weight로 각 destination에 해당하는 `v1`과 `v3`에 대해서 wegith를 50으로 설정하자.

5. `/productpage`를 브라우저에서 새로고침해보면, 50% 확률로 red star가 보이게 될 것이다.
```
주의 : 최신 Envoy side의 구현에 따르면, /product 페이지를 많이 새로고침(15번 이상)해야 배포된 설정를 확인할 수 있다.
만약 v3로 90% weight 설정을 한다면, 더 많은 새로고침이 필요하게 된다.
```

6. 만약 `reviews:v3`에 대해서 안정된 버전이라는 것이 검증되었다면, 이제 트래픽을 100%으로 변경해야할 것이다.
```console
$ kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml
```
  - 이제 부터는 `/productpage`로 새로고침해보면, red start만이 보이게 될 것이다.

### Wrap it up
이번 섹션에서는 `reviews` 서비스에 대해서 old version에서 new version으로 트래픽을 전환하기 위해서 istio의 weighted routing 기능을 사용해보았다.
이렇게 istio에서 트래픽을 전환하는 방법은 Container ochestration platform에서의 사용하는 feature(instance의 scaling을 조정하는 방법)와는 다른 방법이다. 
istio에서는 두 버전의 service 각각에 대해서, 두 서버스간의 트래픽 분산에 대해 영향없이 scaling up/down 가능하다. 
istio를 이용한 Autoscaling에 관심이 있다면, [Canary Deployments using Istio](https://istio.io/blog/2017/0.1-canary/)를 살펴보자.

### Cleanup
1. 배포한 routing rules 삭제하자.
```console
$ kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

2. 만약 다음 섹션들을 보지 않을 것이라면, [Bookinfo cleanup](https://github.com/grepsean/study-istio/blob/master/examples.md#cleanup) 섹션을 참고해서 예제에서 생성한 모든 Bookinfo resource를 삭제할 수 있다.
