vif = virtual network interface



h2. OVN의 흐름도

* ovn의 구성 데이터는 위(North)에서 아래(South)로 흐름
  * 1. CMS => (ovn/cms 플러그인 이용) => northbound database
    2. northbound database => ovn-northd
    3. ovn-northd가 configuration을 lower-level로 컴파일
    4. ovn-northd => southbound database
    5. southbound database => 모든 chassis로 설정 전달



* ovn의 상태 정보는 아래(South)에서 위로(North)

  * ovn-northd가 northbound database의 Logical_Switch_Port 테이블 "up" 열을 채운다.

  * Southbound Port_binding 테이블에 있는 논리적 포트의 "chassis"열이 

    * 비어있을 경우 false
    * 비어있지 않을 경우 true

  * 상태정보를 이용하여 CMS가 알 수 있는 것

    * VM의 네트워킹이 "up"된 시기를 알 수 있다.
    * CMS에서 전달한 configuration의 적용 여부에 대한 피드백을 알 수 있다.

    

* 시퀀스 번호 프로토콜
  * 1. CMS가 northbound 데이터 베이스의 설정을 업데이트
       * CMS에서 전달한 configuration이 적용된 시기를 알고자 할 경우
         * 동일한 트랜잭션의 일부로, NB_Global 테이블에서 NB_cfg 열의 값을 증가시킨다.
    2. ovn-northd가 지정된 northbound 데이터 베이스 스냅샷을 기반으로 southbound 데이터 베이스를 업데이트
       * northbound database의 데이터가 southbound database로 이동하는 시기를 알고자 할 경우
         * 동일한 트랜잭션의 일부로, nb_cfg를 northbound NB_Global에서 southbound 데이터 베이스의 SB_Global 테이블로 복사한다.
    3. ovn-northd가 southbound 데이터 베이스 서버로부터 변경사항이 커밋되었음을 확인
    4. northbound NB_Global 테이블의 sb_cfg를 새롭게 푸쉬된 nb_cfg 버전으로 업데이트한다.
       * CMS나 모니터링을 하는 사용자들은 southbound 데이터베이스에 연결하지 않고도 southbound 데이터베이스가 갱신되었는지 여부를 알아낼 수 있다.
    5. 각 chassis의 ovn-controller 프로세스는 업데이트된 nb_cfg와, 업데이트된 southbound database 정보를 받는다.
       * chassis의 Open vSwitch인스턴스에 설치된 물리적 흐름을 업데이트
    6. Open vSwitch로 부터 물리적 흐름이 업데이트 되었음을 확인
    7. southbound 데이터 베이스의 자체 chassis 레코드에서 NB_cfg를 업데이트
    8. ovn-northd는 southbound 데이터 베이스의 모든 chassis 레코드에 있는 nb_cfg 열을 모니터링
    9. 모든 레코드 중 최소값을 추적, NB_Global 테이블의 hv_cfg 열에 복사
       * CMS와 모니터링을 하는 사용자들은 하이퍼바이저들이 Southbound cfg에 맞춰 언제 갱신되었는지 알 수 있다. 



h2. Chassis setup

* chassis의 구성

  * ovn 전용 Open vSwitch로 구성

  * 시스템 startup 스크립트로 ovn-controller 시작 전에 브릿지 생성 가능

    * ovn-controller가 시작할 때 브릿지가 없으면 기본 구성으로 자동 브릿지가 생성된다.
    * 기본 구성( 통합 브릿지 포트 )
      * 1. tunnel port
           * 모든 chassis에서 ovn이 논리적 네트워크 연결을 유지하는 데에 사용하는 포트
           * OVN이 관리(추가, 업데이트, 제거)
        2. VIF
           * 하이퍼 바이저의 논리적 네트워크에 연결된다.
           * 하이퍼바이저 또는 Open vSwitch와 하이퍼바이저 간의 통합물이 vif를 관리한다.
           * OVS를 지원하는 하이퍼 바이저에 이미 존재하는 통합 작업으로, OVN의 현/새로운 기능이 아니다.
        3. Physical Port
           * gateway에서 논리적 네트워크 연결에 사용되는 포트
           * ovn-controller을 시작하기 전, 브릿지에 추가하는 포트
           * 보다 복잡한 설정을 할 경우, 다른 브릿지에는 이러한 포트가 물리적 포트가 아닌, patch 포트로 여겨질 수 있다. 

  * port 구성 시 주의할 점

    * 위의 기본 포트 이외의 포트는 통합 브릿지에 연결되어서는 안된다.

    * *특히* underlay 네트워크에 연결된 물리적 포트는 통합 브릿지에 연결해서는 안된다. 

      * 이러한 포트는 이와 분리된 Open vSwitch  브릿지에 연결해야 한다.

      * 결론 : 그 어떤 브릿지에도 연결할 필요가 없는 포트라는 뜻

        

  * 통합 브릿지 구성 방법

    * 각 설정에 대한 효과는  ovs-vswitchd.conf.db(5)에 보다 상세하게 설명되어있다.
    * fail−mode=secure
      * ovn-controller 시작 전에 독립된 논리 네트워크 간의 패킷 전환을 방지
    * other−config:disable−in−band=true
      * 통합 브리지에 대한 대역 내 제어 흐름을 억제
        * OVN은 원격 컨트롤러 대신 로컬 컨트롤러(Unix domain socket)를 사용하기 때문에 이러한 흐름이 나타나는 것은 드문 일
        * 그러나 종종 같은 시스템 내의 서로 다른 브릿지가 대역 내 원격 컨트롤러를 가지고 있을 수 있기 때문에, 이를 방지하고자 하는 설정





