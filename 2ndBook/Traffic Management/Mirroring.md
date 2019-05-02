## Mirroring
"shadowing"이라고도 불리는 Traffic mirroring은 아주 적은 risk로 어떤 변경사항을 production으로 배포가능하게 해준다. 
Mirroring은 live traffic을 복사해서 mirroed serivce로 똑같이 보내준다. 이런 mirrored traffic은 주요한 service의 critical request path에 대한 처리영역 밖(out of band)에서 일어나게 된다. 


### 준비
- Istio 설치 : https://github.com/grepsean/study-istio/blob/master/setup.md
- [httpbin](https://github.com/istio/istio/tree/release-1.1/samples/httpbin)를 로깅이 가능한 두개의 버전으로 배포하자. 
  - httpbin-v1:
```console
$ cat <<EOF | istioctl kube-inject -f - | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: httpbin-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: httpbin
        version: v1
    spec:
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
        ports:
        - containerPort: 80
EOF
```
  - httpbin-v2:
```console
$ cat <<EOF | istioctl kube-inject -f - | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: httpbin-v2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: httpbin
        version: v2
    spec:
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
        ports:
        - containerPort: 80
EOF
```
  - httpbin Kubernetes service:
```console
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  labels:
    app: httpbin
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: httpbin
EOF
```

- `sleep` 서비스를 시작해서, `curl`을 이용해서 load할 수 있다.
  - sleep service: 
```console
$ cat <<EOF | istioctl kube-inject -f - | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: sleep
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
      - name: sleep
        image: tutum/curl
        command: ["/bin/sleep","infinity"]
        imagePullPolicy: IfNotPresent
EOF
```

### default routing policy를 설정하자
기본적으로 Kunernetes 두 버전의 `httpbin` 서비스에 대해서 balanced하게 로드할 수있게 해준다.
우선 이 동작을 변경해서 모든 트래픽을 `v1`으로 향하게 할것이다.

1. 모든 트래픽을 `v1`으로 향하게 하기위한 route rule을 설정하자.
  - 주의 : `mutual TLS Authentication enabled` 환경이라면, DestinationRule을 apply하기전에 TLS traffic policy에 `mode: ISTIO_MUTUAL`와 같은 설정을 추가해야한다.
  - 그렇지 않으면 503 에러가 발생할 것이다. 자세한 내용은 [여기](https://istio.io/help/ops/traffic-management/troubleshooting/#503-errors-after-setting-destination-rule)를 살펴보자.

```console
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
    - httpbin
  http:
  - route:
    - destination:
        host: httpbin
        subset: v1
      weight: 100
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF
```
  - 이제 `httpbin:v1`으로 모든 트래픽을 흘러갈 것이다. 

2. 일부 트래픽을 해당 서비스로 보내보자.
```console
$ export SLEEP_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
$ kubectl exec -it $SLEEP_POD -c sleep -- sh -c 'curl  http://httpbin:8000/headers' | python -m json.tool
{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin:8000",
    "User-Agent": "curl/7.35.0",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "eca3d7ed8f2e6a0a",
    "X-B3-Traceid": "eca3d7ed8f2e6a0a",
    "X-Ot-Span-Context": "eca3d7ed8f2e6a0a;eca3d7ed8f2e6a0a;0000000000000000"
  }
}
```

3. `httpbin` pod에 있는 `v1`과 `v2`의 로그를 확인해보자.
```console
$ export V1_POD=$(kubectl get pod -l app=httpbin,version=v1 -o jsonpath={.items..metadata.name})
$ kubectl logs -f $V1_POD -c httpbin

127.0.0.1 - - [07/Mar/2018:19:02:43 +0000] "GET /headers HTTP/1.1" 200 321 "-" "curl/7.35.0"
```
  - `v1`은 로그를 확인할 수 있지만,
```console
export V2_POD=$(kubectl get pod -l app=httpbin,version=v2 -o jsonpath={.items..metadata.name})
kubectl logs -f $V2_POD -c httpbin

<none>
```
  - `v2`는 로그가 <none> 없다.
  

### `v2`로 미러링해보자
1. `v2`로 미러링 트래픽을 보내게 설정해보자.
```console
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
    - httpbin
  http:
  - route:
    - destination:
        host: httpbin
        subset: v1
      weight: 100
    mirror:
      host: httpbin
      subset: v2
EOF
```
  - 위 처럼 spec.http.route.mirror 설정을 통해서, `httpbin` 서비스의 `v2`라는 pod으로 mirrored 트래픽을 전송할 수 있게 된다.
  - 이러한 mirrored 트래픽의 경우에는, `Host`/`Authority` header에 `-shodow`라는게 추가된다.
    - 예를 들면, `cluster-1` => `cluster-1-shadow`처럼 되는 것이다.
    - 또한 _"fire and forget"_ 라는 전략에 따라서 mirrored 요청에 대한 response는 무시된다.

2. 트래픽을 전송하자.
```console
$ kubectl exec -it $SLEEP_POD -c sleep -- sh -c 'curl  http://httpbin:8000/headers' | python -m json.tool
```
  - 위처럼 실행하면, `v1`과 `v2` 모두에서 access log를 확인할 수 있을 것이다.
```console
$ kubectl logs -f $V1_POD -c httpbin
127.0.0.1 - - [07/Mar/2018:19:02:43 +0000] "GET /headers HTTP/1.1" 200 321 "-" "curl/7.35.0"
127.0.0.1 - - [07/Mar/2018:19:26:44 +0000] "GET /headers HTTP/1.1" 200 321 "-" "curl/7.35.0"
```
```console
$ kubectl logs -f $V2_POD -c httpbin
127.0.0.1 - - [07/Mar/2018:19:26:44 +0000] "GET /headers HTTP/1.1" 200 361 "-" "curl/7.35.0"
```

3. 만약 트래픽 내부에 대해서 살펴보려면, 아래와 같은 커맨드를 실행시켜보자.
```console
$ export SLEEP_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
$ export V1_POD_IP=$(kubectl get pod -l app=httpbin -l version=v1 -o jsonpath={.items..status.podIP})
$ export V2_POD_IP=$(kubectl get pod -l app=httpbin -l version=v2 -o jsonpath={.items..status.podIP})
$ kubectl exec -it $SLEEP_POD -c istio-proxy -- sudo tcpdump -A -s 0 host $V1_POD_IP or host $V2_POD_IP
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
05:47:50.159513 IP sleep-7b9f8bfcd-2djx5.38836 > 10-233-75-11.httpbin.default.svc.cluster.local.80: Flags [P.], seq 4039989036:4039989832, ack 3139734980, win 254, options [nop,nop,TS val 77427918 ecr 76730809], length 796: HTTP: GET /headers HTTP/1.1
E..P2.X.X.X.
.K.
.K....P..W,.$.......+.....
..t.....GET /headers HTTP/1.1
host: httpbin:8000
user-agent: curl/7.35.0
accept: */*
x-forwarded-proto: http
x-request-id: 571c0fd6-98d4-4c93-af79-6a2fe2945847
x-envoy-decorator-operation: httpbin.default.svc.cluster.local:8000/*
x-b3-traceid: 82f3e0a76dcebca2
x-b3-spanid: 82f3e0a76dcebca2
x-b3-sampled: 0
x-istio-attributes: Cj8KGGRlc3RpbmF0aW9uLnNlcnZpY2UuaG9zdBIjEiFodHRwYmluLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwKPQoXZGVzdGluYXRpb24uc2VydmljZS51aWQSIhIgaXN0aW86Ly9kZWZhdWx0L3NlcnZpY2VzL2h0dHBiaW4KKgodZGVzdGluYXRpb24uc2VydmljZS5uYW1lc3BhY2USCRIHZGVmYXVsdAolChhkZXN0aW5hdGlvbi5zZXJ2aWNlLm5hbWUSCRIHaHR0cGJpbgo6Cgpzb3VyY2UudWlkEiwSKmt1YmVybmV0ZXM6Ly9zbGVlcC03YjlmOGJmY2QtMmRqeDUuZGVmYXVsdAo6ChNkZXN0aW5hdGlvbi5zZXJ2aWNlEiMSIWh0dHBiaW4uZGVmYXVsdC5zdmMuY2x1c3Rlci5sb2NhbA==
content-length: 0


05:47:50.159609 IP sleep-7b9f8bfcd-2djx5.49560 > 10-233-71-7.httpbin.default.svc.cluster.local.80: Flags [P.], seq 296287713:296288571, ack 4029574162, win 254, options [nop,nop,TS val 77427918 ecr 76732809], length 858: HTTP: GET /headers HTTP/1.1
E.....X.X...
.K.
.G....P......l......e.....
..t.....GET /headers HTTP/1.1
host: httpbin-shadow:8000
user-agent: curl/7.35.0
accept: */*
x-forwarded-proto: http
x-request-id: 571c0fd6-98d4-4c93-af79-6a2fe2945847
x-envoy-decorator-operation: httpbin.default.svc.cluster.local:8000/*
x-b3-traceid: 82f3e0a76dcebca2
x-b3-spanid: 82f3e0a76dcebca2
x-b3-sampled: 0
x-istio-attributes: Cj8KGGRlc3RpbmF0aW9uLnNlcnZpY2UuaG9zdBIjEiFodHRwYmluLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwKPQoXZGVzdGluYXRpb24uc2VydmljZS51aWQSIhIgaXN0aW86Ly9kZWZhdWx0L3NlcnZpY2VzL2h0dHBiaW4KKgodZGVzdGluYXRpb24uc2VydmljZS5uYW1lc3BhY2USCRIHZGVmYXVsdAolChhkZXN0aW5hdGlvbi5zZXJ2aWNlLm5hbWUSCRIHaHR0cGJpbgo6Cgpzb3VyY2UudWlkEiwSKmt1YmVybmV0ZXM6Ly9zbGVlcC03YjlmOGJmY2QtMmRqeDUuZGVmYXVsdAo6ChNkZXN0aW5hdGlvbi5zZXJ2aWNlEiMSIWh0dHBiaW4uZGVmYXVsdC5zdmMuY2x1c3Rlci5sb2NhbA==
x-envoy-internal: true
x-forwarded-for: 10.233.75.12
content-length: 0


05:47:50.166734 IP 10-233-75-11.httpbin.default.svc.cluster.local.80 > sleep-7b9f8bfcd-2djx5.38836: Flags [P.], seq 1:472, ack 796, win 276, options [nop,nop,TS val 77427925 ecr 77427918], length 471: HTTP: HTTP/1.1 200 OK
E....3X.?...
.K.
.K..P...$....ZH...........
..t...t.HTTP/1.1 200 OK
server: envoy
date: Fri, 15 Feb 2019 05:47:50 GMT
content-type: application/json
content-length: 241
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 3

{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin:8000",
    "User-Agent": "curl/7.35.0",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "82f3e0a76dcebca2",
    "X-B3-Traceid": "82f3e0a76dcebca2"
  }
}

05:47:50.166789 IP sleep-7b9f8bfcd-2djx5.38836 > 10-233-75-11.httpbin.default.svc.cluster.local.80: Flags [.], ack 472, win 262, options [nop,nop,TS val 77427925 ecr 77427925], length 0
E..42.X.X.\.
.K.
.K....P..ZH.$.............
..t...t.
05:47:50.167234 IP 10-233-71-7.httpbin.default.svc.cluster.local.80 > sleep-7b9f8bfcd-2djx5.49560: Flags [P.], seq 1:512, ack 858, win 280, options [nop,nop,TS val 77429926 ecr 77427918], length 511: HTTP: HTTP/1.1 200 OK
E..3..X.>...
.G.
.K..P....l....;...........
..|...t.HTTP/1.1 200 OK
server: envoy
date: Fri, 15 Feb 2019 05:47:49 GMT
content-type: application/json
content-length: 281
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 3

{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin-shadow:8000",
    "User-Agent": "curl/7.35.0",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "82f3e0a76dcebca2",
    "X-B3-Traceid": "82f3e0a76dcebca2",
    "X-Envoy-Internal": "true"
  }
}

05:47:50.167253 IP sleep-7b9f8bfcd-2djx5.49560 > 10-233-71-7.httpbin.default.svc.cluster.local.80: Flags [.], ack 512, win 262, options [nop,nop,TS val 77427926 ecr 77429926], length 0
E..4..X.X...
.K.
.G....P...;..n............
..t...|.
```
  - `istio-proxy` 컨테이너에서 tcpdump를 이용해서 `v1`, `v2`의 패킷을 찍어본 것이다.
  
### Cleanup
1. 위에서 생성한 rules를 삭제하자.
```console
$ kubectl delete virtualservice httpbin
$ kubectl delete destinationrule httpbin
```

2. `httpbin` 서비스와 client를 삭제하자.
```console
$ kubectl delete deploy httpbin-v1 httpbin-v2 sleep
$ kubectl delete svc httpbin
```
