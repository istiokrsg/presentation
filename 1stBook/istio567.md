class: center, middle

# Introducing Istio Service Mesh for Microservices

### 5,6,7 장

## 윤평호 2019-04-16
---
class: center, middle
# 목차

1. Chaos Testing
2. Observability
3. Security(ACL)
4. More things

---
# Chaos Testing

![drawing](chaos.png "Chaos")

---

## 카오스 엔지니어링
1. (실 서비스에) 장애를 주입하여
2. (출시 전) 테스트에서 드러나지 않는 아키텍처상의 문제를 직접 드러내는
> 프로덕션 서비스의 각종 장애 조건을 견딜 수 있는 시스템 신뢰성을 확보하기 위해 분산 시스템을 실험하고 배우는 분야

- https://www.slideshare.net/Channy/chaos-engineering-in-action-for-kubernates
- https://www.slideshare.net/AmazonWebServices/chaos-engineering-and-scalability-at-audiblecom-arc308-aws-reinvent-2018
- https://www.slideshare.net/AmazonWebServices/globalizing-player-accounts-at-riot-games-while-maintaining-availability-arc314-aws-reinvent-2018

---

## 장애 주입(Failure Injection)
### 작게 시작해서 점진적으로 신뢰성 구축
- 애플리케이션 부하 테스트
- 호스트 서버 이슈
- 데이터베이스 문제
- 자원 부족(CPU, memory, disk, …)
- 네트워크 부족(종속성, 지연, …)
- 서비스 부족

Q. Istio 에서 할 수 있는 것은?
- HTTP(S) 관련 장애 주입
---
## HTTP error 장애 주입

istiofiles/virtual-service-recommendation-503.yml .red[*]
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: recommendation
spec:
  hosts:
  - recommendation
  http:
  - fault:
      abort:
        httpStatus: 503
        percent: 50
    route:
    - destination:
        host: recommendation
        subset: app-recommendation