h2. Logical Networks



* OVN의 논리적 네트워크
  * 논리적 스위치 ( 이더넷 스위치의 논리적 버전 )
  * 논리적 라우터 ( IP 라우터의 논리적 버전 )
  * 물리적인 장비들과 마찬가지로 정교한 토폴로지로 연결될 수 있다.
  * 구현 방식
    * OVN에 속한 각 하이퍼바이저에서 분산된 방식으로 구현
    * 순수한 논리적 엔티티로, 물리적 위치에 연결/바인딩 되지 않는다.

* LSP(Logical Switch Port)
  * 논리적 스위치 내/외부 연결 지점
  * 가장 일반적인 유형의 port
    * VIF 논리 포트
      * VM 또는 컨테이너에 대한 연결지점을 나타냄
      * VM의 물리적 위치와 연결되며, VM이 마이그레이션 될 때 따라서 변경된다.
      * VM이 inactive(전원이 꺼지거나, 일시 중단된 상태)하더라도 연결 할 수 있다.
      * 물리적인 위치나 연결성과 관련없이 연결될 수 있다.

* LRP(Logical Router Port)
  * 논리적 라우터 내/외부 연결 지점
  * 논리적 라우터를 논리적 스위치/다른 논리적 라우터에 연결하기 위한 port
  * 논리적 스위치에 연결하여 VM, 컨테이너, 기타 네트워크 노드에 간접적으로 연결한다.



* 논리적 스위치와 논리적 라우터의 포트를 얘기할 때, "논리적 스위치 포트"나 "논리적 라우터 포트"로 언급하는 것이 좋다.
* 상세한 구분 설명 없이 "논리적 포트"라고만 언급되는 것은 대체로 "논리적 스위치 포트"를 의미한다.



* VM이 VIF 논리적 스위치 포트로 패킷을 보내는 순서
  * Open vSwitch flow 테이블이 패킷을 전송하는 논리적 스위치와 마주할 수 있는 다른 논리적 라우터 및 논리적 스위치를 고려한다.
  * 다른 논리적 라우터 및 논리적 스위치를 통한 패킷의 이동을 시뮬레이션한다.
    * 물리적 매체로 직접 패킷을 전송하지 않은 채로 시뮬레이션한다.
    * Flow 테이블에서 모든 스위칭 및 라우팅을 통한 결정과 동작을 구현한다.
  * 시뮬레이션 후, Flow 테이블에서 다른 하이퍼바이저에 연결된 논리 포트에서 패킷을 출력하기로 결정
    * 패킷은 이때, 물리적인 네트워크 전송을 위해 캡슐화된 후, 전송된다. 



h3. Logical Switch Port 종류

* LSP는 논리적 스위치를 논리적 라우터에 연결하고, 연결된 특정 LRP(Logical router port)를 peer로 지정합니다.
* VIF 포트는 가장 일반적인 종류의 LSP
  * OVN northbound 데이터 베이스에서, VIF의 포트 type은 정의되어 있지 않다.(empty string)
  * 이외의 포트 유형
    * localnet 논리적 스위치 포트
      * 논리적 스위치를 물리적 VLAN에 브릿지
      * localnet LSP가 있는 경우, 논리적 스위치는 단 하나의 LSP만 추가로 가질 수 있다.
        * 그래서 일부 gateway에서는 두 번째 포트로 라우터 포트를 지닌 논리적 스위치를 사용한다.
        * 만약 추가로 있는 LSP가 VIF인 경우
          * VIF를 포트로 가지면 논리적 스위치는 논리적 네트워크라고 볼 수 없다.
            * 논리적 스위치가 격리되지 않고, 물리적 네트워크에 브릿지 되어있으므로. 
            * 이런 경우, 독립적/중복 가능한  IP 주소와 네임스페이스 또한 설정이 불가능해진다.
            * 그러나 OVN 배포 시, 컨트롤 플레인의 장점과 특징을 활용하기 위해 이용할 때도 있다.
              * 장점과 특징 : 포트 보안, ACL(Access Control List) 등등
    * localport 논리적 스위치 포트
      * 특별한 종류의 VIF 논리적 스위치 포트
        * 특정 chassis에 바인딩 되는 것이 아니라, 모든 chassis에 존재한다.
        * localport를 통하는 트래픽
          * tunnel을 통해 전달되는 트래픽 X
          * 동일한 chassis로만 전달되는 트래픽 O
        * 비슷한 예시 : Openstack Neutron이 메타데이터를 전하는 방식
          * localport 포트를 이용하여 VM에 메타데이터를 제공
          * 메타데이터 프록시 프로세스가 모든 호스트의 localport 포트에 연결
          * 트래픽을 터널로 보낼 필요 없이, 프로세스가 포트에 연결되면 동일한 네트워크 내의 모든 VM이 동일안 IP/MAC 주소에 도달할 수 있게된다.

