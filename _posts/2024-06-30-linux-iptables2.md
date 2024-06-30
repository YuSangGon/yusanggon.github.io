---
layout: post
title: Linux (Ubuntu) IPTABLES 명령어
subtitle: Linux Study Note2
author: 코딩하는 랄로
categories: linux
tags: [linux, ubuntu, iptables]
---


## 들어가며
저번 글에서는 linux 의 iptables 의 주요 개념들과 전체적인 흐름에 대해서 간단하게 알아보았습니다. packet 은 네트워크의 흐름에 따라 chain 을 지나게 되는데 chain은 table 들이 연결된 것이고 더 세세하게 본다면 결국은 rule 들이 연결된 것입니다. 

이렇게 각각의 규칙들을 지나면서 최종적으로 해당 요청에 대한 packet 을 거부 또는 받아들일 것인지에 대한 결정을 함으로써 방화벽의 역할을 수행할 수 있게 됩니다. 이번 글에서는 iptables 의 여러 명렁어 들을 살펴보면서 어떻게 규칙을 다루는지에 대해서 알아보겠습니다. 


## 적용 순서
저번 글에서 한번 다루었지만 테이블의 규칙이 각 체인에서 어떤 순서로 적용되는지는 명렁어를 통해 rule 을 추가하고 다루는 것을 배울 이번 포스팅에서 더욱 중요하기 때문에 한번 더 해당 표를 가지고 왔습니다.

![figure 3](/assets/images/iptables/iptables_photo3.png)


## iptables 명령 기본 표기법

iptables 의 명령 기본 표기법은 다음과 같습니다.

```shell
iptables [-t table] -Options [chain] [matching options] [action] 
```

각긱은 다음을 의미합니다.

- table : 이름 그대로 어떤 table 을 사용하는지 지정해줍니다. 없을 경우, 기본값으로 filter table 로 지정됩니다.
- Options [chain] : chain 에 대한 옵션을 설정. 
  - A : 특정 chain 의 마지막에 rule 을 추가
  - C : 사용자 정의 chain 에 rule 을 추가하기 전에 먼저 확인
  - D : chain 을 제거
  - E : 사용자 정의 chain 의 이름 변경
  - F : 해당 chain 의 rule 을 제거. chain 을 지정해주지 않으면 테이블의 모든 룰을 제거
  - h : 옵션들을 리스트업 = help
  - I : chain 의 rule 을 첫 부분에 추가
  - L : 현재 chain 의 rule 들을 보여줌
  - N : 새로운 사용자 정의 chain 을 생성
  - P : chain 의 기본 rule 을 적용
  - R : chain 의 규칙을 교체
  - X : 사용자 정의 비어있는 chain 을 제거
  - Z : table 의 모든 chain 들의 byte 와 packet counters 를 0 으로 초기화
- matching options : 매칭 조건을 설정한다. 조건이 일치하면, action 을 수행하고 일치하지 않으면 체인의 그 다음 rule 로 넘어간다. matching options 는 3개의 타입으로 구분된다.
  - Generic parameters
    - p : protocol
    - s : source ip
    - d : destination ip
    - i : input interface
    - o : output interface
  - Implicit parameters
    - TCP
    - -sport : source port
    - -dport : destination port
    - -tcp-flags : flag 를 지정. 첫번째 인수는 검사하고자 하는 지시자 리스트, 두번째 인수는 지시자에 어떻게 할 것인가를 설정
  - explicit parameters
    - Match extensions
    - -m
    - Conntrack
    - dscp
    - ecn
    - iprange
  - actions : 매칭 조건을 만족 시켰을 때 실행할 작업이다. 대게 -j 를 사용하여 해당 action 을 실행시킨다.
    - ACCEPT : 연결을 허용
    - REJECT : 연결을 막음
    - DROP : 연결을 막고 오류를 보냄
    - RETURN : 체인을 끝냄


## 예제를 통해서 사용법 익히기
몇가지 예제를 통해서 iptables 의 여러 옵션들을 다루는 방법을 익혀보겠습니다. 


# iptable 의 기본 정책 설정

```
iptables --pulish INPUT DROP
iptables --pulish OUTPUT DROP
iptables --pulish FORWARD DROP
```

INPUT, OUPUT, FORWARD 에 대한 모든 traffic 을 막습니다. 반대로 허용하기 위해서는 ACCEPT 를 해주면 됩니다.


# 특정 IP 주소 막기
```
iptables -A INPUT -s 192.268.07.23 -j DROP
```

192.268.07.23 에서 오는 요청은 모두 막아버립니다. 반대로 ACCEPT 하면 모두 허용합니다.


# 특정 port 에 대한 traffic 막기
```
iptables -A OUTPUT -p tcp --dport 22 -j DROP
```

destination 의 port 가 22 인 것은 DROP 합니다. 반대로 ACCEPT 하면 허용합니다.


# 시스템에서 보내는 SMTP 막기
```
iptables -A OUTPUT -p tcp --dport 25, 465, 587 -j DROP
```

시스템이 SMTP 로 보내는 메일을 모두 못 나가게 막아버립니다.


# 정책이 잘 적용되었는지 ping 으로 확인해보기
```
iptables -A INPUT -s 192.268.07.23 -j DROP
ping 192.268.07.23
```
ping 명렁어를 이용하여 정책이 잘 적용되었는지 확인할 수 있습니다.


# iptables rule 저장하고 불러오기
iptables 의 rule 을 저장하고 불러오는 것도 가능합니다. 이를 이용하여 필요에 따라 여러 버전의 설정들을 해놓고 원하는 설정을 적용할 수 있는 것이죠. 먼저 저장하는 방법입니다. 

```
sudo iptables-save > filename.txt
```

이때 filename 은 원하는 파일명을 지정해주면 됩니다. 다음은 저장해 높은 설정 파일을 불러오는 방법입니다.

```
sudo iptables-restore > filename.txt
```

매번 시스템 부팅시, 수동으로 백업된 rule 을 불러오기 번거롭기 떄문에 이번에는 자동으로 해당 백업 파일을 불러와서 설정 파일을 적용시키는 방법을 알아보겠습니다.

```
#root 사용자로 변경
sudo -s

#iptables 자동 복구 툴인 iptable-persistant 설치
apt install iptables-persistent

#설치가 제대로 되었는지 확인. 설치가 제대로 되었다면 해당 폴더가 생성
ls -al /etc/iptables/

#서비스 상태 확인
systemctl status netfilter-persistent
systemctl is-enabled netfilter-persistent

#테스트를 위한 룰 추가
iptables -A INPUT -s 192.268.07.23 -j DROP

#리부팅 후 해당 룰이 저장되었는지 확인
iptables-save | grep 192.268.07.23
```


## 마무리
간단하게 iptables 의 여러 옵션들을 통해서 rule 을 설정하는 방법을 알아보았는데, 옵션들도 워낙 많고 적용 순서부터 생각해야 할 요소가 너무 많은 것을 느낄 수 있었습니다. 그렇다 보니 각 옵션들에 대해서 그리고 여러 rule 을 잘 구축하여 제대로 된 방화벽을 구축하기 위해서는 많은 노력이 필요합니다. 그래서 다음 포스팅에서는 이런 iptables 를 더 직관적인고 쉽게 다룰 수 있는 많은 사람들이 애용하고 있는 툴인 ufw 를 한번 알아보겠습니다.



<br>
<br>
<br>

# Reference
[https://data-flair.training/blogs/what-are-iptables-in-linux/](https://data-flair.training/blogs/what-are-iptables-in-linux/)<br>
[https://tommypagy.tistory.com/581](https://tommypagy.tistory.com/581)