# Architecture

- 보이지 않을 시 아래 링크 타고 들어갈 것
    - [https://docs.openstack.org/install-guide/_images/openstack-arch-kilo-logical-v1.png](https://docs.openstack.org/install-guide/_images/openstack-arch-kilo-logical-v1.png)

![Untitled](https://user-images.githubusercontent.com/67780144/98341467-3abffd80-2052-11eb-84fb-526b0b8810fc.png)

![Untitled 1](https://user-images.githubusercontent.com/67780144/98341470-3bf12a80-2052-11eb-8639-95999aa2b0b6.png)


# OpenStack Service Type

**[1] Compute → Nova**

인스턴스(가상머신, 서버)의 생성, 중지, 스케줄링 등 인스턴스의 라이프사이클을 담당한다. (실질적, 내부적으로는 KVM을 관리)

Firewall (외부에 있는 방화벽, Security Group)

Logical Architecture로 알아보는 Nova

원 논리 아카텍쳐는 그림이 너무 작아 Nova가 어떤 기능을 하는지 잘 보이질 않습니다. 그래서 논리(Logical) 아키텍처 중 Nova 부분만 크게 잘라봤습니다. 아키텍쳐를 잘 보면 Nova는 아래와 같은 프로세스로 일을 한다는 것을 확인할 수 있습니다.

1. Nova는 데쉬보드나 커맨드 라인 명령어가 호출하는 nova-api로부터 시작됩니다.
2. nova-api는 Queue를 통해 nova-compute에 인스턴스를 생성하라는 명령어를 전달하고,
3. 인스턴스 생성 명령어를 전달받은 nova-compute는 하이퍼바이저 라이브러리를 통해
4. 하이퍼바이저에게 인스턴스를 생성하라는 명령어를 다시 전달합니다.
5. 그 때 하이퍼바이저는 인스턴스를 생성하게 되는거죠!
6. 생성된 인스턴스는 nova-console를 통해 사용자가 접근할 수 있게 되는 겁니다.

Key Pair (SSH Key)

**[2] Networking → Neutron**

LAN (Network) → L2 ( VLAN, DHCP )

ROUTING        → L3

Load-Balance  → L4

Network-Based Firewall

** Floating IP : 정적으로 외부에 노출시킬 수 있는 IP, 인스턴스를 외부에 노출시키는 기술으로 client는 VM이 어떻게 구성되었는지 알 필요가 없다.

**[3] Block Storage → Cinder**

인스턴스의 영구 저장장치인 블록 장치를 제공한다. 블록 스토리지 장치를 생성하고 관리할 수 있도록 플러그인 가능한 드라이버 아키텍쳐 구조를 가지고 있다.

**[4] Identity → Keystone**

모든 컴포넌트의 인증을 담당하며, LDAP과 같은 기술을 사용하여 사용자의 중앙 디렉터리 기능을 한다.

Catalog를 관리한다.

** Catalog : OpenStack의 여러 컴포넌트들의 목록 ( Service의 목록이라고도 함 ) ⇒ Nova, Glance, Cinder, Swift 등등을 의미

1. Service 목록
2. Endpoint의 목록 ( Endpoint란 컴포넌트에 접근하기 위한 주소, 서비스에 접근하기 위한 주소 )

**[5] Dashboard → Horizon**

오픈스택 환경을 운영 및 관리할 수 있는 웹 기반의 셀프 서비스 포탈 인터페이스 제공

**[6] Telemetry → Ceilometer**

오픈스택 전체 환경을 에이전트 기반으로 데이터를 수집하여 모네터링 및 사용량, 벤치마킹, 확장성, 통계 등을 제공한다.

**[7] Orchestration → Heat**

템플릿 기반으로 다양한 클라우드 어플리케이션을 배치하고 관리할 수 있는 오케스트레이션 기능을 제공한다

IaC를 제공한다. → 오픈스택의 이미지, 볼륨을 코드로 구성하고 관리할 수 있다.

- Heat ↔ Celiometer 연동

**[8] Object Storage → Swift**

사용자가 사용할 수 있는 클라우드 스토리지이다. HTTP 기반의 RESTful API를 제공한다. 수평 확장 (Scale-Out)이 가능한 분산 스토리지이며, 복제본을 구성하여 안전한 대용량 스토리지를 제공한다.

독립적으로 작동할 수 있다. → 굳이 상호작용하자면 Glance의 이미지 저장 장소가 될 수 있지만 단독으로 작동할 수 있다. (유일함)

⇒ 위 서비스들이 Core Service이다.

**[9] Image Storage → Glance**

이미지 서비스를 제공하는 Glance

**[10] Workflow → Mistral**

이벤트 기반의 작업을 처리할 수 있는 기능이다. REST API를 사용하며 YAML 기반 언어를 사용하여 처리할 수 있다.

**[12] Database → Trove**

Database-as-a-Service 기능을 제공하며, 오픈소스 관계형 데이터베이스 또는 비-관계형 데이터베이스 엔진을 서비스로 사용할 수 있게 해준다. (오픈소스만 제공 가능하다.)

**[13] Elastic Map Reduce → Sahara**

대량의 데이터를 처리하기 위해 분산 응용 프로그램을 지원하는 Hadoop 클러스터를 쉽고 빠르게 배포하고 관리하는 기능을 제공한다.

**[14] Bare Metal → Ironic**

PXE, IPMI 기능을 사용해 베어메탈 시스템에 오픈스택 기능을 배포한다.

**[15] Messaging → Zaqar**

웹 개발자를 위한 멀티 테넌트 기반의 클라우드 메시지 큐 서비스를 제공한다.

⇒ 비동기화 방식의 통신이 가능하다.

💯 **Queue** : FIFO (First In First Out)
App이 여러개 있다고 가장하자. 통신을 다이렉트로 통신하는 방법도 있고 여러가지 있겠지만 
가운데에 Queue를 두고 전송하는 방법이 있다. 즉, 중앙에서 입출력을 통제하는 하나의 서비스를 Queue라고 한다. => 비동기화 방식

**[16] Shared File System → Manila**

공유 파일 시스템을 제공한다.

없었다면, VM을 하나 올려서 NFS 서버를 올렸어야 했지만 이 서비스가 나옴으로써 해결되었다.

**[17] DNS → Designate**

BIND, PowerDNS 등을 포함한 DNS 서비스를 관리하는 DNS-as-a-Service 기능을 제공한다.

**[18] Search → Searchlight**

ElasticSearch 오픈소스의 데이터 인덱싱 기술을 사용하여, 전체 오픈스택 서비스의 검색기능을 제공한다.

**[19] Key Manager → Barbican**

보안 저장소나 보안 관리 등을 위한 키 관리 기능을 제공한다.

Key-Value Storage라고도 한다. 간단한 텍스트를 안전하게 보관하기 위한 용도도 가능

→ Vault라고도 한다. 

**[20] Container → Magnum**

Docker Swarm, k8s, Mesos 등 컨테이너 엔진을 제공해 컨테이너 라이프사이클을 관리한다. 

# Openstack Node 구성

1. **컨트롤러 노드 : 전체 오픈스택 구성을 제어하기 위한 노드** 

    ( Keystone ⇒ SQL, Horizon ⇒ Message Queue, Glance, Ceilometer, Heat )

    - 주요 구성 서비스

        → Identity Service

        → Image Service

        → Dash Board Service

        → SQL Database

        → Message Queue

        → NTP (Network Time Protocol) ⇒ 시간이 맞지 않으면 Token, ID/PW가 맞다 하더라도 인증이 되질 않는다.

    - 옵션 구성요소

        → Orchestration Service

        → Telemetry Service

2. **네트워크 노드 : 외부 네트워크 뿐만 아니라 내부 네트워크 및 테넌트의 가상 네트워크를 제공한다.**

    ( Neutron )

    - 구성요소

        Networking Service ( L2, L3, DHCP, Metadata)

3. **컴퓨트 노드 : 하이퍼바이저를 통해 가상머신/인스턴스를 제공한다.**

    (Nova (KVM ))

    - 구성요소

        → Hypervisior

        → 가상머신/인스턴스 사용할 네트워크 플러그인 및 각종 에이전트 서비스

        → Telemetry 에이전트

4. **스토리지 노드 : 오픈스택에서 제공하는 스토리지 기능을 가진다. 블록 스토리지 노드, 오브젝트 스토리지 노드를 얘기함**

    ( Cinder (Block), Swift (Object), Manila (File) ) 

    - 블록 스토리지 노드 구성요소

        → Cinder Service

        → Cinder Storage

    - 오브젝트 스토리지 노드 구성요소

        → Swift Proxy Service

        → Swift Storage Service