h3. 구현 디테일

* OVN이 내부적으로 구현되는 방법에 대한 디테일
* 논리적 데이터 경로 (Logical datapath)
  * OVN southbound 데이터 베이스에 있는 논리적 네트워크의 구현 세부사항
  * ovn-northd의 역할
  * Northbound 데이터베이스의 각 논리적 스위치, 라우터
    * southbound 데이터 베이스의 Datapath_Binding 테이블 값으로 변환
  * OVN Northbound 데이터베이스의 각 논리적 스위치 포트
    * southbound 데이터 베이스의 Port_Binding 테이블의 레코드로 변환
      * southbound Port_Binding 테이블 == northbound Logical_Switch_Port 테이블
      * Port_Biding 테이블에는 여러가지 유형의 logical port가 있으며, 대부분 Northbound LSP 유형과 일치한다.
      * 이 방식으로 처리되는 LSP 유형
        * VIF(정의되지 않은 타입 - empty string), localnet, localport, vtep 및 L2 gateway
* Port Binding 테이블 추가 설명
  * LSP 유형과 일치하지 않는 port binding 유형
    * Patch port binding
      * 논리적 패치 포트
      * 한 쌍으로 이루어진다.
      * 한쪽에 들어온 패킷은 다른 쪽으로 나온다.
      * ovn-northd가 논리적 스위치와 논리적 라우터를 함께 연결할 때 이용
      * 이 방식으로 처리되는 LSP 유형
        * vtep, L2 gateway, L3 gateway, chassis 리디렉션 유형의 포트 바인딩



h2. Gateway

* 논리적 네트워크와 물리적 네트워크 간의 제한된 연결을 제공



h3. VTEP Gateways

* VTEP이란?
  * VXLAN Tunnel Endpoint의 줄임말
  * OVN 논리적 네트워크를 Open vSwitch에 수반되는 OVSDB VTEP 스키마를 구현하는 물리적/가상 스위치에 연결
  * 주요 사용 사례
    * OVSDB VTEP 스키마를 지원하는 물리적 TOR 스위치를 사용하여 물리적 서버를 OVN 논리적 네트워크에 연결하는 것



h3. L2 Gateway

* chassis에서 사용할 수 있는 지정된 물리적 L2 세그먼트를 논리 네트워크에 연결
  * 물리적 네트워크를 논리적 네트워크의 일부로 만들 수 있다.
* L2 gateway 설정방법
  * CMS가 적절한 논리적 스위치에 L2 gateway LSP를 추가한다.
  * CMS가 LSP 옵션을 설정하여 바인딩해야하는 chassis의 이름을 지정한다.
  * ovn-northd가 위의 config를 southbound Port_Binding 레코드에 복사한다.
  * 지정된 chassis에서 ovn controller가 패킷을 물리적 세그먼트로 전달한다.
* L2 gateway port와 localnet port의 차이점
  *  localnet port
    * 물리적 네트워크가 하이퍼바이저 간의 전송 수단
  * L2 gateway
    * localnet과 달리 여전히 tunnel을 패킷 전송 수단으로 이용 가능
    * L2 gateway port를 물리적 네트워크에 있는 패킷을 전달해야할 때 사용할 뿐.
* L2 gateway와 VTEP gateway
  * 공통점
    * 가상화되지 않은 시스템을 논리적 네트워크에 추가하는 애플리케이션이다.
  * 차이점
    * VTEP gateway와 달리 L2  gateway에는 TOR 하드웨어 스위치의 지원이 필요하지 않다.



h3. L3 Gateway Router

