# msa-capstone-project
## 🕙 Schedule

- 일자별 진행

  |   일자    | 진행           | 내용                                |
  | :-------: | :------------- | :---------------------------------- |
  | 09/21 AM  | OJT            | 과정설명<br>과제수행환경설명        |
  | 09/21 PM  | Brain Storming | msaez.io                            |
  | 09/22 ALL | Team Project   | 팀별과제 수행                       |
  | 09/23 AM  | Team Project   | 팀별과제 수행                       |
  | 09/23 PM  | Wrap Up        | 과제 제출<br>시작:14시<br>마감:16시 |

## 👫 Team

  | 팀  |   성명   | 직급 | 소속             |
  | :-: | :------: | :--: | :--------------- |
  |  2  | 🎖 최원식 | 과장 | 서비스운영2팀  |
  |     |  황상식 | 과장 | 디지털워크그룹  |
  |     |  김영준  | 대리 | Platform개발팀   |
  |     |  이재영 | 대리 | 디지털워크그룹  |

## ✏️ Evaluation

- 분석설계
- SAGA Pattern (Pub / Sub)                           : 공통
- CQRS Pattern                                       : 
- Correlation / Compensation(Unique Key)             :
- Request / Response (Feign Client / Sync.Async)     : Order
- Gateway                                            : 김영준 대리
- Deploy / Pipeline                                  : 공통
- Circuit Breaker                                    : Payment
- Autoscale(HPA)                                     : 공통
- Self-Healing(Liveness Probe)                       : 공통(deployment.yaml에 작성)
- Zero-Downtime Deploy(Readiness Probe)              : 공통(deployment.yaml에 작성)
- Config Map / Persistence Volume                    : 
- Polyglot

## 📑 To-Do

- <a href="https://www.msaez.io/#/" target="_blank">Brain Storming</a>

  - 팀별로 주제 선정 및 이벤트 스토밍 진행

- GitHub : [https://github.com/seonguk9303/hw5_capstone_proj]
- GitPod
  - Github 계정 및 Repositoy(public) 준비 ( **for FORK** )
  - gitpod.io/#/{Github-Repository-URL} or Browser Extension 설치(https://www.gitpod.io/docs/browser-extension)
  - Collaboration & Sharing
    - 팀장 : github.com > repository > Settings > Collaborators > Add People ; 팀원초대
    - 팀장 / 팀원 : gitpod.io > Settings > Integrations > GitHub > Edit Permissions > Public_repo Check ; GitPod - GitHub 권한설정
  * gitpod 초기 연동시 필요한 라이브러리들이 없는 상태이며 **.gitpod.yml** 파일에 선언한 명령어들 자동 실행됨
  * 실행 안되는 명령어들이 있으면 직접 설치
- AWS (_약 15~20분 소요_)
  - AWS IAM 계정(MSA5차수).xlsx 참고
  - Region-Code : 메일 내 Region
  - Cluster-Name : Account-Id
  - Image-Repository-Name : Account-Id

--------------------------------------------------
## 분석 설계

### 배달의민족

#### 기능적 요구사항
1. 고객이 메뉴를 선택하여 주문 후 결제한다.
2. 주문이 되면 주문 내역이 입점상점주인에게 전달된다
3. 상점주인이 확인하여 요리해서 배달 출발한다
4. 고객이 주문을 취소할 수 있다
5. 고객이 주문상태를 중간중간 조회한다

#### 비기능적 요구사항
1. 트랜잭션
결제가 되지 않은 주문건은 아예 거래가 성립되지 않아야 한다 (Sync 호출)

2. 장애격리
상점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency
결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 Circuit breaker, fallback

3. 성능
고객이 자주 상점관리에서 확인할 수 있는 배달상태를 주문시스템(프론트엔드)에서 확인할 수 있어야 한다 CQRS

--------------------------------------------------
## 이벤트스토밍
- 이벤트스토밍 결과
![image](https://user-images.githubusercontent.com/23250734/191643996-1c8d90db-0506-4f55-ad9e-e4a85bd4d625.png)

- 주문 -> 배달 프로세스
![image](https://user-images.githubusercontent.com/23250734/191655483-b91ea937-f779-4d52-8047-ec465fb21c03.png)

```
1. 고객이 음식을 주문함과 동시에 결제가 진행된다. (트랜잭션)
2. 음식점에서 주문 정보를 확인한다
3. 조리를 시작하고 음식 조리가 완료된다.
4. 배달원에 의해 배달이 시작된다.
5. 배달원이 음식을 픽업한다.
6. 배달원이 음식 배송을 완료한다.
7. 주문 전체 과정을 대시보드를 통해 사용자가 조회한다.
```

- 트랜잭션 
![image](https://user-images.githubusercontent.com/23250734/191655924-98f7cff8-3a0c-4aab-b04f-2de851d1d7a7.png)
```
주문 -> 결제 과정을 req/res로 처리하여 트랜잭션 처리하였다.
결제가 이루어지지 않으면 주문은 성립되지 않는다.
```
- 이벤트 처리
```
주문 -> 결제를 제외한 모든 액션은 이벤트 방식으로 처리하였다.
상점 관리 기능이 수행되지 않더라도 주문을 받을 수 있으며, kafka를 통해 비동기 처리된다.
```

--------------------------------------------------
## SAGA Pattern (Pub / Sub)
```
SAGA 패턴은 MSA 개발 환경에서, 분산된 여러 서비스들 간 데이터의 일관성을 지키기 위한 방법 중 하나.
이번 교육에선 이벤트 pub/sub를 통한 SAGA 패턴에 대해 학습하였음.
이번 배달의민족 사례에선 사용자가 주문을 취소하였을 때 case로 테스트를 진행.
```
- 이벤트스토밍 프로세스
![image](https://user-images.githubusercontent.com/23250734/191657494-dc6fb681-1c6b-4e0c-8bd2-9f62e3f987e2.png)
```
사용자가 주문을 취소하면, 결제/상점 주문이 동시에 취소되어야 한다.
따라서 주문 취소 이벤트를 각 서비스에서 수신받아, 각 서비스의 주문 상태를 업데이트한다. (비동기 일관성 유지)
```