```

.footnote[참고: .red[*] 
책과 다릅니다. RouteRule --> [VirtualService](https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/#VirtualService
)(v0.7)

[HTTP status code](https://www.tutorialspoint.com/http/http_status_codes.htm),
[fault.abort](https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/#HTTPFaultInjection-Abort)
]
---
## HTTP delay 장애 주입

istiofiles/virtual-service-recommendation-delay.yml
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: recommendation
spec:
  hosts:
  - recommendation
  http:
  - fault:
      delay:
        fixedDelay: 7.000s
        percent: 50
    route:
    - destination:
        host: recommendation
        subset: app-recommendation
```
.footnote[참고: 
[fault.delay](https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/#HTTPFaultInjection-Delay)
]
---

# Observability
![drawing](visibility.png "visibility")


---
## Monolith vs Micro service architecture
![drawing](https://19yw4b240vb03ws8qm25h366-wpengine.netdna-ssl.com/wp-content/uploads/Monolith-vs-Microservices-1024x568.png "Monolith vs MSA")


---
## MSA 의 구성 요소 간에 관계
![drawing](https://graphaware.com/assets/whyyoushouldstart/LeanDependencyManagement.png "Service relationships")

Traffic Pattern!
이것들이 모이면...
---
## MSA 의 미래?!
![drawing](https://graphaware.com/assets/whyyoushouldstart/MicroservicesArchitecture.png "MSA future?")

Q. Istio 에서 할 수 있는 것은?
- 경로에 따른 서비스 간에 관계성(종족성)
- 서비스 흐름 정상 모니터링
- 로드밸런싱, 풀링, 서킷 브레이크, 배포
- ...
---
## Span, Trace

from Jaeger 
### Span
- "작업명칭, 시작시간, 소요시간을 가진 작업의 논리 단위. 중첩 가능하고 인과 관계를 모델링 할 수 있음. i.e. RPC call"

### Trace
- "시스템을 통한 데이터/실행 경로로 span의 방향성있는 비순환 그래프 형식으로 표현"

https://istio.io/docs/reference/config/policy-and-telemetry/templates/tracespan/
---
## 서비스 추적을 위해 필요한 HTTP 헤더(전파해야함)

```java
...
public class HttpHeaderForwarderHandlerInterceptor extends HandlerInterceptorAdapter {

  private static final Set<String> FORWARDED_HEADER_NAMES = ImmutableSet.of(
    "x-request-id",
    "x-b3-traceid",
    "x-b3-spanid",
    "x-b3-parentspanid",
    "x-b3-sampled",
    "x-b3-flags",
    "x-ot-span-context",
    "user-agent"
  );

  @Override
  public boolean preHandle( ... ) {
  Map<String, List<String>> headerMap = Collections.list(request.getHeaderNames())
    .stream()
    .map(String::toLowerCase)
    .filter(FORWARDED_HEADER_NAMES::contains)
    .collect(Collectors.toMap(
      Function.identity(),
      h -> Collections.list(request.getHeaders(h))
    ));
  HEADERS_THREAD_LOCAL.set(headerMap);
  return super.preHandle(request, response, handler);
  }
```

.footnote[ HTTP Tracing: https://zipkin.io/pages/instrumenting.html]
---

## Metrics

Istio는 기본적으로 텔레메트리 데이터를 수집합니다.

![drawing](tracing.png "\?")
---

## Metrics
![drawing](tracing-grafana.png "\?")


http://grafana-istio-system.192.168.99.101.nip.io/dashboard/db/istio-dashboard?var-source=All
---
![drawing](tracing-grafana-response.png "\?")

---

# Security(ACL)

![drawing](acl.png "acl")

https://istio.io/docs/tasks/policy-enforcement/denial-and-list/
---
## 블랙 리스트

Mixer selector를 이용하여 조건별로(특정 호출 경로) 명시적으로 거부

istiofiles/acl-blacklist.yml (customer -> preference 일 때에 거부(403 Forbidden))
```yaml
apiVersion: "config.istio.io/v1alpha2"
kind: denier
metadata:
  name: denycustomerhandler
spec:
  status:
    code: 7
    message: Not allowed
---
apiVersion: "config.istio.io/v1alpha2"
kind: checknothing
metadata:
  name: denycustomerrequests
spec:
---
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: denycustomer
spec:
  match: destination.labels["app"] == "preference" && source.labels["app"]=="customer"
  actions:
  - handler: denycustomerhandler.denier
    instances: [ denycustomerrequests.checknothing ]
```
---
### Mixer 제어 요소

#### denier:
- https://istio.io/docs/reference/config/policy-and-telemetry/adapters/denier/
- https://istio.io/blog/2017/adapter-model/
- https://github.com/istio/istio/wiki/Mixer-Compiled-In-Adapter-Dev-Guide

#### checknothing:
- https://istio.io/docs/reference/config/policy-and-telemetry/templates/checknothing/

#### rule: 
- https://istio.io/docs/reference/config/policy-and-telemetry/istio.policy.v1beta1/#Rule
- https://istio.io/docs/reference/config/policy-and-telemetry/istio.policy.v1beta1/#Action
- https://istio.io/docs/reference/config/policy-and-telemetry/istio.policy.v1beta1/#Handler

---
## 화이트 리스트

승인된 호출 경로를 제외하고는 모든 규칙을 거부

istiofiles/acl-whitelist.yml(recommendation -> preferences)
```yaml
apiVersion: "config.istio.io/v1alpha2"
kind: listchecker
metadata:
  name: preferencewhitelist
spec:
  overrides: ["recommendation"]
  blacklist: false
---
apiVersion: "config.istio.io/v1alpha2"
kind: listentry
metadata:
  name: preferencesource
spec:
  value: source.labels["app"]
---
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: checkfromcustomer
spec:
  match: destination.labels["app"] == "preference"
  actions:
  - handler: preferencewhitelist.listchecker
    instances:
    - preferencesource.listentry
```
---
### Mixer 제어요소

#### listentry:
- https://istio.io/docs/reference/config/policy-and-telemetry/templates/listentry/

#### listchecker:
- https://istio.io/docs/reference/config/policy-and-telemetry/adapters/list/

#### rule: 
- https://istio.io/docs/reference/config/policy-and-telemetry/istio.policy.v1beta1/#Rule
- https://istio.io/docs/reference/config/policy-and-telemetry/istio.policy.v1beta1/#Action
- https://istio.io/docs/reference/config/policy-and-telemetry/istio.policy.v1beta1/#Handler

---
### IP CIDR 로 구분 가능

```
apiVersion: config.istio.io/v1alpha2
kind: listchecker
metadata:
  name: whitelistip
spec:
  # providerUrl: ordinarily black and white lists are maintained
  # externally and fetched asynchronously using the providerUrl.
  overrides: ["10.57.0.0/16"]  # overrides provide a static list
  blacklist: false
  entryType: IP_ADDRESSES
```
---
# Things left

## Traffic flow
- [https://bit.ly/2Pcuqyg](https://medium.com/devopslinks/understanding-how-envoy-sidecar-intercept-and-route-traffic-in-istio-service-mesh-20fea2a78833)
- [Debugging Envoy and Pilot](https://istio.io/help/ops/traffic-management/proxy-cmd)
- https://www.youtube.com/watch?v=I9ZskIT-jxg
- https://mt165.co.uk/speech/life-of-a-packet-istio-cloud-native/

---
# Hands-on

### fault injection
- https://learn.openshift.com/servicemesh/6-fault-injection
- https://www.katacoda.com/courses/istio/increasing-reliability

### monitoring
- https://learn.openshift.com/servicemesh/3-monitoring-tracing
- https://www.katacoda.com/courses/istio/observing-microservices

### access control
- https://learn.openshift.com/servicemesh/5-advanced-routerules

---
# fault injection

```bash
#!/bin/bash
cd /root/
launch.sh>&2
if [[ ! -d "/root/istio-1.0.0" ]]; then
  echo "Downloading Istio... this may take a couple of moments">&2
  curl -s -L https://git.io/getLatestIstio | ISTIO_VERSION=1.0.0 sh -
  echo "Download completed. Configuring Kubernetes.">&2
else
  echo "Istio already exists">&2
fi
export PATH="$PATH:/root/istio-1.0.0/bin";
cd /root/istio-1.0.0
kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml -n istio-system
kubectl apply -f install/kubernetes/istio-demo-auth.yaml
kubectl apply -f /root/katacoda.yaml
kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v2.yaml

gurl=https://gist.githubusercontent.com/BenHall/e5fa7eed7e1b0bc21ac0abbd431efc37/raw
curl -s -L -o samples/bookinfo/networking/virtual-service-reviews-v1.yaml \
$gurl/bed904fb75516e8e0dd87c86c5b274fb4c5e372c/virtual-service-reviews-v1.yaml
curl -s -L -o samples/bookinfo/networking/virtual-service-reviews-v2.yaml \
$gurl/cf8426de87eb29716f41070bb619c6f4fbd759af/virtual-service-reviews-v2.yaml
curl -s -L -o samples/bookinfo/networking/virtual-service-reviews-chrome-v2.yaml \
$gurl/c3c3a25721af90e180c1b02c618d6c8b660402d7/virtual-service-reviews-chrome-v2.yaml
```
---
# katacode tip

https://wsend.net/

## install
```bash
wget https://wsend.net/wsend
chmod +x wsend
```

## usage
```bash
user@system:~$ wsend README.md
https://wsend.net/73efe0fd8bb12baac9d023708a9db634/README.md
```