* 일반적은 OVN 논리적 라우터는 분산 형태
  * => 한 곳에 구현되지 않고 모든 하이퍼바이저 chassis마다 구현되어 있다.
  * SNAT, DNAT과 같은 Stateful 서비스에는 알맞지 않은 형태이다.
    * 중앙 집중식으로 구현이 필요
    * => 이를 위한 보안책, L3 gateway router

* L3 gateway router?

  * 지정된 chassis에 구현된 OVN 논리적 라우터

  * 사용처

    * 분산 논리적 라우터와 물리적 네트워크 사이를 연결하는 곳에 이용

    * 연결되는 순서

      * 1. 하나의 하이퍼바이저 맨 뒷단에 논리적 스위치가 있고, 이 스위치는 분산 논리적 라우터와 연결된다.

      * 2. 분산 논리적 라우터는 또다른 논리적 스위치를 중간에 두고 gateway 라우터와 연결된다.

        * Join 논리적 스위치
          * 중간에서 라우터와 라우터를 연결해주는 스위치
          * Join 논리적 스위치로 두 라우터를 연결해주는 이유
            * OVN 구현체는 논리적 스위치와 연결된 상태의 gateway 논리적 라우터만을 지원한다.
            * 논리적 스위치를 이용하면 분산 라우터에 필요한 IP 주소의 갯수를 줄일 수 있다.

      * 3. gateway 라우터는 물리적 네트워크와 연결되어있는 localnet port를 가진 또다른 논리적 스위치와 연결한다. 

* L3 gateway router 구성 방법

  * 1. CMS가 라우터의 northbound Logical_Router 테이블의 "options:chassis"를 chassis의 이름으로 설정한다.
  * 2. ovn-northd가 인접 네트워크에 논리적 라우터를 연결하기 위해 southbound 데이터베이스에 있는 특정한 L3 gateway port binding을 이용한다.
  * 3. ovn-controller가 지정된 L3 게이트웨이 chassis에 바인딩된 포트로 패킷을 전송한다.
       * 로컬에서 처리하는 것이 아니다.



* 이외
  * DNAT 및 SNAT 규칙은 일대다 SNAT(IP 위장이라고도 함)를 처리할 수 있는 중앙 위치를 제공하는 게이트웨이 라우터에 연결할 수 있다.



h3. Distributed Gateway Port

*  Distributed Gateway Port란?
  * 하나의 특정한 chassis를 지정하도록 특별히 구성된 논리적 라우터 포트.
    * 중앙 집중식 처리를 위해 chassis를 지정한다.
    * 지정된 chassis는 *gateway chassis*라고 명칭한다.
  * localnet 포트를 사용하는 논리적 스위치에 연결해야한다.
  * 패킷 전송 여부
    * 분산 게이트웨이로 들어오고 나가는 패킷은 가능한 경우 gateway chassis를 포함하지 않고 처리된다.
    * 필요할 때는 추가 홉을 통해 처리된다.



* ovn-northd가 생성하는 두 가지 southbound Port_Binding 레코드
  * patch port binding
    * 가능한한 많은 트래픽에 이용되는 LRP를 위해 명명된 포트 바인딩
  * chassisredirect 타입의 포트 바인딩(cr-port)
    * 특화 기능이 있는 포트 바인딩
      * 패킷이 출력될 때, 흐름 테이블이 패킷을 gateway chassis로 전송한다.
        * 이후 패킷은patch port binding으로 자동 출력된다.
        * => 즉, gateway chassis에서 특정한 작업 수행이 필요할 때, flow table은 바로 path port binding으로 출력을 할 수 있다는 의미이다.
        * 이러한 방식의 출력 이외에 gateway chassis는 사용되지 않는다.
          * 예 : gateway chassis는 절대 패킷을 수신하지 않는다.



* CMS는 distributed gateway port를 3가지의 방법으로 설정할 수 있다.
* Distributed gateway port의 특징
  * 고가용성(high-availability)을 지원한다.
  * 둘 이상의 gateway chassis가 지정되어있는 경우
    * OVN은 한번에 하나의 gateway chassis만을 이용한다.
  * gateway 연결 여부를 모니터링하기 위해 BFD(Bidirectional Forwarding Detection)를 수행한다.
    * 사용가능하고, 우선순위로 놓여진 gateway를 위주로 통신하기 위하여 

***



h2. VIF의 생명주기

* VIF?
  * VM이나 하이퍼바이저 상에서 직접적으로 구동중인 컨테이너에 연결된 가상 네트워크 인터페이스
  * 아래 VIF 수명주기의 단계를 설명하는 부분은 OVN 및 OVN Northbound 데이터베이스 스키마에 대한 세부 정보를 참조

1. CMS 관리자가 CMS 사용자 인터페이스 또는 API를 사용하여 새 VIF를 생성 후 스위치에 추가

   * 이때, CMS는 자체 구성을 업데이트
     * 고유하고 영구적인 식별자 vif-id와 이더넷 주소 mac을 VIF와 연결하는 작업이 포함된다.

