---
layout: post
title: Linux (Ubuntu) IPTABLES 이해하기
subtitle: Linux Study Note1
author: 코딩하는 랄로
categories: linux
tags: [linux, ubuntu, iptables]
---


## 들어가며

리눅스 서버에 도커 환경을 구축해서 애플리케이션을 배포하고 싶어, 리눅스에 대해 지식이 전무하지만 부딪히면서 배워보자는 생각에 무작정 리눅스 서버를 설치하고 도커를 설치하였습니다. 그리고 ufw 를 사용하여 원격 접속과 관련해서 설정을 해 준다음 테스트 삼아 도커 컨테이너를 띄웠는데, 해당 포트를 허용하지 않았음에도 컨테이너로 접속되는 것을 발견할 수 있었습니다. ( 원인과 해결 방법에 관련해서는 이후의 포스팅에서 다루어보겠습니다. )

어찌저찌 구글링을 통해서 여러 방법과 시행착오를 겪으면서 결국에 도커에 방화벽 설정을 적용하는데에 성공하였습니다. 또 구글링을 통해 해결해보려했지만 해당 부분에 대해서 자료도 별로 없어서 진척이 잘 안되던 중에, 너무 도커와 ufw 에 매몰되어 있다는 생각이 들었습니다.

리눅스에서 방화벽이 어떤 구성으로 되어 있고 어떻게 동작하는지, 네트워크의 흐름을 어떻게 제어하는지에 대한 이해없이 무작정 해결방법만 찾고 무작정 적용해보니 잘 안 되는것 같다는 생각이 들어, 문제를 잠시 내려놓고 기초 지식을 먼저 쌓아야 겠다는 생각이 들었고, 필요한 지식들을 쌓아가면서 결국에는 오류없이 도커에 방화벽을 적용할 수 있게 되었습니다.

그러한 과정을 글로 남기고 싶었고, 그 첫번째를 linux 를 설치하면 기본적으로 같이 설치되는 iptables 에 대한 글로 시작하려 합니다. 이제 막 리눅스를 공부하고 있는 단계라 틀린 부분이 있으면 언제든지 지적해주시면 감사히 배우겠습니다. ( 여러 글들을 참고하였고 글 맨마지막 부분에 링크 걸어두었으니 들어가서 읽어보셔도 좋을꺼 같습니다. )


## IPTABLES

iptables 는 리눅스의 방화벽 역할을 합니다. 그럼 방화벽이란 무엇일까요? 방화벽은 network traffic 을 filtering 해 주는 유일한 방법입니다. 즉, linux 에서는 ipatbles 를 통해 네트워크 트래픽을 통제 하는 것이죠. 참고로 위에서 언급한 ufw 는 이런 iptables 를 직접 사용하기에는 다소 어려워 더 쉽고 직관적이게 설정할 수 있게 해주는 도구입니다. 

더 자세히 이야기하면, iptables 가 직접 packet 등을 filtering 하면서 네트워크를 통제하지는 않습니다. 실제로 리눅스 커널에 탑재된 netfilter 기능이 이러한 역할을 수행하고 iptables 를 통해 netfilter 의 규칙을 구축하는 것일 뿐입니다. 즉, iptables 는 netfilter 의 룰셋구축 툴입니다. ( iptables 가 네트워크를 통제한다는 표현을 쓰더라도 실제로는 룰셋구축 툴을 통해서 netfilter의 룰을 설정해준다라고 이해해주시면 될꺼 같습니다. )

iptables 는 terminal interface 이기 때문에 명렁어를 통해서 네트워크의 흐름을 통제하기 위한 규칙을 설정할 수 있고 이러한 설정을 통해 특정 네트워크를 막거나 또는 허용함으로써 리눅스 서버를 더욱 안전하게 구축할 수 있게 됩니다. 보안은 네트워크 환경에서 상당히 중요한 요소 중 하나이기 때문에 ipatbles 를 이해하고 사용할 줄 알아야 하는 것이죠. ( 리눅스 서버는 매우 안전하지만, 인터넷 세계에는 워낙 방대하기 때문에 더욱 보안적인 요소를 강화시킬 필요가 있습니다. )

참고로, iptables 는 특정 ip address 에 대해 허용 또는 막는 등의 설정 테이블을 만든다 해서 붙여진 이름입니다.


## IPTABLES 의 구성 요소

iptables 동작 방식을 이해하기 위해서는 당연히 iptables 이 사용하는 개념(구성요소) 에 대해서 알고 있어야 합니다. iptables 에는 3가지의 중요한 개념이 있는데 아래와 같습니다.
<br>
- Tables
- Chains
- Rules
<br>
각각에 대해서 알아보겠습니다.
<br>

# Tables 
Tables 는 규칙들을 정의한 그룹입니다. ( 저는 이렇게 이해했는데, 더 좋은 해석이 있다면 알려주세요!! ) Iptables 는 table 을 5개 가지고 있고 이 5개는 3개의 main 과 2개의 sub 로 나뉘게 됩니다. main, sub 라는 말 그대로, 3개의 main table 이 주요 기능을 가지고 있고 sub 는 보조 기능을 가지고 통제를 합니다. 먼저 3가지 main table 부터 알아보겠습니다.

<br>

- Filter table : main 테이블 중에서도 default 테이블로 별다른 설정이 없다면 default 테이블인 filter table 이 적용됩니다. filter table 은 packet 을 filtering 하는 규칙들의 그룹
- NAT table : NAT 은 Network Address Translation 의 약어로, 이름 그대로 네트워크 주소를 변환하는 규칙들의 그룹
- Mangle table : packet의 IP headers 를 변경하는 규칙들의 그룹

