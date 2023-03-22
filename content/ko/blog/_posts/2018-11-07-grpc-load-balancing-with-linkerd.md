---
layout: blog
title: '쿠버네티스에서 어려움 없이 gRPC 로드밸런싱하기'
date: 2018-11-07
---

**저자**: William Morgan (Buoyant)

**번역**: 송원석 (쏘카), 김상홍 (국민대), 손석호 (ETRI)

다수의 새로운 gRPC 사용자는 쿠버네티스의 기본 로드
밸런싱이 종종 작동하지 않는 것에 놀란다. 예를 들어, 만약
[단순한 gRPC Node.js 마이크로서비스
앱](https://github.com/sourishkrout/nodevoto)을 만들고 쿠버네티스에 배포하면 어떤 일이 생기는지 살펴보자.

![](/images/blog/grpc-load-balancing-with-linkerd/Screenshot2018-11-0116-c4d86100-afc1-4a08-a01c-16da391756dd.34.36.png)

여기 표시된 `voting` 서비스는 여러 개의 파드로 구성되어 있지만, 쿠버네티스의 CPU 그래프는 명확하게 
파드 중 하나만 실제 작업을 수행하고 있는 것(하나의 파드만 트래픽을 수신하고 있으므로)을 
보여준다. 왜 그런 것일까?

이 블로그 게시물에서는, 이런 일이 발생하는 이유를 설명하고,
[CNCF](https://cncf.io)의 서비스 메시(mesh)인 [Linkerd](https://linkerd.io) 및 서비스 사이드카(sidecar)를 활용한
gRPC 로드밸런싱을 쿠버네티스 앱에 추가하여, 이 문제를 쉽게 해결할 수 있는 방법을 설명한다.

# 왜 gRPC에 특별한 로드밸런싱이 필요한가?

먼저, 왜 gRPC를 위해 특별한 작업이 필요한지 살펴보자.

gRPC는 애플리케이션 개발자에게 점점 더 일반적인 선택지가 되고 있다. 
JSON-over-HTTP와 같은 대체 프로토콜에 비해, gRPC는 
극적으로 낮은 (역)직렬화 비용과, 자동 타입 
체크, 공식화된 APIs, 적은 TCP 관리 오버헤드 등에 상당한 이점이 있다.

그러나, gRPC는 쿠버네티스에서 제공하는 것과 마찬가지로
표준(일반)적으로 사용되는 연결 수준 로드밸런싱(connection-level load balancing)을 어렵게 만드는 측면도 있다. gRPC는 HTTP/2로 
구축되었고, HTTP/2는 하나의 오래 지속되는 TCP 연결을 갖도록 설계되있기 때문에,
모든 요청은 *다중화(multiplexed)*(특정 시점에 다수의 요청이 
하나의 연결에서만 동작하는 것을 의미)된다. 일반적으로, 그것은
연결 관리 오버헤드를 줄이는 장점이 있다. 그러나, 그것은 또한
(상상할 수 있듯이) 연결 수준의 밸런싱(balancing)에는 유용하지 않다는 것을 의미한다. 일단
연결이 설정되면, 더 이상 밸런싱을 수행할 수 없기 때문이다. 모든 요청이
아래와 같이 단일 파드에 고정될 것이다.

![](/images/blog/grpc-load-balancing-with-linkerd/Mono-8d2e53ef-b133-4aa0-9551-7e36a880c553.png)

# 왜 HTTP/1.1에는 이러한 일이 발생하지 않는가?

HTTP/1.1 또한 수명이 긴 연결의 개념이 있지만,
HTTP/1.1에는 TCP 연결을 순환시키게 만드는 여러 특징들이 있기 때문에,
이러한 문제가 발생하지 않는다. 그래서,
연결 수준 밸런싱은 "충분히 양호"하며, 대부분의 HTTP/1.1 앱에 대해서는
더 이상 아무것도 할 필요가 없다.

그 이유를 이해하기 위해, HTTP/1.1을 자세히 살펴보자. HTTP/2와 달리
HTTP/1.1은 요청을 다중화할 수 없다. TCP 연결 시점에 하나의 HTTP 요청만 
활성화될 수 있다. 예를 들어 클라이언트가 'GET /foo'를 요청하고,
서버가 응답할 때까지 대기한다. 요청-응답 주기가 
발생하면, 해당 연결에서 다른 요청을 실행할 수 없다.

일반적으로, 우리는 병렬로 많은 요청이 발생하기 원한다. 그러므로,
HTTP/1.1 동시 요청을 위해, 여러 HTTP/1.1 연결을 만들어야 하고,
그 모두에 걸쳐 우리의 요청을 발행해야 한다. 또한, 수명이 긴 HTTP/1.1
연결은 일반적으로 일정 시간 후 만료되고 클라이언트(또는 서버)에 의해 끊어진다.
이 두 가지 요소를 결합하면 일반적으로 HTTP/1.1 요청이
여러 TCP 연결에서 순환하며, 연결 수준 밸런싱이 작동한다.

# 그래서 우리는 어떻게 gRPC의 부하를 분산할 수 있을까??

이제 gRPC로 돌아가보자. 연결 수준에서 밸런싱을 맞출 수 없기 때문에, gRPC 로드 밸런싱을
수행하려면, 연결 밸런싱에서 *요청* 밸런싱으로 전환해야
한다. 즉, 각각에 대한 HTTP/2 연결을 열어야
하고, 아래와 같이, 이러한 연결들로 *요청*의 밸런싱을 맞춘다.

![](/images/blog/grpc-load-balancing-with-linkerd/Stereo-09aff9d7-1c98-4a0a-9184-9998ed83a531.png)

네트워크 측면에서, L3/L4에서 결정을 내리기 보다는 L5/L7에서 결정을 
내려야 한다. 즉, TCP 연결을 통해 전송된 프로토콜을 이해해야 한다.

우리는 이것을 어떻게 달성해야할까? 몇 가지 옵션이 있다. 먼저, 우리의 애플리케이션
코드는 대상에 대한 자체 로드 밸런싱 풀을 수동으로 유지 관리할 수 있고, 
gRPC 클라이언트에 [로드 밸런싱 풀을 
사용](https://godoc.org/google.golang.org/grpc/balancer)하도록 구성할 수 있다. 이 접근 방식은
우리에게 높은 제어력을 제공하지만, 시간이 지남에 따라 파드가 리스케줄링(reschedule)되면서 풀이 변경되는
쿠버네티스와 같은 환경에서는 매우 복잡할 수 있다. 이 경우, 우리의
애플리케이션은 파드와 쿠버네티스 API를 관찰하고 자체적으로 최신 상태를 
유지해야 한다.

대안으로, 쿠버네티스에서 앱을 [헤드리스(headless)
서비스](/ko/docs/concepts/services-networking/service/#헤드리스-headless-서비스)로 배포할 수 있다.
이 경우, 쿠버네티스는 서비스를 위한 DNS 항목에 
[다중 A 레코드를 생성할 것이다](/ko/docs/concepts/services-networking/service/#헤드리스-headless-서비스). 
만약 충분히 진보한 gRPC 클라이언트를 사용한다면,
해당 DNS 항목에서 로드 밸런싱 풀을 자동으로 유지 관리할 수 있다.
그러나 이 접근 방식은 우리를 특정 gRPC 클라이언트로를 사용하도록 제한할 뿐만 아니라, 
헤드리스 서비스만 사용하는 경우도 거의 없으므로 제약이 크다.

마지막으로, 세 번째 접근 방식을 취할 수 있다. 경량 프록시를 사용하는 것이다.

# Linkerd를 사용하여 쿠버네티스에서 gRPC 로드 밸런싱

[Linkerd](https://linkerd.io)는 [CNCF](https://cncf.io)에서 관리하는 쿠버네티스용 
*서비스 메시*이다. 우리의 목적과 가장 관련이 깊은 Linkerd는, 클러스터 수준의 권한 
없이도 단일 서비스에 적용할 수 있는 
*서비스 사이드카*로써도 작동한다. Linkerd를 서비스에 추가하는
것은, ​​각 파드에 작고, 초고속인 프록시를 추가하는 것을 의미하며, 이러한 프록시가
쿠버네티스 API를 와치(watch)하고 gRPC 로드 밸런싱을 자동으로 수행하는 것을 의미이다. 우리가 수행한 배포는
다음과 같다.

![](/images/blog/grpc-load-balancing-with-linkerd/Linkerd-8df1031c-cdd1-4164-8e91-00f2d941e93f.io.png)

Linkerd를 사용하면 몇 가지 장점이 있다. 첫째, 어떠한 언어로 작성된 서비스든지, 어떤 gRPC 클라이언트든지
그리고 어떠한 배포 모델과도 (헤드리스 여부와 상관없이) 함께 작동한다.
Linkerd의 프록시는 완전히 투명하기 때문에, HTTP/2와 HTTP/1.x를 자동으로 감지하고 
L7 로드 밸런싱을 수행하며, 다른 모든 트래픽을
순수한 TCP로 통과(pass through)시킨다. 이것은 모든 것이 *그냥 작동*한다는 것을 의미한다.

둘째, Linkerd의 로드 밸런싱은 매우 정교하다. Linkerd는
쿠버네티스 API에 대한 와치(watch)를 유지하고 파드가 리스케술링 될 때
로드 밸런싱 풀을 자동으로 갱신할 뿐만 아니라, Linkerd는 응답 대기 시간이 가장 빠른 파드에 
요청을 자동으로 보내기 위해 *지수 가중 이동 평균(exponentially-weighted moving average)* 을 
사용한다. 하나의 파드가 잠시라도 느려지면, Linkerd가 트래픽을 
변경할 것이다. 이를 통해 종단 간 지연 시간을 줄일 수 있다.

마지막으로, Linkerd의 Rust 기반 프록시는 매우 작고 빠르다. 그것은 1ms 미만의 
p99 지연 시간(<1ms of p99 latency)을 지원할 뿐만 아니라, 파드당 10mb 미만의 RSS(<10mb of RSS)만 필요로 하므로
시스템 성능에도 거의 영향을 미치지 않는다.

# 60초 안에 gRPC 부하 분산

Linkerd는 시도하기가 매우 쉽다. 단지 &mdash;랩탑에 CLI를 설치하고&mdash; [Linkerd 시작 
지침](https://linkerd.io/2/getting-started/)의 단계만 따르면 된다. 
클러스터에 컨트롤 플레인과 "메시" 
서비스(프록시를 각 파드에 주입)를 설치한다. 서비스에 Linkerd가 즉시 
실행될 것이므로 적절한 gRPC 밸런싱을 즉시 확인할 수 있다.

Linkerd 설치 후에, 예시 `voting` 서비스를 
다시 살펴보자.

![](/images/blog/grpc-load-balancing-with-linkerd/Screenshot2018-11-0116-24b8ee81-144c-4eac-b73d-871bbf0ea22e.57.42.png)

그림과 같이, 모든 파드에 대한 CPU 그래프가 활성화되어 모든 파드가
&mdash;코드를 변경하지 않았지만&mdash; 트래픽을 받고 있다. 짜잔,
마법처럼 gRPC 로드 밸런싱이 됐다!

Linkerd는 또한 내장된 트래픽 수준 대시보드를 제공하므로, 더 이상 
CPU 차트에서 무슨 일이 일어나고 있는지 추측하는 것이 필요하지 않다. 다음은 각 파드의 
성공률, 요청 볼륨 및 지연 시간 백분위수를 보여주는
Linkerd 그래프이다.

![](/images/blog/grpc-load-balancing-with-linkerd/Screenshot2018-11-0212-15ed0448-5424-4e47-9828-20032de868b5.08.38.png)

각 파드가 약 5 RPS를 얻고 있음을 알 수 있다. 또한, 
로드 밸런싱 문제는 해결되었지만 해당 서비스의
성공률에 대해서는 아직 할 일이 남았다는 것도 살펴볼 수 있다. (데모 앱은 독자에 대한 연습을 위해 의도적으로
실패 상태로 만들었다. Linkerd 대시보드를 사용하여
문제를 해결할 수 있는지 살펴보자!)

# 마지막으로

쿠버네티스 서비스에 gRPC 로드 밸런싱을 추가하는 방법에 
흥미가 있다면, 어떤 언어로 작성되었든 상관없이, 어떤 gRPC
클라이언트를 사용중이든지, 또는 어떻게 배포되었든지, Linkerd를 사용하여 단 몇 개의 명령으로 
gRPC 로드 밸런싱을 추가할 수 있다.

보안, 안정성 및 디버깅을 포함하여 Linkerd에는 더 많은 특징
및 진단 기능이 있지만 이는 향후 블로그 게시물의 주제로 남겨두려 한다.

더 알고 싶은가? 빠르게 성장하고 있는 우리 커뮤니티는 여러분의 참여를 환영한다!
Linkerd는 [CNCF](https://cncf.io) 프로젝트로
[GitHub에 호스팅 되어 있고](https://github.com/linkerd/linkerd2), [Slack](https://slack.linkerd.io),
[Twitter](https://twitter.com/linkerd), 그리고 [이메일 리스트](https://lists.cncf.io/g/cncf-linkerd-users)를 통해 커뮤니티를 만날 수 있다.
접속하여 커뮤니티에 참여하는 즐거움을 느껴보길 바란다!