2. CMS 플러그인이 Logical_Switch_Port 테이블에 행을 추가

   2-1. OVN Northbound 데이터베이스를 업데이트하여 새 VIF를 포함

   * 새롭게 추가된 행을 구성하는 내용
     * name : vif-id
     * mac : mac
     * switch : OVN 논리적 스위치의 Logical_Switch 레코드
     * 나머지 행은 적절한 내용으로 초기화

3. ovn-northd가 OVN northbound 데이터베이스 업데이트를 수신

   3-1. OVN Southbound 데이터 베이스에 동일한 내용을 업데이트

   * 새로운 포트를 반영하기 위해서, Southbound 데이터베이스의 Logical_Flow 테이블에 새로운 행을 추가
     * 새 포트의 MAC 주소로 향하는 패킷이 전달되어야 함을 인식하는 흐름을 추가
     * 새 포트를 포함하도록 브로드 캐스트 및 멀티 캐스트 패킷을 전달하는 흐름을 업데이트.
   * Binding table에 레코드를 생성하고, chassis를 식별하는 열을 제외한 모든 열에 내용을 추가

4. 모든 하이퍼바이저 상에서, ovn-controller는 이전에 ovn-northd가 구성한 Logical_Flow table 업데이트 내용을 수신한다.

   * 그러나, VM의 전원이 꺼져있는 경우 ovn-controller는 제기능을 할 수 없다.
     * VIF가 어디에도 존재하지 않게 되기 때문에, VIF로부터 온 패킷의 송, 수신을 관리하는 것도 불가능해진다.

5. [사용자가 VIF를 포함한 VM의 전원을 킨다.]

   5-1.  전원이 켜진 VM의 하이퍼바이저에서 하이퍼바이저와 Open vSwitch 간의 통합이 VIF를 OVN 통합 브리지에 추가

   5-2. vif-id를 external_ids:iface-id에 저장

   * => 추가된 인터페이스가 새로은 VIF의 실체화임을 나타내기 위해서

6. 전원이 켜진 VM의 하이퍼바이저에서, ovn-controller는 external_ids:iface−id가 새로운 인터페이스 임을 감지한다.

   6-1. OVN Southbound DB의 Binding_table에 있는 chassis 컬럼의 행을 업데이트

   * 하이퍼바이저와 external_ids:iface−id의 논리적 포트를 연결하는 행

   6-2. 이후, ovn-controller는 VIF로부터 송수신 되는 패킷이 올바르게 전달될 수 있도록 로컬 하이퍼바이저의 OpenFlow table을 업데이트 한다.

7. Openstack을 포함한 몇 CMS 시스템은 네트워킹 서비스가 완전히 준비되었을 때만 VM이 실행될 수 있도록 한다.

   * 이와 같은 기능을 지원하기 위한 업데이트 내용 전달 방식

   * 1. ovn-northd가 Binding table 내용에 맞게 chassis 열이 업데이트 되었음을 확인
     2. VIF가 사용 가능함을 알리기 위해, OVN Northbound 데이터 베이스의 Logical_Switch_Port의 "up" 컬럼을 업데이트

     * CMS가 이와같은 기능을 이용한다면, VM 실행여부를 승인함으로써 업데이트에 대해 반응할 수 있을 것이다.

