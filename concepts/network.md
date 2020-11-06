## Openstack을 구성하기 위해서는 총 3가지가 필요해보인다.

- Management Network
    - 모든 Node가 통신
    - SSH 기반 통신
- External Network
    - Network Node만 NIC를 연결
- VM Data Network
    - Network, Compute Node만 연결
    - 내부 VM들이 이 네트워크를 통해 통신을 한다.

# 개념 정리

- Neturon은 네트워크 생성/변경/삭제에 대한 API만 제공!

    ⇒ 실제로는 Linux Plug-in을 통해서 네트워크가 구성되는 것임

## TUN/TAP

- TUN/TAP 장치는 가상 네트워크 인터페이스를 생성하며 TUN 장치는 IP 패킷을, TAP 장치는 Ethernet 패킷을 제어하는 기능을 제공한다.
- TUN/TAP 장치를 이용하면 qemu로 완전한 가상 네트워크 환경을 구축할 수 있음.
- TUN : network **TUN**nel (Layer 3 패킷을 다룸)

    →IP 패킷을 읽거나 쓰기가 가능

- TAP : network **TAP** (Layer 2 패킷을 다룸)

    → Ethernet 인터페이스와 연동되어 Ethernet Header가 추가된 Ethernet 패킷을 읽거나 쓰기 가능

- GRE Tunnel (Generic Routing Encapsulation)

    → 인터넷 프로토콜 네트워크를 통해 가상 지점 간 링크 또는 지점 간 멀티 포인트 링크 내에 다양한 네트워크 계층 프로토콜을 캡슐화할 수 있는 터널링 기술이다.

- Tunneling : 라우터와 라우터, 네트워크와 네트워크 사이에 논리적인 경로를 통하여 연결하는 방법
- Overlay Network : 물리 네트워크를 위해 성립되는 가상의 네트워크

    ⇒ Overlay Network 안의 노드는 가상, 논리적 링크로 연결 될 수 있으며 각 링크는 네트워크 안에서 많은 물리적 링크를 통하지만 그렇다고 물리적 링크를 고려하지 않는다.

- SDN(Software Defined Network)
- Open vSwitch
- dnsmasq
- br-int
- qbr
- veth
- NAT : 한정된 하나의 공인IP를 여러개의 내부 사설 IP로 변환하여 공인IP를 절약하고, 외부 침입에 대한 보안성을 높이기 위한 기술
- L3 Agent : Openstack 내부 Tenant 네트워크 상의 VM들의 외부망 연결을 위해 L3/NAT 포워딩 서비스를 제공하는 Agent (Openstack 내부 가상 네트워크에서 라우터의 역할을 구현한 서비스)
- Tenant : 오픈스택 내의 사용자 그룹
- VXLAN : eXtensible한 VLAN을 의미한다.

    → VLAN, Layer 2의 좀 더 큰 확장성을 제공하겠다는 뜻이다.

# Network 동작 원리의 이해