<br>

다음의 2가지는 sub table 입니다. 간단하게 알아보겠습니다.
- RAW table : 해당 테이블은 connection tracking 에 사용되는 테이블입니다. packet 을 mark 하고 어떤 connetion or session 의 packet 인지를 파악할 수 가 있습니다.
- Security table : 보안 테이블은 패킷에 내부 SELinux 보안 콘텐츠 마크를 전송하는 데 사용되며, 이는 SELinux 또는 SELinux 보안 컨텍스트를 해석할 수 있는 다른 시스템이 이러한 패킷을 처리하는 방식에 영향을 미칩니다.

<br>
<br>

# Chains
Chains 는 이름에서 알 수 있듯이 packet 이 지나는 경로(흐름) 의 특정 지점에 규칙들의 연결을 말합니다. Iptables 는 그 특정 지점이 어떤 위치에 있냐에 따라 다음의 5개의 chain 으로 구분합니다. 

<br>

- Pre Routing : incoming packet 이 시스템 내부로 막 진입했을 때의 point
- Input :  시스템 내부로 들어올 때 pre routing 을 거친 다음의 point
- Forward : pre routing을 거친 다음 forward 로 시스템 외부로 나갈 때의 point
- Output : 시스템 내부에서 외부로 나갈 때의 point
- Post Routing : pre routing 의 반대로, output, forward 등으로 packet 이 외부로 빠져나가기 직전의 point

<br>

이를 쉽게 이해하기 위해 그림으로 나타내면 다음과 같습니다.

![figure 1](/assets/images/iptables/iptables_photo1.png)

<br>

다시 tables 로 돌아가면 tables 은 규칙들의 그룹입니다. 그리고 chain 은 이러한 그룹들이 연결된 것입니다. 즉, prerouting point에서 여러 테이블이 체인 형식으로 연결되어 있는 것을 prerouting chain 이라고 하는 것이죠. 이를 이해하기 쉽게 그림으로 표현하면 다음과 같습니다.

![figure 2](/assets/images/iptables/iptables_photo2.png)

<br>

하지만 여기서 중요한 점은 어떤 chain 이냐에 따라 연결할 수 있는 table 에 대한 제한이 있다는 것입니다. 각각의 main 테이블이 어떤 Chain 에서 사용될 수 있는지는 아래와 같습니다.

<br>

- Filter table : input, output, forward
- NAT table : prerouting, ouput, postrouting
- Mangle table : prerouting, input, output, forwarding, postrouting

<br>

위의 제한에 따라 연결된 Table 들은 해당 Chain 내에서 순서에 따라 자신이 가지고 있는 규칙을 packet 에 매칭해보고 일치하면 적용시키는 것이죠.


# Rules
Table 은 규칙(Rule)들의 그룹이라고 하였는데 그 규칙이 바로 사용자가 네트워크 트랙픽을 관리하기 위해서 정의하는 명령이 입니다. packet 이 들어오면 각각의 chain 에서 연결된 table 을 순차적으로 돌면서 테이블에 정의된 규칙에 매칭이 되면 action 을 수행하고 매칭이 안되면 다음 rule 또는 table 로 넘어가게 됩니다. 이러한 Rule 은 두개의 component 를 가지고 있습니다.

<br>

- matching component : 허용가능한 packet 의 상태. ex. protoco type, IP address, interface header, ...
- target component : packet 이 rule 에 매칭되었을 때의 action. 이 action 에 따라 target 은 또 2 가지의 target 으로 나누어지게 됩니다.
  - Terminating targets : 특정 chain 에서 이 action 을 끝으로 더 이상 Chain 을 돌 지 않게 되는 action. : Accept, Drop, Queue, Reject, Return
  - Non-terminating targets : 해당 action 이 후에도 계속해서 chain 을 도는 action

<br>


## 정리
리눅스는 커널에 탑재된 netfilter 를 이용하여 해당 시스템으로 들어오는 네트워크 packet 을 통제하고 다룹니다. 이 때 iptables 라는 룰셋업 툴을 사용하여 netfilter 의 규칙을 구축하게 됩니다. 이런 iptables 는 네트워크의 흐름에 따라 해당하는 chain 을 지나게 되는데 chain은 테이블들이 연결된 것이고 더 세세하게 본다면 결국은 규칙들이 연결된 것입니다. ( table 은 규칙들이 모인 그룹이기 때문에 )

이렇게 각각의 규칙들을 지나면서 최종적으로 해당 요청에 대한 packet 을 거부 또는 받아들일 것인지에 대한 결정을 함으로써 시스템 내부의 보안을 더 강화시키는 방화벽의 역할을 수행할 수 있게 됩니다. 다음 글에서는 요번 글에서의 개념을 토대로 iptables 에 여러 명렁어 들을 살펴보면서 어떻게 규칙을 다루는지에 대해서 알아보겠습니다. 

<br>
<br>
<br>

# Reference
[https://data-flair.training/blogs/what-are-iptables-in-linux/](https://data-flair.training/blogs/what-are-iptables-in-linux/)
[https://andrewpage.tistory.com/38](https://andrewpage.tistory.com/38)
[https://m.blog.naver.com/skauter/220052866346](https://m.blog.naver.com/skauter/220052866346)