8. 모든 하이퍼바이저 중에서도, VIF가 위치한 하이퍼 바이저에서, ovn-controller가 완전히 채워진 Binding table의 행을 감지

   * 이것을 통해 ovn-controller가 논리적 포트의 물리적 위치를 감지
     * 각 인스턴스들이 각 스위치의 OpenFlow table(을 업데이트 함으로써, VIF로부터의 패킷을 터널을 통해 올바르게 송수신할 수 있다.
       * 이때, OpenFlow table은 OVN DB Logical_Flow table의 논리적 데이터 경로 흐름에 기반한다.

9. [사용자가 VM의 전원을 끈다.]

   * OVN 통합 브릿지에서 VIF가 삭제된다.

10. 전원이 꺼진 VM이 올라가있는 하이퍼 바이저에서, ovn-controller가 VIF가 삭제되었음을 감지

    * 논리적 포트에 대한 Binding table에서 chassis 컬럼을 삭제한다.

11. 논리적 포트에 대한 Binding table에서 chassis 컬럼이 비었다는 것을 모든 하이퍼 바이저의 ovn-controller가 감지한다.

    * 이것은 ovn-controller가 더이상 논리적 포트의 물리적 위치를 알지 못한다는 뜻이다.
    * 각 인스턴스는 이를 반영하여 OpenFlow table을 업데이트한다.

12. VIF가 더이상 사용되지 않는다면, CMS 관리자는 사용자 인터페이스/API를 이용하여 VIF를 삭제한다.

    * CMS도 자체 구성을 다시 업데이트한다

13. CMS 플러그인이 OVN Northbound 데이터베이스로부터 VIF를 삭제한다.

    * 삭제 방법 : OVN Northbound 데이터베이스의 Logical_Switch_Port 테이블의 열 삭제

14. ovn-northd가 OVN Northbound의 업데이트 내용을 수신한다.

    14-1. 이에 맞게 OVN Southbound의 데이터베이스도 업데이트한다. 

    * 업데이트 방식 : 방금 삭제된 VIF와 관련된 OVN Southbound 데이터베이스의 Logical_Flow 테이블과 Binding 테이블의 열 삭제

15. 모든 하이퍼바이저에서, 이전 단계에 ovn-northd가 생성한 Logical_Flow 테이블을 수신한다.

    15-1. ovn-controller는 이를 반영하여 OpenFlow table을 업데이트한다.

    * 사실 VIF가 삭제된 이상, ovn-controller가 새롭게 업데이트할 내용은 많지 않다.

***

h2. VM 내 컨테이너 인터페이스의 수명주기



* OVN은 가상 네트워크 추상화를 제공

  * OVN_NB 데이터베이스에 기록된 정보를  각 하이퍼바이저의 OpenFlow로 변경하여 제공

  * OVN을 이용한 멀티 테넌트용 보안 가상 네트워킹을 사용할 수 있는 상황
    * ovn-controller가 Open vSwitch 흐름을 수정할 수 있는 유일한 엔티티일 경우 제공될 수 있다.
    * 그러므로, 만약 하이퍼바이저 내에 Oven vSwitch 통합 브릿지가 설정되어있는 경우
      *  VM 내의 테넌트 워크로드는 Open vSwitch의 흐름을 어떤 것도 수정할 수 없을 것이라고 볼 수 있다. 



* 컨테이너 실행이 가능한 전제조건
  * 1. 인프라 공급자가 컨테이너 내의 애플리케이션을 신뢰
  * 2. Open vSwitch의 흐름에 컨테이너의 오류가 영향을 주지 않는다.
  * 이 전제 조건은 다음의 상황에서도 동일하게 적용된다.
    * VM 내의 ovn-controller에 의해 흐름이 추가된 Open vSwitch 통합 브릿지 상에서 실행되는 컨테이너
  * 위와 같은 실행과 관련된 수명주기는 VIF의 수명주기와 같다.



* 앞으로 설명되는 단계들은 컨테이너 인터페이스의 순환주기이다.(CIF - container interface)
  * 하이퍼 바이저 상에 Open vSwitch 통합 브릿지가 설정되어있고, 이러한 하이퍼 바이저에서 구동되는 VM에서 컨테이너가 생성되었을 때의 순환주기
  * 위의 상황에서 실행되는 컨테이너들은 모두 애플리케이션에 문제가 생기더라도 다른 테넌트들에게 영향을 줄 수 없는 컨테이너들이다.



* VM 내에서 다중 컨테이너가 생성되었을 때
  * CIF 또한 다중 생성된다.
    * 가상 네트워크 추상화를 지원하기 위해, Open vSwitch는 CIF들에 도달할 수 있어야 한다.
    * OVN은 서로 다른 CIF들을 각각 구분할 수 있어야 한다.
    * CIF의 네트워크 트래픽을 구분하는 방법
      * 1. 각 CIF 마다 VIF를 제공한다.( 1:1 방식 )
           * 하이퍼바이너 내에 다수의 네트워크 장비가 존재할 수 있다는 것을 의미
           * OVS의 속도가 느려지고, VIF 관리를 위한 추가적인 CPU가 필요할 것이다.
           * 컨테이너를 생성하는 entity가 컨테이너에 맞는 VIF를 생성할 수 있어야 한다.
        2. 모든 CIF에 대한 VIF를 생성한다.( 1:다 방식 )
           * 각 패킷마다 기록된 tag를 이용하여 각기 다른 CIF를 구분할 수 있다.
           * OVN이 선택한 방식은 1:다 방식이고, VLAN을 tagging에 사용한다.

* CIF 순환주기

  * 1. [시작]

    * VM을 생성한 CMS, 혹은 VM을 소유한 테넌트, 혹은 VM을 생성한 CMS가 아닌 또다른 컨테이너 오케스트레이션 시스템에 의해 컨테이너가 시작될 때, CIF가 시작한다.
    * entity가 알아야하는 것
      * vif-id
        * 컨테이너 인터페이스가 통과해야하는 VM의 네트워크 인터페이스와 연결되어 있다.
      * unused VLAN
        * CIF 생성을 위해 VM에서 사용되지 않는 VLAN을 골라야한다.

  * 2. 컨테이너 생산 entity가 새로운 CIF를 포함하도록 OVN northbound를 업데이트한다.
       * Logical_Swtich_Port 테이블에 행을 추가하는 방식
       * 행에 기록되는 요소
         * name : 고유 식별자
         * parent_name : vif-id
         * tag : VLAN tag

  * 3. ovn-northd가 OVN Northbound 데이터베이스의 업데이트 내용을 수신한다.
       * OVN Southbound 데이터 베이스에 동일한 내용을 업데이트
         * 새로운 포트를 반영하기 위해서, Southbound 데이터베이스의 Logical_Flow 테이블에 새로운 행을 추가
         * Binding table에 새로운 행을 생성하고, chassis를 식별하는 열을 제외한 모든 열에 내용을 추가

  * 4. 모든 하이퍼바이저에서 ovn-controller는 Binding table의 변경사항을 구독
       * Binding table의 parent_port 열 값을 포함하는 새로운 행이 생성된다면, 
         * external_ids:iface−id의 vif-id와 동일한 값을 가진 OVN 통합 브릿지가 있는 하이퍼바이저의 ovn-controller가 로컬 하이퍼 바이저의 OpenFlow 테이블을 업데이트한다.
         * 특정 VLAN 태그가 달린 VIF 패킷을 올바르게 다루기 위함
       * 이후, 물리적 위치를 반영하기 위해 Binding table의 chassis 컬럼을 업데이트한다.

  * 5. 기반 네트워크가 모두 준비된 후, 컨테이너의 애플리케이션을 실행할 수 있다.
       * ovn-northd가 Binding table의 업데이트된 chassis 컬럼을 감지
       * CIF가 구동되었음을 알리기 위해 OVN northbound 데이터베이스의 Logical_Swtich_Port 테이블의 "up" 컬럼을 업데이트
       * entity는 이 값을 쿼리하여 값의 상태에 맞게 애플리케이션을 실행할 책임이 있다.

  * 6. [정지]
       * 컨테이너를 생성, 실행한 entity가 컨테이너를 중지한 단계
       * CMS를 통해 entity는 Logical_Switch_Port의 행을 삭제한다.

  * 7. ovn-northd가 OVN Northbound의 업데이트를 수신한다.
       * 이에 맞게 OVN Southbound 데이터 베이스를 업데이트
         * 삭제된 CIF와 관련된 OVN Southbound 데이터 베이스의 Logical_Flow 테이블의 행을 삭제하거나 업데이트
         * Binding table에서도 CIF와 관련된 행을 삭제한다.

  * 8. 모든 하이퍼바이저에서, ovn-controller가 Logical_Flow 테이블의 업데이트 내용을 수신한다.
       * ovn-controller가 이를 반영하여 OpenFlow 테이블을 업데이트한다.

***

h2.  패킷의 구조 물리적 수명주기



* OVN을 통해 한 가상시스템/컨테이너에서 다른 가상 시스템으로 이동하는 방법에 대해 설명한다.
* 패킷의 물리적 처리에 초점을 맞추며, 먼저 데이터/ 메타데이터 필드에 대해 설명한다.



* tunnel key
  * OVN이 Geneve나 다른 터널에서 패킷을 캡슐화할 때,
    * 추가 데이터를 덧붙이고 OVN 인스턴스가 패킷을 올바르게 처리할 수 있도록 해주는 key
  * 특정 캡슐화에 따라 다른 형태를 취할 수 있다.
* 논리적 데이터경로 필드
  * 패킷이 처리되는 논리적 데이터 경로를 나타내는 필드
  * OVN은 논리적 데이터 경로를 저장하기 위해, OpenFlow 1.1+에서 명칭되는 메타데이터를 사용한다.
    * 이 필드값은 패킷 송수신 시, tunnel key의 일부로 터널을 통과하게 된다.
* 논리적 입력 포트 필드
  * 패킷이 논리적 데이터 경로로 들어간 논리적 포트를 나타내는 필드
  * OVN은 이 필드값을 Open vSwtich 확장 레지스터 번호 14에 저장한다.
  * Geneve와 STT 터널은 이 필드를 tunnel key의 일부로써 통과하게 된다.
  * VXLAN은 논리적 입력 포트 필드를 명시적으로 전달하지 않는다.
    * 따라서 OVN은 VXLAN을 오로지 gateway와 통신할 때만 이용한다.
    * 이때, OVN 관점에서 gateway는 단일 논리적 포트로 구성되어있다.
    * OVN은 OVN 논리적 ingress 파이프라인에서 수신할 때 이 단일 포트에 논리적 입력 포트 필드를 설정한다.
* 논리적 출력 포트 필드
  * 논리적 데이터패스에서 패킷이 출력되는 포트를 나타내는 필드
  * 논리적 ingress 파이프라인에서 시작할 때 0으로 초기화된 값으로 시작한다.
  * OVN은 이 필드값을 Open vSwtich 확장 레지스터 번호 15에 저장한다.
  * 입력 필드와 마찬가지로, Geneve와 STT 터널은 이 필드를 tunnel key의 일부로써 통과하게 된다.
  * VXLAN이 논리적 출력 포트 필드를 tunnel key안에 포함시켜 송신하지 않기 때문에, 
    * OVN 하이퍼바이저에 의해 VXLAN 터널로 부터 패킷이 수신되면
    * 패킷은 출력 포트를 지정하기 위해 table 8에 다시 제출된다.
    * 패킷이 32번 테이블에 도달하면, VXLAN 터널에서 패킷이 도착하였을 때 설정되는 MLF_RCV_FROM_VX 태그를 확인한 뒤
      * 로컬 전송을 위해 테이블 33에 다시 제출된다.
* 논리적 포트를 위한 conntrack 존 필드
  * 논리적 포트의 연결 추적 영역을 나타내는 필드
  * 로컬에서만 유의성을 가지고, chassis 간에는 무의미한 값이다.
  * 논리적 ingress pipeline이 시작할 때 0으로 초기화된다.
  * OVN은 이 필드값을 Open vSwtich 확장 레지스터 번호 13에 저장한다.
* 라우터를 위한 conntrack 존 필드
  * 라우터의 연결 추적 영역을 나타내는 필드
  * 로컬에서만 유의성을 가지고, chassis 간에는 무의미한 값이다.
  * OVN은 이 필드값 중 DNAT에 대한 정보는 Open vSwtich 확장 레지스터 번호 11에 저장한다.
  * OVN은 이 필드값 중 SNAT에 대한 정보는 Open vSwtich 확장 레지스터 번호 12에 저장한다.
* 논리적 흐름 플래그
  * 후속 테이블에서 일치되는 규칙을 결정해야할 때, 테이블 간 context 유지를 관리하기 위한 플래그이다.
  * 로컬에서만 유의성을 가지고, chassis 간에는 무의미한 값이다.
  * OVN은 논리적 플래그를 Open vSwitch 확장 레지스터 번호 10에 저장한다.
* VLAN ID
  * VLAN ID는 OVN과 VM 내부에 중첩된 컨테이너 간의 인터페이스로 사용된다.
  * 자세한 내용은 VM 내부의 컨테이너 인터페이스 수명 주기 참조.

***

* [시작] : ingress 하이퍼바이저 상의 VM/컨테이너가 OVN 통합 브릿지에 연결된 포트로 패킷을 전송한다.

* 1. OpenFlow 0번 테이블이 물리적 정보를 논리적 정보로 변환한다.

     * 변환된 정보는 패킷의 수신 포트와 일치한다.

     1-1. 패킷이 통과하고 있는 논리적 데이터 경로를 식별하도록 논리적 데이터패스 필드를 설정

     1-2. 수신 포트를 식별하는 논리적 입력 포트 필드를 설정

     1-3. 위의 작업을 마친 후 논리적 메타데이터로 패킷에 주석을 추가

     1-4. 이후, 논리적 ingress pipline에 접근하기 위해 패킷을 테이블 8로 전송한다.

     * VM에 중첩된 컨테이너에서 생성된 패킷은 약간 다른 방식으로 처리된다.
       * 컨테이너는 VIF별 VLAN ID 값으로 구분될 수 있다.
       * 물리=>논리적 변환 작업이 일어날 때, 
         * VLAN ID 매칭 작업이 추가적으로 진행된다.
         * VLAN 헤더를 분리하는 작업이 이루어진다.
     * 0번 테이블은 다른 chassis로부터 수신되는 패킷도 처리한다.
       * ingress port(tunnel) 정보로 다른 패킷과 구분한다.
       * 패킷이 OVN 파이프 라인에 수신되는 패킷과 같이
         *  패킷에 논리적 출력 포트 필드값과 논리적 입력 포트 필드값을 메타데이터로 하여 주석을 단다.
       * 논리적 출력 포트 필드값도 이 작업을 할 때 함께 설정된다.
         * OVN에서 tunneling이란 논리적 출력 포트값을 알게된 후 진행되는 것이므로, 가능한 작업이다.
     * 이와 같은 세가지 방식의 정보가 tunnel 캡슐화 메타데이터를 통해 알 수 있는 정보다.

     1-5. 각 정보를 알게된 후, logical egress pipline으로 패킷을 보내기 위해 33번 테이블에 패킷을 재전송하게 된다.

     

