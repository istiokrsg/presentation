# 시청각 자료 리뷰

## 영상

* [Istio - The Packet's-Eye View - Matt Turner, Tetrate](https://youtu.be/zJnYuFsLHfY) Kubecon 영상 리뷰 - 이상근
  - https://mt165.co.uk/speech/life-of-a-packet-istio-cloud-native/

* 리뷰내용

```text
~ 1:58 
자기소개, 인사
Istio 들어본 사람, 써 본 사람, 프로덕션에 적용해 본 사람 호구 조사

~ 2:11 
아웃라인

~ 3:33
Istio 공식 문서 아키텍쳐 다이어그램 설명
공짜 L7이 어쩌고, 서비스카, 사이드메쉬에 대해서 중얼

~ 5:20
Istio 전체 구조에서 생략된 내용들, etcd, API 서버, istioctl
컨트롤 플레인마다 API 서버가 배포, HA 가능 
kubernetes를 쓴다면 istio는 user namespace에 Pod

~  8:29
Ingress 부터 패킷 흐름
envoy가 기존 kubernetes의 인그레스를 대체
로드밸런서를 통해서 들어온 요청을  인그레스 계층 envoy에서 가로채서 서비스로 전달하는 과정
istio에서 kubernetes에 비해 flexible, powerful한 리소스 타입을 정의

~9:20
서비스로 트래픽이 들어오면 envoy가 가로채는데 이 과정은 매직처럼 보임

~ 11:15
자세한 내용을 알아보기 위해 컨테이너를 뜯어봄
리눅스에서 컨테이너는 퍼스트 클래스가 아니며 namespace가 존재할 뿐, 6가지 namespace 설명
+ cgroup으로 hw isolation가 함께 컨테이너를 만듬


~12:45
Kubernetes Pod 구조 설명
mnt, uts namespace 공유


~16:32
Iptable 변경해서 envoy로 모든 트래픽이 거쳐서 전달되도록 사이드카를 만드는 과정
Scala 등 JVM 언어로 사이드카 만들기가 어렵다는 내용 설명하면서 Linkerd 등 언급하면서 디스

~20:40
CRI를 통해서 쿠버네티스가 Pod 생성하는 과정 설명, init 컨테이너를 활용, alphine 이미지, core dump 설정, Istio-init 스크립트 실행
의문점) 앞의 Kubernetes Pod 설명에는 pid namespace를 공유하고 있었는데 왜 여기서는 분리되어 있는가?

-26:27
서비스 A에서 서비스 B 호출 시 어떻게 Pod을 선택하는지
Kubernetes Service Discovery와 API로 endpoint 목록을 찾는 과정

~28:04
pilot과 envoy data plane API 상호작용 설명
Pilot은 Ahead of time configuration, Preconfigured
v1으로 90%, v2로 10%로 트래픽 보내는 것

~29:07
API global rate limit, ACL 적용 등 호출할 때 마다 확인해야하는 Policy는 Mixer로 처리

~33:58
IP 네트워크 라우팅 아키텍쳐 설명 
각종 프로토콜과 control plane 동작
인터럽트와 Dataplane 동작
Envoy, Pilot, Mixer를 IP 라우팅 아키텍쳐에 비교해서 설명
IP 라우팅에서 할 수 없는 L7 수준 처리 설명(언어 처리, 블랙리스트 등) 

~36:40
Mixer Caching 설명, SPOF 처럼 보이지만 Caching되어 문제가 없음
Mixer는 Enovy를 절대 블러킹하지 않음

~38:43
VPC 등이어도 보안 적용해야 하는 이유, Citadel, mTLS
지금은 인증서 일치하지 않으면 에러만 나지만, 커널이나 컨테이너 이미지 등 검증도 앞으로는 가능할 것으로 기대

~39:17
Outgoing Egress Layer

~40:36
요약

~ 42:25
질문1) 믹서 레이턴시로 성능 문제는 없나?

~ 44:08 
질문2) Reverse Proxy 대체냐 보완이냐?

~ 46:16
질문3) Istio ingress Gateway에서 기존의 deprecated된 Kubernetes Ingress를 다른 타입의 인그레스(hepatic Contour 등)으로 교체될 수 있도록 Istio의 Ingress 계층을 별도로 분리하려는 시도가 있는가?


```