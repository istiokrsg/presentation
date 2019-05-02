## Circuit Breaking
이번 섹션에서는 connection, requests, outlier detection에 대한 circuit breaking 설정하는 방법에 대해서 살펴본다.
Circuit breaking은 resilient한 miscroserice application을 만들기 위한 중요한 패턴중 하나이다. 
따라서 특별한 failure 상황이나, latency가 갑자기 튄다던지하는 원치않는 장애 상황에서 circuit breaker를 사용해서 어떻게 대처하는지 살펴보자.


### 준비 
- Istio 설치 : https://github.com/grepsean/study-istio/blob/master/setup.md
- [httpbin](https://github.com/istio/istio/tree/release-1.1/samples/httpbin) 샘플 설치
  - autumatic sidecar injection이 설정되어 있다면,
  ```console
  $ kubectl apply -f samples/httpbin/httpbin.yaml
  ```
  - autumatic sidecar injection이 설정되어 있지 않다면,
  ```console
  $ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)
  ```

### Circuit breaker 설정
1. destination rule을 통해서 circuit breaking 설정을 해보자.
  - 주의 : `mutual TLS Authentication enabled` 환경이라면, DestinationRule을 apply하기전에 TLS traffic policy에 `mode: ISTIO_MUTUAL`와 같은 설정을 추가해야한다.
  - 그렇지 않으면 503 에러가 발생할 것이다. 자세한 내용은 [여기](https://istio.io/help/ops/traffic-management/troubleshooting/#503-errors-after-setting-destination-rule)를 살펴보자.
```console
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutiveErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
EOF
```

2. 적용된 destination rule을 확인하자.
```console
$ kubectl get destinationrule httpbin -o yaml
```
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
  ...
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
      tcp:
        maxConnections: 1
    outlierDetection:
      baseEjectionTime: 180.000s
      consecutiveErrors: 1
      interval: 1.000s
      maxEjectionPercent: 100
```


### Client 추가하기
이제 `httpbin` 서비스로 트래픽을 보낼 client를 만들어보자. 이 섹션에서는 [fortio](https://github.com/istio/fortio)라는 istio에서 Go언어로 개발한 simple load-testing tool을 사용할 것이다.
Fortio는 nGrinder나 Siege같은 outgoing HTTP call에 대해서 conntions의 개수나, 동시 접속, delay등을 컨트롤할 수 있게 해준다. 
이 Tool을 이용해서 client가 `DestinationRule`에 설정해둔 circuit breaker policy를 _"trip"_ 할 수 있게 해준다.

1. 우선 Istio가 network interaction을 통제할 수 있게 Fortio client를 istio sidecar proxy에 주입해보자.
```console
$ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/sample-client/fortio-deploy.yaml)
```

2. client pod에 로그인후 fortio tool을 이용해서 `httpbin`을 호출해보자. `-curl` 옵션을 전달하면 단 한번만 호출하게 한다.
```console
$ FORTIO_POD=$(kubectl get pod | grep fortio | awk '{ print $1 }')
$ kubectl exec -it $FORTIO_POD  -c fortio /usr/bin/fortio -- load -curl  http://httpbin:8000/get
HTTP/1.1 200 OK
server: envoy
date: Tue, 16 Jan 2018 23:47:00 GMT
content-type: application/json
access-control-allow-origin: *
access-control-allow-credentials: true
content-length: 445
x-envoy-upstream-service-time: 36

{
  "args": {},
  "headers": {
    "Content-Length": "0",
    "Host": "httpbin:8000",
    "User-Agent": "istio/fortio-0.6.2",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "824fbd828d809bf4",
    "X-B3-Traceid": "824fbd828d809bf4",
    "X-Ot-Span-Context": "824fbd828d809bf4;824fbd828d809bf4;0000000000000000",
    "X-Request-Id": "1ad2de20-806e-9622-949a-bd1d9735a3f4"
  },
  "origin": "127.0.0.1",
  "url": "http://httpbin:8000/get"
}
```
  - 위처럼 request를 확인했다면 일단 성공한 것이다.
  
### Tripping the circuit breaker
`DestinationRule`에 `maxConnections: 1`과 `http1MaxPendingRequests: 1`로 설정했는데, 이는 만약 1개의 connection을 초과하거나 동시에 요청이 들어왔을 때에 `istio-proxy`가 circuit을 _open_ 해서 이후 요청에 대해서는 failure하게 된다.

1. `-c 2` 옵션을 이용해서 2개의 동시 connection으로 설정하여, 20개의 reqeusts를 보내보자.
```console
kubectl exec -it $FORTIO_POD -c fortio /usr/bin/fortio -- load -c 2 -qps 0 -n 20 -loglevel Warning http://httpbin:8000/get

Fortio 0.6.2 running at 0 queries per second, 2->2 procs, for 5s: http://httpbin:8000/get
Starting at max qps with 2 thread(s) [gomax 2] for exactly 20 calls (10 per thread + 0)
23:51:10 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
Ended after 106.474079ms : 20 calls. qps=187.84
Aggregated Function Time : count 20 avg 0.010215375 +/- 0.003604 min 0.005172024 max 0.019434859 sum 0.204307492
# range, mid point, percentile, count
>= 0.00517202 <= 0.006 , 0.00558601 , 5.00, 1
> 0.006 <= 0.007 , 0.0065 , 20.00, 3
> 0.007 <= 0.008 , 0.0075 , 30.00, 2
> 0.008 <= 0.009 , 0.0085 , 40.00, 2
> 0.009 <= 0.01 , 0.0095 , 60.00, 4
> 0.01 <= 0.011 , 0.0105 , 70.00, 2
> 0.011 <= 0.012 , 0.0115 , 75.00, 1
> 0.012 <= 0.014 , 0.013 , 90.00, 3
> 0.016 <= 0.018 , 0.017 , 95.00, 1
> 0.018 <= 0.0194349 , 0.0187174 , 100.00, 1
# target 50% 0.0095
# target 75% 0.012
# target 99% 0.0191479
# target 99.9% 0.0194062
Code 200 : 19 (95.0 %)
Code 503 : 1 (5.0 %)
Response Header Sizes : count 20 avg 218.85 +/- 50.21 min 0 max 231 sum 4377
Response Body/Total Sizes : count 20 avg 652.45 +/- 99.9 min 217 max 676 sum 13049
All done 20 calls (plus 0 warmup) 10.215 ms avg, 187.8 qps
```
  - 출력되는 아래 부분을 통해서 200은 19번, 503이 1번 응답했다는것을 확인할 수 있다.

2. 이제 동시 connection을 3으로 올려보자.
```console
$ kubectl exec -it $FORTIO_POD  -c fortio /usr/bin/fortio -- load -c 3 -qps 0 -n 30 -loglevel Warning http://httpbin:8000/get

Fortio 0.6.2 running at 0 queries per second, 2->2 procs, for 5s: http://httpbin:8000/get
Starting at max qps with 3 thread(s) [gomax 2] for exactly 30 calls (10 per thread + 0)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
23:51:51 W http.go:617> Parsed non ok code 503 (HTTP/1.1 503)
Ended after 71.05365ms : 30 calls. qps=422.22
Aggregated Function Time : count 30 avg 0.0053360199 +/- 0.004219 min 0.000487853 max 0.018906468 sum 0.160080597
# range, mid point, percentile, count
>= 0.000487853 <= 0.001 , 0.000743926 , 10.00, 3
> 0.001 <= 0.002 , 0.0015 , 30.00, 6
> 0.002 <= 0.003 , 0.0025 , 33.33, 1
> 0.003 <= 0.004 , 0.0035 , 40.00, 2
> 0.004 <= 0.005 , 0.0045 , 46.67, 2
> 0.005 <= 0.006 , 0.0055 , 60.00, 4
> 0.006 <= 0.007 , 0.0065 , 73.33, 4
> 0.007 <= 0.008 , 0.0075 , 80.00, 2
> 0.008 <= 0.009 , 0.0085 , 86.67, 2
> 0.009 <= 0.01 , 0.0095 , 93.33, 2
> 0.014 <= 0.016 , 0.015 , 96.67, 1
> 0.018 <= 0.0189065 , 0.0184532 , 100.00, 1
# target 50% 0.00525
# target 75% 0.00725
# target 99% 0.0186345
# target 99.9% 0.0188793
Code 200 : 19 (63.3 %)
Code 503 : 11 (36.7 %)
Response Header Sizes : count 30 avg 145.73333 +/- 110.9 min 0 max 231 sum 4372
Response Body/Total Sizes : count 30 avg 507.13333 +/- 220.8 min 217 max 676 sum 15214
All done 30 calls (plus 0 warmup) 5.336 ms avg, 422.2 qps
```
  - 이번에는 36.7% 정도에 대해서는 circuit이 open된 상태에서 응답을 받았을 것이다.  

3. `istio-proxy`의 stats을 확인해보자.
```console
$ kubectl exec -it $FORTIO_POD  -c istio-proxy  -- sh -c 'curl localhost:15000/stats' | grep httpbin | grep pending

cluster.outbound|80||httpbin.springistio.svc.cluster.local.upstream_rq_pending_active: 0
cluster.outbound|80||httpbin.springistio.svc.cluster.local.upstream_rq_pending_failure_eject: 0
cluster.outbound|80||httpbin.springistio.svc.cluster.local.upstream_rq_pending_overflow: 12
cluster.outbound|80||httpbin.springistio.svc.cluster.local.upstream_rq_pending_total: 39
```
  - `upstream_rq_pending_overflow`가 12로 나왔는데, 이 값이 실제 circuit breaking에 의해서 falgged된 요청이다.
  
### Cleanup
1. 위에서 설정한 desitination rule을 삭제하자.
```console
$ kubectl delete destinationrule httpbin
```

2. `httpbin` 서비스와 함께 추가한 client도 삭제해보자.
```console
$ kubectl delete deploy httpbin fortio-deploy
$ kubectl delete svc httpbin
```
