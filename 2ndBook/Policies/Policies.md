class: center, middle

# Istio Policy

## https://istio.io/docs/tasks/policy-enforcement/ 관련 정리

## 윤평호 2019.06.25

---
## 목차

- 정책으로 할 수 있는 작업
- 강제로 정책 적용 하기
- 정책의 구성과 작동
- 리퀘스트 제한하기
- 헤더 제한과 라우팅
- 서비스 거부 및 화이트 블랙박스 리스팅
- 부록

---
## 정책으로 할 수 있는 작업

- 라우팅 정책
  - 로드 밸런싱
  - 트래픽 조정
  - 리퀘스트 타임아웃, 재시도, 장애 주입
- 쿼터 정책
- 모니터링 정책
  - 메트릭, 로깅, 트레이싱
- 보안 정책
  - 서비스 간에 mTLS 인증
  - 단순 인증, 거부, 화이트/블랙 리스트, 표현식을 통한 어트리뷰트 기반 접근 제어(ABAC)
- 인증 정책
  - 서비스별 mTLS 사용 여부
  - 사용자 인증
- 인가 정책
  - 역할 기반 접근 제어(RBAC)
  - Open Policy Agent
  - ...

**그러나, istio 문서에서는 .red[아답터 관련 부분만 정책]으로 가정** :)

