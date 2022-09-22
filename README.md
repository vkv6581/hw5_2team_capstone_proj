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
## 분석 설계 (공통)

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
## 이벤트스토밍 (공통)
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
## 사전 준비 - kafka 생성 및 모니터링 (kafka)

```diff
version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - 22181:2181
  
  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - 9092:9092
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      
+   kafka-ui:
      image: provectuslabs/kafka-ui
      container_name: cluster-kafka-ui
      ports:
        - "9500:8080"
      environment:
        KAFKA_CLUSTERS_0_NAME: local
        KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: PLAINTEXT://kafka:29092
        KAFKA_CLUSTERS_0_ZOOKEEPER: "zookeeper:2181"
      depends_on:
        - zookeeper
        - kafka
```
- 기본 제공된 kafka에 kafka-ui를 추가하여 kafka를 모니터링하였음.
![image](https://user-images.githubusercontent.com/23250734/191666106-e8b6c957-fb70-4d92-8205-456f1530b61b.png)


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

#### 이벤트 발행 소스
Order.java의 주문 취소 소스.
```
    //Order.java에서 주문이 취소되었을 때 (DELETE 요청) OrderCanceled이벤트 발행.
    @PostRemove
    public void onDeletePersist() {
        //order취소 시 orderCanceled 이벤트 발행.
        OrderCanceled orderCanceled = new OrderCanceled(this);

        orderCanceled.setId(this.id);
        orderCanceled.setOrderStatus("CANCELED");

        orderCanceled.publishAfterCommit();
    }
```


#### 이벤트 수신 소스
Payinfo.java의 주문 취소 이벤트 수신 소스.
```
    //주문 취소 (OrderCanceled 이벤트 수신)
    public static void payCancel(OrderCanceled orderCanceled) {
    
        //취소된 주문 ID로 pay정보 검색 후 상태 업데이트
        Payinfo payinfo = repository().findByOrderId(orderCanceled.getId());
        if(payinfo != null) {
            payinfo.setStatus("ORDER_CANCELED");
            repository().save(payinfo);
            
            //상태 업데이트 후 결제취소 이벤트 발행.
            PaymentCanceled paymentCanceled = new PaymentCanceled(payinfo);
            paymentCanceled.publishAfterCommit();
        }
    }
```

Store.java의 주문 취소 이벤트 수신 소스.
```
    public static void orderRecevie(OrderCanceled orderCanceled) {

        //주문 취소 시 관련 store 취소처리.
        Store store = repository().findByOrderId(orderCanceled.getId());
        if(store != null) {
            store.setOrderStatus(orderCanceled.getOrderStatus());
            repository().save(store);
            
            StoreCanceled storeCanceled = new StoreCanceled();
            storeCanceled.setOrderId(orderCanceled.getId());
            storeCanceled.publishAfterCommit();
        }
    }
```

#### 테스트

주문 취소 호출. (3번 주문 취소)
```
http DELETE localhost:8081/orders/3
```
![image](https://user-images.githubusercontent.com/23250734/191670387-a3b7752a-68c1-488b-9edd-d7cde77af7a5.png)

kafka 이벤트 로그. (이벤트 수신까지 포함.)
주문 취소 -> 결제 취소 / 상점 취소까지 연속적으로 이루어진 것을 확인 가능.
![image](https://user-images.githubusercontent.com/23250734/191670577-ae74f15e-d0d6-49b1-b2dc-61a6b1eaf3d1.png)

3번 주문과 연결된 상점 정보 취소된 것 확인 가능. 
![image](https://user-images.githubusercontent.com/23250734/191671603-cab72c63-3625-44da-a74f-41844c3a2e44.png)

3번 주문과 연결된 결제정보또한 취소됨.
![image](https://user-images.githubusercontent.com/23250734/191671979-196c1513-1073-41e8-92e2-0a26725a7d8a.png)

--------------------------------------------------
## CQRS

```
명령 및 쿼리 역할 구분 CQRS (Command and Query Responsibility Segregation)
데이터를 조회하는 부분을 분리하는 것으로 이해.
데이터가 중복으로 저장되는것을 감안하고, 비즈니스 로직과 데이터를 조회하는 영역을 분리하는 것.
이번 실습에선 dashboard서비스를 생성하여, 주문의 전체적인 상태를 조회할 수 있도록 CQRS 생성.
```
#### Dashboard 서비스
```
최초 주문이 되었을 때, 생성되는 별도의 객체 (주문과 1:1 맵핑)
거의 모든 이벤트를 수신하여 이벤트에 따라 주문의 상태, 날짜 등을 업데이트하여 화면에 보여줌
```

이벤트들을 수신하여 Dashboard 정보를 변경하는 소스 - DashBoardViewHandler.java
```diff
+   //Ordered(주문됨) 이벤트 발생 시 Dashboard 객체 신규 생성.
    @StreamListener(KafkaProcessor.INPUT)
    public void whenOrdered_then_CREATE_1(@Payload Ordered ordered) {
        try {
            if (!ordered.validate()) return;

            // view 객체 생성
            Dashboard dashboard = new Dashboard();
            // view 객체에 이벤트의 Value 를 set 함
            dashboard.setOrderId(ordered.getId());
            dashboard.setStoreName(ordered.getStoreName());
            dashboard.setItemName(ordered.getItemName());
            dashboard.setItemQty(ordered.getItemQty());
            dashboard.setPrice(ordered.getPrice());
            dashboard.setStatus("ORDERED");
            dashboard.setOrderDt(ordered.getOrderDate());
            // view 레파지 토리에 save
            dashboardRepository.save(dashboard);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
+   //주문이 취소되었을 경우 취소 상태로 업데이트하는 소스. 
    @StreamListener(KafkaProcessor.INPUT)
    public void whenOrderCanceled_then_UPDATE_4(
        @Payload OrderCanceled orderCanceled
    ) {
        try {
            if (!orderCanceled.validate()) return;
            // view 객체 조회
            Optional<Dashboard> dashboardOptional = dashboardRepository.findById(
                orderCanceled.getId()
            );

            if (dashboardOptional.isPresent()) {
                Dashboard dashboard = dashboardOptional.get();
                // view 객체에 이벤트의 eventDirectValue 를 set 함
                dashboard.setStatus("ORDER_CANCELED");
                // view 레파지 토리에 save
                dashboardRepository.save(dashboard);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

#### 결과 확인
주문 시작

![image](https://user-images.githubusercontent.com/23250734/191677781-e6d25546-eae9-4aa3-a727-5e9f6dae0c9c.png)

특정 주문 조리완료

![image](https://user-images.githubusercontent.com/23250734/191677861-5e4b13b1-6779-45a1-a837-22422b93c738.png)

대쉬보드 상태 변경된 것 확인

![image](https://user-images.githubusercontent.com/23250734/191677704-e76ac785-2fb4-49c8-97c7-9e2431900260.png)

--------------------------------------------------
## gateway
```
여러 마이크로서비스를 실행 시, 각 마이크로서비스로의 진입점이 달라 포트 번호를 전부 지정해서 호출해야 하는 문제를 해결하기 위해 gateway사용.
이번 실습에선 spring-cloud에서 제공하는 gateway기능을 통해 구현하였음.
```

gateway 생성을 위한 application.yml 파일.
```diff
server:
+  port: 8088   //gateway 포트. gateway포트로 들어오면 하위 URL에 따라 각각의 서비스로 redirect시켜줌.

---
spring:
  profiles: default
  cloud:
    gateway:
      routes:
+      //아래 routes경로에 서비스들의 정보를 입력. /orders/~~가 포함된 URL로 gateway에 접근하면, 8081 포트에 떠 있는 order서비스로 redirect시켜줌.
+        - id: order
+          uri: http://localhost:8081
+          predicates:
+            - Path=/orders/**, 
        - id: store
          uri: http://localhost:8082
          predicates:
            - Path=/stores/**, 
        - id: delivery
          uri: http://localhost:8083
          predicates:
            - Path=/deliveries/**, 
        - id: payment
          uri: http://localhost:8084
          predicates:
            - Path=/payinfos/**, 
        - id: dashboard
          uri: http://localhost:8085
          predicates:
            - Path=, /dashboards/**
+            //특정 경로가 없을 시 8080포트로 띄워진 프론트엔드 호출.
+        - id: frontend
+          uri: http://localhost:8080
+          predicates:
+            - Path=/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true
```

테스트
![image](https://user-images.githubusercontent.com/23250734/191675243-a11c2059-c4c4-44e8-b172-94f1c6a6517b.png)

![image](https://user-images.githubusercontent.com/23250734/191675583-68e5f227-7634-423e-b18a-7a31d6c2288a.png)