- Reference

    [https://m.blog.naver.com/love_tolty/220237750951](https://m.blog.naver.com/love_tolty/220237750951)

    [https://printf.kr/entry/Openstack-Network-구축-과정-이해-–-LinuxBridge](https://printf.kr/entry/Openstack-Network-%EA%B5%AC%EC%B6%95-%EA%B3%BC%EC%A0%95-%EC%9D%B4%ED%95%B4-%E2%80%93-LinuxBridge)

# 개요

- 오픈스택을 정확히 이해하기 위해서는 내부가 어떻게 통신되는지 이해하는 것이 중요합니다.
- 그래서 각 내부 VM 인스턴스들이 서로 어떻게 통신되고, VM 인스턴스들이 외부와 통신할 때 어떤 원리로 이루어지며, 각 서비스들 간의 유기적인 관계를 파악하고자 합니다.
- 최종적으로 어느정도 오픈스택에 대한 이해도가 생긴다면 직접 Manual을 확인하여 OpenStack Stable 버젼을 구축해보고자 합니다.

# Openstack Network 종류

1. Management Network : 관리용 네트워크, 컴포넌트(Nova, Neutron, ..)들이 서로 API 호출시 사용
2. Tunnel Network : VM 인스턴스 간 네트워크를 구축할 때 사용되는 네트워크

    ⇒ GRE, VXLAN 등의 기술이 사용되며, **Overlay Network**라고 부름!!

3. External Network : VM 인스턴스가 인터넷과 통신하기 위한 네트워크

## Provider / Self-Service Network ?

### 1. Provider Network

⇒ 오픈스택을 서비스하는 사람이 구축한 네트워크가 VM 인스턴스에 할당되는 네트워크

⇒ 이 네트워크는 인터넷과 연결되어 있는 네트워크 (그렇다고 Public Network는 X)

### 2.Self-Service Network

⇒ 오픈스택을 사용하는 사용자(Tenant)가 직접 자신만의 VM 인스턴스를 위한 네트워크를 구축할 수 있는 네트워크

⇒ 이 네트워크는 Provider Network를 기반으로 GRE, VXLAN 등의 터널링을 통해 구축된다.

### 3가지 네트워크 종류 연관지어 정리

- **Provider Network = External Network**
- **Self-Service Network = Tunnel Network + External Network**

# Openstack의 Neutron

## Openstack의 Plugin <Open vSwitch>

- Open vSwitch(OVS) : 클라우드 서비스 내 가상의 Network Bridge와 Flow rule을 사용하여 효율적으로 가상머신에 패킷을 포워딩하기 위한 가상의 스위치
- Neutron 프로젝트에서 가장 많이 사용되는 Plug-in이며 FLAT, VLAN, GRE 3가지 모드를 지원한다.

🤢 FLAT : 가상의 네트워크들이 L2 도메인 상에 공유한다.

🤢 VLAN : 각 가상의 네트워크들이 VLAN 태그로 그룹화한다.

🤢 GRE : 인스턴스들의 Traffic들이 GRE Tunnel을 통한 경로를 제공한다.

![Untitled](https://user-images.githubusercontent.com/67780144/98341705-a0ac8500-2052-11eb-9c12-0da9719cff35.png)

- OVS를 처음 설치하면 일종의 스위치 역할을 하는 하나의 Bridge가 생성된다. 그리고 vnet이라는 Port가 생성되고 각 Port는 Instanve들과 tap이라는 가상의 인터페이스를 통해 연결된다. 여기서 Bridge에는 Packet Flows라는 Flow Table이 있는데 여기에는 생성된 인스턴스들의 MAC 주소 저옵를 가지고 있으며 인스턴스에 대한 패킷을 컨트롤 할 때 활용된다.
- OVS는 크게 OVS와 사용자가 연결되는 User Space와 OVS와 각 Instanve 혹은 관련 모듈과 연결되는 Kernel Space 부분으로 구분된다. 처음 OVS를 설치하면 ovs-vswitchd라는 데몬이 실행되고 ovs db-server라는 프로세스가 실행된다. 사용자는 ovs-vsctl 이라는 커맨드를 통해 컨트롤하게 되면 해당 명령은 ovsdb-server를 거쳐 ovs-vswitchd를 통해 실제로 Kernel space에 있는 각 모듈들과 통신하게 된다.

⇒ OVS Plugin : Openstack의 인스턴스들을 가상의 Bridge로 연결하고 Packet Table을 통해 Instance에 대한 패킷을 포워딩

## OVS Plugin / VLAN 을 통한 Neutron 통신 과정

![Untitled 1](https://user-images.githubusercontent.com/67780144/98341709-a1ddb200-2052-11eb-879f-fe4bedde59c5.png)

1. VM에서 생성된 패킷은 VM의 가상의 NIC(eth0)에서 Linux Bridge의 가상 NIC(vnet)에 전달한다. 여기서 VM의 가상 NIC에서 Linux Bridge(qbr~)의 가상 NIC까지의 경로를 Tap Interface라고 한다.
2. Linux Bridge(qbr~)로 전달된 데이터 패킷은 veth-pair(qvb~, qvo~) Interface들을 지나 Compute Node의 내부 통합 Bridge (br-int)로 전달된다.

    (VLAN tag = 1, tag = 2로 그룹화되어 같은 br-int에 연결되어 있지만 서로 다른 그룹의 Subnet 임)

    🤢 Tap device인 vnet에서 바로 br-int로 물리지 않고 Linux Bridge(qbr~)를 거쳐 패킷이 지나간다?

    ⇒ VM과 OVS Bridge(br-int) 사이에 Linux Bridge를 넣어 Tap Device와 연결해 iptables이 Linux Device에서 수행되도록 함 (Linux Bridge아 Iptables을 Security Group라고 한다.)

3. br-int의 int-br-eth1 가상 NIC와 Compute Node 물리적 NIC(eth1)과 결합된 Bridge의 phy-br-eth1 가상 네트워크 인터페이스도 함께 veth-pair로 정외된다. br-int로 전달된 패킷은 veth-pair을 거쳐 br-eth1을 통해 Compute Node의 물리적 NIC(eth1)으로 전달된다.
4. Compute Node의 물리적 NIC(eth1)을 통해 나온 데이터 패킷은 Openstack의 Private Network를 구성하는 물리적 L2 Switch를 통해 Network Node의 물리적 NIC(eth1)로 전달된다.
5. Network Node의 물리적 NIC(eth1)로 전달된 패킷은 veth-path 인터페이스(phy-br-eth1 ~ int-br-eth1)를 거쳐 Network Node 내부 통합 Bridge (br-int)로 전달된다.

    [ br-eth1과 br-int는 Open vSwitch를 구성하는 Bridge들이다. ]

6. 내부 통합 Bridge(br-int)는 4개의 내부포트(tap~~~, qr~~~)들로 구성되며 각 포트는 IP주소가 각각 할당된다.
7. Network Node 내부 통합 Bridge(br-int)의 내부 포트 tapXXX, tapWWW에 dnsmasq(DHCP 서버 역할)가 연결된다.
8. 통합 Bridge(br-int)까지 전달된 데이터 패킷들은 qr-YYY, qr-ZZZ 내부 포트에서 qg-VVV 내부 포트로 L3 Agent를 통해 구현된 라우팅 과정을 통해 패킷들은 외부 Bridge(br-ex)로 전달된다.

    ( br-int와 br-ex의 내부 포트들은 dnsmasq를 통해 모두 IP 주소를 가지는데 이는 br-int와 br-ex 사이에 패킷이 이동할 때 라우팅을 통해 이동하기 때문임 )

9. Network Node의 외부 Bridge(br-ex)로 전달된 패킷은 Network Node의 물리적 NIC(eth0)을 통하여 외부망(인터넷)까지 전달된다.

## 다른 Network Logical Architecture

![Untitled 2](https://user-images.githubusercontent.com/67780144/98341712-a2764880-2052-11eb-8b75-973d031525d2.png)

### 주요 개념, Command

- linux_bridge
- vxlan

```bash
# Linux Bridge 확인
brctl show

# Linux 네트워크 네임스페이스
ip netns

# DHCP Namespace 생성
ip netns add
ip netns exec
ip link add

# 새로운 Bridge 생성
brctl addbr

# 인터페이스 추가
brctl addif

brctl show
ip link ....
```