---
## 강제로 정책 적용 하기
기본적으로 정책은 강제로 적용하지 않는다. 느려서?! :(
- 이후 테스트를 위해 정책을 강제 적용하도록 한다.

```shell
# 확인
$ kubectl -n istio-system \
get cm istio -o jsonpath="{@.data.mesh}" | \
grep disablePolicyChecks
disablePolicyChecks: true

# 설정하기
$ helm template install/kubernetes/helm/istio \
--namespace=istio-system \
-x templates/configmap.yaml \
--set global.disablePolicyChecks=false | \
kubectl -n istio-system replace -f -

# 재확인
$ kubectl -n istio-system \
get cm istio -o jsonpath="{@.data.mesh}" | \
grep disablePolicyChecks
disablePolicyChecks: true

# helm의 경우 설치/업그레이드시에 --set global.disablePolicyChecks=true 추가로
```

---
## 정책의 구성과 작동: 정책 구성 .red[*]

정책 = 룰 + 인스턴스(=템플릿) + 핸들러(=아답터)
- 핸들러: 아답터 + 아답터 환경 설정
- 인스턴스: 보통, 프로토버퍼를 컴파일한 템플릿(compiledTemplate)을 이용
  - 아답터에 전달할 데이터(애트리뷰트 묶음)
- 룰: 어떤 조건으로, 어떤 데이터를 핸들러에게 넘겨줄 것인가
  - match: 적용 조건
  - handler: 조건 맞으면 호출할 아답터
  - instance: 아답터에 어떤 데이터를 전달해주나

.footnote[
.red[*] https://istio.io/blog/2017/adapter-model/, [Mixer FAQ](https://istio.io/faq/mixer/)
]

---
## 정책의 구성과 작동: 아키텍처 다시 보기

![drawing](https://istio.io/blog/2019/data-plane-setup/arch-2.svg "istio archtecture")

---
## 정책의 구성과 작동: 동작 방식

![drawing](https://venilnoronha.io/assets/images/2018-10-07-set-sail-a-production-ready-istio-adapter/architecture.svg "adapter flow")

- 아키텍처와 동작 방식을 볼수록 밀려오는 물음표 _**요청마다 이렇게?**_
  - 당신이 뭐라 말할지 개발자들도 알고 있다. .red[*]
     - SPOF Myth!? 알아도, 개선해도, 프로덕션에서는!?

> 노력하고 있으니, 쓰는 사람이 잘 써야지? :) .red[**]

.footnote[
.red[*] https://istio.io/blog/2017/mixer-spof-myth/

.red[**] https://github.com/istio/istio/wiki/Istio-Performance
]

---
## 정책의 구성과 작동: Mixer가 하는 일?
정책 확장은 Mixer를 통해 이루어짐
- 핵심 기능: 조건 체크, 쿼터 체크, 텔레메트리 리포팅
- 외부 모듈 플러그인하여 기능을 확장할 수 있음

![drawing](https://camo.githubusercontent.com/5e0fe3547b9dfc61d4fd0572aef30b67c0a1b312/68747470733a2f2f63646e2e7261776769742e636f6d2f77696b692f697374696f2f697374696f2f696d616765732f6f70657261746f7225323074656d706c617465253230616461707465722532306465762e7376673f73616e6974697a653d74727565 "mixer architecture")

---
## 리퀘스트 제한하기: 사전 조건 확인

기본 샘플인 bookinfo에서 다양한 서비스 버전이 있어 임의로 라우팅한다.

|버전|ratings을 호출 여부|비고                   |
|:--:|:-----------------:|:---------------------:|
|v1  |X                  |                       |
|v2  |O                  |rating을 검은 별로     |
|v3  |O                  |rating을 빨간 별로     |

그래서 고정으로 한 곳만 호출하는 서비스 버전 v1을 사용한다.
```shell
$ kubectl apply -f \
  samples/bookinfo/networking/virtual-service-all-v1.yaml
```

---
## 리퀘스트 제한하기: 어떻게 요청을 제한하나?
- 클라이언트 IP를 기반으로 요청을 제한한다.
  - **X-Forwarded-For** HTTP 헤더값을 이용

- 구성할 필드는?
  - QuotaSpec .red[*]
  - QuotaSpecBinding .red[*]
  - Rule
     - Handler(mem-quota vs redis-quota)
     - Instance(compiled template)

.footnote[
[*] 룰에 관계없이 istio는 각 서비스의 리퀘스트 회수를 텔레메트리 값으로 수집한다.
]
---
## 리퀘스트 제한하기: 리퀘스트 쿼터 설정
- 이름, 리퀘스트 쿼터, 어느 서비스에 리퀘스트를 측정할 것인가?

```yaml
kind: QuotaSpec # 측정값
metadata:
  name: request-count # 측정값 이름
...
spec:
  rules:
  - quotas:
    - charge: 1
      quota: requestcountquota
```
```yaml
kind: QuotaSpecBinding # 대상
...
spec:
  quotaSpecs:
  - name: request-count # 측정값 이름 참조
  services:
  - name: productpage
    namespace: default
    #- service: '*'  # 주석 없애면 모든 서비스를 다 측정한다
```

---
## 리퀘스트 제한하기: 인스턴스 설정
```
kind: instance
metadata:
  name: requestcountquota
...
# https://istio.io/docs/reference/config/policy-and-telemetry/expression-language 참고
spec:
  compiledTemplate: quota
  params:
    dimensions:
      source: request.headers["x-forwarded-for"] | "unknown"
      destination: >-
        destination.labels["app"] | destination.service.name | "unknown"
      destinationVersion: >-
        destination.labels["version"] | "unknown"
```

---
## 리퀘스트 제한하기: 룰 설정(memquota)
```yaml
kind: rule
metadata:
  name: quota
...
spec:
  # 로그인 안된 경우에만 측정한다면? 아래 주석을 풀자
  # match: match(request.headers["cookie"], "user=*") == false
  actions:
  - handler: quotahandler
    instances:
    - requestcountquota
```

---
## 리퀘스트 제한하기: memquota 핸들러 설정
```yaml
kind: handler
metadata:
  name: quotahandler
  ...
spec:
  compiledAdapter: memquota
  params:
    quotas:
    - name: requestcountquota.instance.istio-system
      maxAmount: 500
      validDuration: 1s
      overrides:
      - dimensions:
          destination: reviews
        maxAmount: 1     # 1/5 qps
        validDuration: 5s
      - dimensions:
          destination: productpage
          source: "10.28.11.20"
        maxAmount: 500   # 500/1 qps
        validDuration: 1s
      - dimensions:
          destination: productpage
        maxAmount: 2     #  2/5 qps
        validDuration: 5s
```

---
## 룰리퀘스트 제한하기: 설정(redisquota)
```yaml
kind: rule
metadata:
  name: quota
...
spec:
  # 로그인 안된 경우에만 측정한다면? 아래 주석을 풀자
  # match: match(request.headers["cookie"], "user=*") == false
  actions:
  - handler: redishandler # memquota 룰과 요기만 다르다
    instances:
    - requestcountquota
```

---
## 리퀘스트 제한하기: redisquota 핸들러 설정
```yaml
kind: handler
metadata:
  name: redishandler
 ...
spec:
  compiledAdapter: redisquota
  params:
    redisServerUrl: redis-release-master:6379 # redis host:port
    connectionPoolSize: 10
    quotas:
    - name: requestcountquota.instance.istio-system
      maxAmount: 500
      validDuration: 1s
      bucketDuration: 500ms
      rateLimitAlgorithm: ROLLING_WINDOW #
      overrides:
      - dimensions:
          destination: reviews
        maxAmount: 1
      - dimensions:
          destination: productpage
          source: "10.28.11.20"
        maxAmount: 500
      - dimensions:
          destination: productpage
        maxAmount: 2
```

---
## 리퀘스트 제한하기: 적용 및 확인

```shell
# 적용하고
$ kubectl apply -f \
  samples/bookinfo/policy/mixer-rule-productpage-ratelimit.yaml

# 적용 내역 확인
$ kubectl -n istio-system get rule quota -o yaml
$ kubectl -n istio-system get QuotaSpec request-count -o yaml
$ kubectl -n istio-system get QuotaSpecBinding request-count -o yaml
```

리퀘스트 제한을 넘게 요청해보면, HTTP 429 오류 발생

> `RESOURCE_EXHAUSTED:Quota is exhausted for: requestcount.`

---
# HTTP 헤더와 라우팅 변경

- HTTP 헤더와 라우팅 변경 테스트를 위해 httpbin 서비스 라우팅 적용 .red[*]

```yaml
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "*"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /headers
    - uri:
        prefix: /status
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
```

.footnote[
  .red[*] 앞으로 .yaml 파일은 `kubectl apply -f <yaml 파일>` 형식으로 적용
]
---
## HTTP 헤더와 라우팅 변경: 커스텀 아답터 설치

```shell
# keyval 아답터 파드 생성
# 이름은 keyval이고 mixer와 grpc 통신을 위해 9070 포트 노출한다
$ kubectl run keyval \
 --image=gcr.io/istio-testing/keyval:release-1.1 \
 --namespace istio-system \
 --port 9070 \
 --expose

$ kubectl apply -f samples/httpbin/policy/keyval-template.yaml
  # 아답터 템플릿 + 핸들러 형식 설정. 아래 프로토버퍼 정의를 컴파일한 바이너리
  # template.proto + template_handler_service.proto + template_handler.gen.go

$ kubectl apply -f samples/httpbin/policy/keyval.yaml
  # 아답터 환경 구성 형식 설정(config.proto를 컴파일한 바이너리)
```

- [samples/httpbin/policy/keyval-template.yaml](https://raw.githubusercontent.com/istio/istio/release-1.1/samples/httpbin/policy/keyval-template.yaml)
- [samples/httpbin/policy/keyval.yaml](https://raw.githubusercontent.com/istio/istio/release-1.1/samples/httpbin/policy/keyval.yaml)

---
## HTTP 헤더와 라우팅 변경: keyval 핸들러 설정
```yaml
kind: handler
metadata:
  name: keyval
  namespace: istio-system
spec:
  adapter: keyval
  connection:
    address: keyval:9070
  params:
    table:
      jason: admin
```

위 구성은 keyval.yaml에서 정의한 [config.proto](https://github.com/istio/istio/blob/master/mixer/test/keyval/config.proto) `아답터 환경 구성 형식`을 따른다.
```proto3
message Params {
  // Lookup table
  map<string, string> table = 1;
}
```

---
## HTTP 헤더와 라우팅 변경: keyval 인스턴스 설정
```yaml
apiVersion: config.istio.io/v1alpha2
kind: instance
metadata:
  name: keyval
  namespace: istio-system
spec:
  template: keyval
  params:
    key: request.headers["user"] | ""
```

---
## HTTP 헤더와 라우팅 변경: [keyval 아답터](https://github.com/istio/istio/blob/master/mixer/test/keyval/adapter.go) 살펴보기
```go
func (Keyval) HandleKeyval
 (ctx context.Context, req *HandleKeyvalRequest) (*HandleKeyvalResponse, error) {
  params := &Params{}
  if err := params.Unmarshal(req.AdapterConfig.Value); err != nil {
    return nil, err
  }
  key := req.Instance.Key
  value, ok := params.Table[key]
  if ok {
    return &HandleKeyvalResponse{
      Result: &v1beta1.CheckResult{ValidDuration: 5 * time.Second},
      Output: &OutputMsg{Value: value},
    }, nil
  }
  return &HandleKeyvalResponse{
    Result: &v1beta1.CheckResult{
      Status: rpc.Status{
        Code: int32(rpc.NOT_FOUND),
        Details: []*types.Any{
          status.PackErrorDetail(&policy.DirectHttpResponse{
            Body:
              fmt.Sprintf("<error_detail>key %q not found</error_detail>", key),
          })
        },},},}, nil
}
```

---
## HTTP 헤더와 라우팅 변경: keyval 아답터 적용전

```shell
$ curl http://$INGRESS_HOST:$INGRESS_PORT/headers
{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    ...
    "X-Envoy-Internal": "true"
  }
}
```

---
## HTTP 헤더와 라우팅 변경: keyval 아답터 적용

```yaml
kind: rule
metadata:
  name: keyval
  namespace: istio-system
spec:
  actions:
  - handler: keyval.istio-system
    instances: [ keyval ]
    name: x
  requestHeaderOperations:
  - name: user-group
    values: [ x.output.value ]
```

---
## HTTP 헤더와 라우팅 변경: keyval 아답터 적용후

```shell
$ curl -Huser:jason http://$INGRESS_HOST:$INGRESS_PORT/headers
{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "User": "jason",
    "User-Agent": "curl/7.58.0",
    "User-Group": "admin",
    ...
    "X-Envoy-Internal": "true"
  }
}
```

> `-Huser:jason`헤더를 추가 요청하니 응답에 `User-Group: admin`헤더 추가됨

---
## HTTP 헤더와 라우팅 변경: 라우팅 변경 적용

```yaml
kind: rule
metadata:
  name: keyval
  namespace: istio-system
spec:
  match: source.labels["istio"] == "ingressgateway"
  actions:
  - handler: keyval.istio-system
    instances: [ keyval ]
  requestHeaderOperations:
  - name: :path  # URI 를 칭함, 소문자로
    values: [ '"/status/418"' ] # URI를 /status/418로 변경
```
> requestHeaderOperations은 keyval과 전혀 상관없는 rule 기능이다.

> `:path`은 istio의 표현식이나 어트리뷰트 용어가 아니니, 소스를 참조 \*

.footnote[
\* https://github.com/istio/api/blob/8685353777046fb0f063bc9f0a6303edcfbcc4f1/mixer/v1/mixer.pb.go#L445 참조

```go
// Operation on HTTP headers to replace, append, or remove a header. Header
// names are normalized to lower-case with dashes, e.g.  "x-request-id".
// Pseudo-headers ":path", ":authority", and ":method" are supported to modify
// the request headers.
```
]

---
## HTTP 헤더와 라우팅 변경: 라우팅 변경 적용후

```shell
$ curl -Huser:jason -I http://$INGRESS_HOST:$INGRESS_PORT/headers
HTTP/1.1 418 Unknown
server: istio-envoy
...
```

> `/headers`를 요청했는데, `/status/418`로 라우팅되어 응답함

---
## 서비스 거부와 화이트/블랙 리스트

서비스별, 아이피별로 서비스를 거부하는 방법을 논한다.
물론 아답터를 사용한다.

---
### 서비스 거부와 화이트/블랙 리스트: 무조건 거부
```yaml
kind: handler
metadata:
  name: denyreviewsv3handler
spec:
  compiledAdapter: denier # 내장 아답터
  params:
    status:
      code: 7
      message: Not allowed
---
kind: instance
metadata:
  name: denyreviewsv3request
spec:
  compiledTemplate: checknothing
---
kind: rule
metadata:
  name: denyreviewsv3
spec:
  match: |-
    destination.labels["app"] == "ratings" && source.labels["app"]=="reviews" &&
    source.labels["version"] == "v3"
  actions:
  - handler: denyreviewsv3handler
    instances: [ denyreviewsv3request ]
```

---
### 서비스 거부와 화이트/블랙 리스트: 화이트 리스트
```yaml
kind: handler
metadata:
  name: whitelist
spec:
  compiledAdapter: listchecker # 내장 아답터
  params:
    # providerUrl: 보통 화이트/블랙 리스트는 외부(providerUrl)에서 비동기적으로 읽어온다.
    overrides: ["v1", "v2"]
    blacklist: false
---
kind: instance
metadata:
  name: appversion
spec:
  compiledTemplate: listentry
  params:
    value: source.labels["version"]
---
kind: rule
metadata:
  name: checkversion
spec:
  match: destination.labels["app"] == "ratings"
  actions:
  - handler: whitelist
    instances: [ appversion ]
```

---
### 서비스 거부와 화이트/블랙 리스트: IP 기반 화이트/블랙 리스트

```yaml
kind: handler
metadata:
  name: whitelistip
spec:
  compiledAdapter: listchecker
  params:
    overrides: ["10.57.0.0/16"] # 허용 IP 주소
    blacklist: false
    entryType: IP_ADDRESSES # IP 주소 형식 알림
---
kind: instance
metadata:
  name: sourceip
spec:
  compiledTemplate: listentry
  params:
    value: source.ip | ip("0.0.0.0")
---
kind: rule
metadata:
  name: checkip
spec:
  match: source.labels["istio"] == "ingressgateway"
  actions:
  - handler: whitelistip
    instances: [ sourceip ]
```

---
# 부록

## IPVS는 무엇이고 과연 빠른가?
## IPTables 에서 네트워크 패킷 흐름
## Istio에서 파드에 사이드카 인젝션은 어떻게 하지?
## 커스텀 아답터
## 서비스 메쉬 퍼포먼스 벤치마크
## Istio 관련 용어 정리

---
## IPVS: 뭐지?
커널 수준의 L4 로드 밸런서(IP Virtual Server).
  - IPTables처럼 netfilter 커널 모듈 기반
  - 그러나 로드밸런싱만 하고 패킷 필터링이나 마스커레이딩, SNAT 등은 불가 .red[*]
  - 즉, 쿠버네티스에서는 둘다 쓰고 있음
  - IPTables과 달리 hash기반으로 항목이 많아도 빠름

.footnote[
.red[*] https://kubernetes.io/blog/2018/07/09/ipvs-based-in-cluster-load-balancing-deep-dive/
]

---
## IPVS: 쿠버네티스에서 IPVS를 쓰면 빠른가? .red[*]

.left-column[
|측정값            |서비스 개수| IPVS| IPTables|
|:----------------:|:---------:|:---:|:-------:|
|서비스 액세스 타임|      1,000| 10ms|   7-18ms|
|                  |     10,000|  9ms|80-7000ms|
|                  |     50,000|  9ms|동작 안함|
|메모리 사용량     |      1,000| 386M|     1.1G|
|                  |     10,000| 542M|     2.3G|
|                  |     50,000|1272M|      OOM|
|CPU 사용량        |          0|  386|      N/A|
|                  |     10,000|    0|  50~100%|
|                  |     50,000|    0|      N/A|

*서비스 개수는 적어도, 일단 IPVS 방식을 쓰는 편이 좋다.*
]

.right-column[
![drawing](https://www.projectcalico.org/wp-content/uploads/2019/04/Picture1.png)
![drawing](https://www.projectcalico.org/wp-content/uploads/2019/04/Picture2.png)
]

.footnote[
.red[*] 표는 https://www.objectif-libre.com/en/blog/2018/03/19/kubernetes-ipvs/ 발췌

그래프는 https://www.projectcalico.org/comparing-kube-proxy-modes-iptables-or-ipvs/ 발췌
]

---
## IPTables 에서 네트워크 패킷 흐름 .red[*]

(가끔씩은 확인해줘야 하는) IPTables. 패킷 흐름만 기억하자.

![drawing](https://www.booleanworld.com/wp-content/uploads/2017/06/Untitled-Diagram.png "iptables flow")

> 테이블에서 테이블로 이동할 때에 라우팅 여부를 판단하여 갈 방향이 결정됨

.footnote[
.red[*] https://www.booleanworld.com/depth-guide-iptables-linux-firewall/

prerouting, output: SNAT, postrouting: DNAT
]

---
## Istio에서 파드에 사이드카 주입은 어떻게 하지?

- 파일럿을 [쿠버네티스 어드미션 컨트롤러](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)에 등록
- 아래 메타데이터가 있는 경우에 한정하여 주입
  
  설정없이 전체로도, 강제도 가능
  ```
  sidecar.istio.io/inject: "true"
  ```

---
## 커스텀 아답터
필요한 조건 체크, 쿼터 체크, 헤더 조작, 텔레메트리 등이 있다면, 만들면 됩니다? :)

개발 가이드
- [Mixer Out of Process Adapter Walkthrough](https://github.com/istio/istio/wiki/Mixer-Out-Of-Process-Adapter-Walkthrough)
- [Mixer Compiled In Adapter Dev Guide](https://github.com/istio/istio/wiki/Mixer-Compiled-In-Adapter-Dev-Guide)

외부 샘플
- [Istio’s Mixer:Policy Enforcement with Custom Adapters](https://www.slideshare.net/TorinSandall/istios-mixer-policy-enforcement-with-custom-adapters-cloud-nativecon-17)
- [Production-Ready Istio Adapters](https://venilnoronha.io/set-sail-a-production-ready-istio-adapter)
- [Custom Auth Adapter](https://github.com/salrashid123/istio_custom_auth_adapter)
- [Custom Mixer Adapter, JWT & RBAC](http://www.servicemesher.com/blog/custom-istio-mixer-adapter)
- ...

커스텀 아답터 공유 가능
- https://github.com/istio/istio/wiki/Publishing-Adapters-and-Templates-to-istio.io

공유한 커스텀 아답터 목록
- https://istio.io/docs/reference/config/policy-and-telemetry/adapters/

---
## 서비스 메쉬 관련
서비스 메시가 많아짐에 따라, 퍼포먼스 벤치마크도 나옴
- [Meshery](https://layer5.io/meshery/), https://www.youtube.com/watch?v=LxP-yHrKL4M

뿐만 아니라
- [서비스 메시 인터페이스](https://smi-spec.io/) 표준화 활동 이뤄짐

---
## Istio 관련 용어 정리
많아 보이지만, 몇개 안된다. 정리 중...

k8s kind
- DestinationRule: 전송|중단 필터링
- VirtualService: (잉그레스|이그레스) 라우팅
- Gateway: (잉그레스|이그레스) 게이트웨이
- ServiceEntry: 이그레스 대상 지정
- ClusterRbacConfig, ServiceRoleBinding, ServiceRole: RBAC 조정
- Policy, MeshPolicy: 인증 정책

용어
- 믹서: 메트릭, 조건체크, 변조, 쿼터체크 핸들링
  - 아답터: 믹서 플러그인
  - 정책: 룰 + 인스턴스(=템플릿) + 핸들러(=아답터)
