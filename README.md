![image](https://user-images.githubusercontent.com/70673885/97950284-bcf1bd00-1dd9-11eb-8c8a-b3459c710849.png)


# 서비스 시나리오

기능적 요구사항
1. 고객의 주문이 결제 완료가 되면 결제 금액에 따라 페이백 포인트를 제공한다.
1. 결제가 취소되면 제공한 프로모션도 취소처리가 되어야 한다.

비기능적 요구사항
1. 트랜잭션
    1. 페이백 포인트 회수는 "반드시" 결제 취소가 되어야 한다. >> Sync 호출
    1. 결제가 완료되면 페이백 포인트가 적립되고 주문정보에 업데이트 되어야 한다. >> SAGA, 보상 트랜젝션
1. 장애격리
    1. 프로모션 시스템이 수행되지 않더라도 결제는 365일 24시간 처리되어야 한다. >> Async
    1. 프로모션 시스템이 과중되면 결제 취소가 잠시 후에 진행되도록 유도한다. >> Circuit break, fallback
1. 성능
    1. 고객이 모든 진행내역과 프로모션 내역을 조회 할 수 있도록 성능을 고려하여 별도의 view로 구성한다. >> CQRS
    


# 체크포인트

1. Saga
1. CQRS
1. Correlation
1. Req/Resp
1. Gateway
1. Deploy/ Pipeline
1. Circuit Breaker
1. Autoscale (HPA)
1. Zero-downtime deploy (Readiness Probe)
1. Config Map/ Persistence Volume
1. Polyglot
1. Self-healing (Liveness Probe)


# 분석/설계


## AS-IS 조직 (Horizontally-Aligned)
  ![ASIS조직](https://user-images.githubusercontent.com/73939028/98253021-e703d480-1fbd-11eb-9941-592810aa27cd.JPG)

## TO-BE 조직 (Vertically-Aligned)
  ![TOBE조직](https://user-images.githubusercontent.com/73939028/98253034-e9662e80-1fbd-11eb-9602-97cb585fe1f7.JPG)


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/dukpFsKpNqU89oVjZp4DTcUnsXM2/mine/bf907bfa5a2f93c641a3a90fbedcfcd0/-MLNMu6MRdNcvs-zvjyn


### 이벤트 도출
![이벤트도출](https://user-images.githubusercontent.com/73939028/98252952-d94e4f00-1fbd-11eb-9281-7c0475aa6055.JPG)

### 부적격 이벤트 탈락
![부적격이벤트탈락](https://user-images.githubusercontent.com/73939028/98253062-ef5c0f80-1fbd-11eb-88d4-f29819bd2a44.JPG)

    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
	- 폰종류가선택됨, 결제버튼클릭됨, 배송수량선택됨, 배송일자선택됨  :  UI 의 이벤트이지, 업무적인 의미의 이벤트가 아니라서 제외
	- 배송취소됨, 메시지발송됨  :  계획된 사업 범위 및 프로젝트에서 벗어서난다고 판단하여 제외
	- 주문정보전달됨  :  주문됨을 선택하여 제외
	

### 액터, 커맨드 부착하여 읽기 좋게
![액터커맨드부착](https://user-images.githubusercontent.com/73939028/98253088-f551f080-1fbd-11eb-97f0-55bb5f0e4d18.JPG)

### 어그리게잇으로 묶기
![어그리게잇으로묶기](https://user-images.githubusercontent.com/73939028/98253094-f7b44a80-1fbd-11eb-8214-9f82ecbf5b7a.JPG)

    - 주문, 대리점관리, 결제, 프로모션 어그리게잇을 생성하고 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌

### 바운디드 컨텍스트로 묶기

![바운디드컨텍스트로묶기](https://user-images.githubusercontent.com/73939028/98253097-f8e57780-1fbd-11eb-84a7-bfc7be960bcf.JPG)

    - 도메인 서열 분리 
        - Core Domain:  app(front), store : 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 app 의 경우 1주일 1회 미만, store 의 경우 1개월 1회 미만
        - Supporting Domain:  customer(view) : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        - General Domain:  pay : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

![폴리시부착](https://user-images.githubusercontent.com/73939028/98253109-fa16a480-1fbd-11eb-8367-469de3eb64df.JPG)

### 폴리시의 이동

![폴리시이동](https://user-images.githubusercontent.com/73939028/98253114-fb47d180-1fbd-11eb-9bf1-65e4c969c11a.JPG)

### 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![컨텍스트매핑](https://user-images.githubusercontent.com/73939028/98253119-fc78fe80-1fbd-11eb-89df-c6c1dd23134a.JPG)

    - 컨텍스트 매핑하여 묶어줌.

### 완성된 모형

![완성된 모형](https://user-images.githubusercontent.com/73939028/98253124-fd119500-1fbd-11eb-8dc9-4361a29cc035.JPG)

    - View Model 추가

### 기능적 요구사항 검증

![기능적요구사항1](https://user-images.githubusercontent.com/73939028/98255543-da34b000-1fc0-11eb-8daa-de43f529e533.JPG)

   	- 고객이 APP에서 폰을 주문한다. (ok)
   	- 고객이 결제한다. (ok)
	- 결제가 되면 결제 금액에 따라 페이백 포인트를 제공한다. (ok)

![기능적요구사항2](https://user-images.githubusercontent.com/73939028/98255548-db65dd00-1fc0-11eb-9d5b-16b62f0d016b.JPG)

	- 고객이 주문을 취소할 수 있다. (ok)
	- 주문이 취소되면 결제가 취소된다. (ok)
	- 결제가 취소되면 제공한 프로모션이 취소된다. (ok)

![image](https://user-images.githubusercontent.com/73699193/97982928-d3b70480-1e17-11eb-957e-6a9093d2a0d7.png)
  
	- 고객이 모든 진행내역을 볼 수 있어야 한다. (ok)


### 비기능 요구사항 검증

![비기능요구사항1](https://user-images.githubusercontent.com/73939028/98255536-d9038300-1fc0-11eb-8b84-0ee50c936a9f.JPG)

    - 1) 페이백 포인트 회수는 "반드시" 결제 취소가 되어야 한다. (Req/Res)
    - 2) 프로모션 시스템이 수행되지 않더라도 결제는 365일 24시간 처리되어야 한다. (Pub/sub)
    - 3) 프로모션 시스템이 과중되면 결제 취소가 잠시 후에 진행되도록 유도한다. (Circuit breaker)
    - 4) 결제가 완료되면 페이백 포인트가 적립되고 주문정보에 업데이트 되어야 한다.  (SAGA, 보상트렌젝션)
    - 5) 고객이 모든 진행내역과 프로모션 내역을 조회 할 수 있도록 성능을 고려하여 별도의 view로 구성한다. (CQRS, DML/SELECT 분리)


## 헥사고날 아키텍처 다이어그램 도출 (Polyglot)

![헥사고날아키텍처2](https://user-images.githubusercontent.com/73939028/98322473-d3428780-202a-11eb-9f7c-050389bbc43a.JPG)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐
    - 대리점과 프로모션의 경우 Polyglot 검증을 위해 Hsql로 셜계


# 구현:

서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd app
mvn spring-boot:run

cd pay
mvn spring-boot:run 

cd store
mvn spring-boot:run  

cd customer
mvn spring-boot:run  

cd promotion
mvn spring-boot:run  
```

## DDD 의 적용

각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 promotion 마이크로 서비스). 
이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 
하지만, 일부 구현에 있어서 영문이 아닌 경우는 실행이 불가능한 경우가 있기 때문에 계속 사용할 방법은 아닌것 같다. 
(Maven pom.xml, Kafka의 topic id, FeignClient 의 서비스 id 등은 한글로 식별자를 사용하는 경우 오류가 발생하는 것을 확인하였다)

![DDD1어그리게잇](https://user-images.githubusercontent.com/73939028/98259530-998b6580-1fc5-11eb-8c5b-471ea0db62af.JPG)

Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다

![DDD2리포지토리](https://user-images.githubusercontent.com/73939028/98259556-a14b0a00-1fc5-11eb-8a88-613d998a68ef.JPG)


## 폴리글랏 퍼시스턴스
프로모션의 경우 H2 DB인 주문과 결제와 달리 Hsql으로 구현하여 MSA간 서로 다른 종류의 DB간에도 문제 없이 동작하여 다형성을 만족하는지 확인하였다. 


app, pay, customer의 pom.xml 설정

![image](https://user-images.githubusercontent.com/73699193/97972993-baf32280-1e08-11eb-8158-912e4d28d7ea.png)


promotion, store의 pom.xml 설정

![image](https://user-images.githubusercontent.com/73699193/97973735-e0346080-1e09-11eb-9636-605e2e870fb0.png)



## CQRS 패턴

사용자 View를 위한 프로모션 정보 조회 서비스를 위한 별도의 프로모션 정보 저장소를 구현

- 이를 하여 기존 CQRS 서비스인 Customer 서비스를 활용
- 모든 정보는 비동기 방식으로 호출한다.
![DDD1어그리게잇](https://user-images.githubusercontent.com/73939028/98259530-998b6580-1fc5-11eb-8c5b-471ea0db62af.JPG)

- Customer에 저장하는 서비스 정책 (PolicyHandler)구현
![image](https://user-images.githubusercontent.com/73939028/98325832-0c7ef580-2033-11eb-8d12-66a7f47792ea.png)
![게이트웨이2](https://user-images.githubusercontent.com/73939028/98325919-4b14b000-2033-11eb-8712-17b6a4227871.JPG)



## Gateway 적용

gateway > applitcation.yml 설정

![Gateway_applicationyaml설정](https://user-images.githubusercontent.com/73939028/98259589-ad36cc00-1fc5-11eb-9658-c26aa7e7e6fd.JPG)

gateway 테스트

```
http POST http://gateway:8080/orders item=a1 qty=1 price=10
```
![게이트웨이](https://user-images.githubusercontent.com/73939028/98310720-0a0ba400-2011-11eb-948a-e037b960ccec.JPG)



## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 결제취소(pay)->프로모션취소(promotion) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 프로모션서비스를 호출하기 위하여 FeignClient 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 
```
# (pay) external > PromotionService.java

package phoneseller.external;

@FeignClient(name="promotion", url="${api.url.promotion}")
public interface PromotionService {

    @RequestMapping(method= RequestMethod.POST, path="/promotions")
    public void promoCancel(@RequestBody Promotion promotion);

}
```
![동기식호출과폴백](https://user-images.githubusercontent.com/73939028/98259640-b9228e00-1fc5-11eb-88c7-c135f4554f70.JPG)


- 주문을 받은 직후 결제를 요청하도록 처리
```
# (pay) Payment.java (Entity)

    @PreUpdate
    public void onPreUpdate(){

        phoneseller.external.Promotion promotion = new phoneseller.external.Promotion();
        promotion.setOrderId(getOrderId());
        promotion.setPoint((double)-1);
        promotion.setProcess("PayCancelled");
        // mappings goes here
        PayApplication.applicationContext.getBean(phoneseller.external.PromotionService.class)
                .promoCancel(promotion);
    }
```
![동기식호출+1](https://user-images.githubusercontent.com/73939028/98322951-05a0b480-202c-11eb-8c7e-30f7ec78abfe.JPG)

- 동기식 호출이 적용되서 결제 시스템이 장애가 나면 결제취소도 못받는다는 것을 확인:

```
#프로모션(promotion) 서비스를 잠시 내려놓음

#주문취소하기(order)
http PATCH http://localhost:8081/orders/5 status="cancel"   #Fail
```
![동기식호출과폴백_실행_서비스킬_결과](https://user-images.githubusercontent.com/73939028/98311214-32e06900-2012-11eb-8f73-7c5d963974e9.JPG)

```
#프로모션(promotion) 서비스 재기동
cd pay
mvn spring-boot:run

#주문취소하기(order)
http PATCH http://localhost:8081/orders/5 status="cancel"   #Success
```
![동기식호출과폴백_실행_서비스온_결과](https://user-images.githubusercontent.com/73939028/98311224-3b38a400-2012-11eb-8beb-63888f064e60.JPG)



## 비동기식 호출 / 시간적 디커플링 / 장애격리 


결제(pay)가 이루어진 후에 프로모션(promotion)으로 이를 알려주는 행위는 비 동기식으로 처리하여 프로모션(promotion)의 처리를 위하여 결제가 블로킹 되지 않아도록 처리한다.
 
- 결제승인이 되었다(payCompleted)는 도메인 이벤트를 카프카로 송출한다(Publish)
 
![비동기식+1](https://user-images.githubusercontent.com/73939028/98323343-efdfbf00-202c-11eb-9c23-6c63f14730cf.JPG)


- 프로모션(promotion)에서는 결제승인(PayCompleted) 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다.
- 포인트적립(Promote)는 송출된 결제승인(PayCompleted) 정보를 PromotionRepository에 저장한다.
 
![비동기식호출2](https://user-images.githubusercontent.com/73939028/98259725-d0617b80-1fc5-11eb-8c19-c7c63a1af939.JPG)


프로모션(promotion)시스템은 주문(app)/결제(pay)와 완전히 분리되어있으며(sync transaction 없음), 이벤트 수신에 따라 처리되기 때문에, 프로모션(promotion)이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다.(시간적 디커플링)

```
#프로모션(promotion) 서비스를 잠시 내려놓음

#주문하기(order)
http POST http://localhost:8081/orders item="iphone12pro" qty=1 price=1200  #Success

#주문상태 확인
http GET http://localhost:8081/orders/3    # 포인트 정보가 업데이트되지 않고 NULL임을 확인
```
![비동기식호출_실행_서비스킬](https://user-images.githubusercontent.com/73939028/98311419-b8fcaf80-2012-11eb-8874-75e9f6fde50d.JPG)

```
#프로모션(promotion) 서비스 기동
cd promotion
mvn spring-boot:run

#주문상태 확인
http get http://localhost:8081/orders/3     # NULL이었던 포인트가 가격에 따라 적립된 것을 확인
```
![비동기식호출_실행_서비스온](https://user-images.githubusercontent.com/73939028/98311424-bac67300-2012-11eb-9bb2-52f4aa42deef.JPG)



# 운영

## Deploy / Pipeline

- 네임스페이스 만들기
```
kubectl create ns pp82
kubectl get ns
```
![네임스페이스만들기](https://user-images.githubusercontent.com/73939028/98314252-ed736a00-2018-11eb-994c-9d47bb501b4e.JPG)


- 폴더 만들기, 해당폴더로 이동
```
mkdir pp82
cd pp82
```
![폴더만들기이동](https://user-images.githubusercontent.com/73939028/98314296-fd8b4980-2018-11eb-99ad-bdb082eb9c18.JPG)


- 소스 가져오기
```
git clone https://github.com/eunjeeheo/promotion.git
```
![소스가져오기](https://user-images.githubusercontent.com/73939028/98314294-fc5a1c80-2018-11eb-8420-0a07965e60af.JPG)


- 빌드하기
```
cd promotion
mvn package -Dmaven.test.skip=true
```
![빌드하기](https://user-images.githubusercontent.com/73939028/98314379-2f9cab80-2019-11eb-9992-8acbbfae91aa.JPG)


- 도커라이징: Azure 레지스트리에 도커 이미지 푸시하기
```
az acr build --registry admin37 --image admin37.azurecr.io/app:latest .
```
![도커라이징](https://user-images.githubusercontent.com/73939028/98314255-ee0c0080-2018-11eb-9f03-174c0bf6fd6a.JPG)


- 컨테이너라이징: 디플로이 생성 확인
```
kubectl create deploy promotion --image=admin37.azurecr.io/app:latest -n pp82
kubectl get all -n pp82
```
![컨테이너라이징](https://user-images.githubusercontent.com/73939028/98314419-49d68980-2019-11eb-867a-7b4482afa15f.JPG)


- 컨테이너라이징: 서비스 생성 확인
```
kubectl expose deploy promotion --type="ClusterIP" --port=8080 -n pp82
kubectl get all -n pp82
```
![컨테이너라이징_서비스생성](https://user-images.githubusercontent.com/73939028/98314424-4a6f2000-2019-11eb-8c36-c214f46bdc30.JPG)



-(별첨)deployment.yml을 사용하여 배포 

- deployment.yml 편집
```
namespace, image 설정
env 설정 (config Map) 
readiness 설정 (무정지 배포)
liveness 설정 (self-healing)
resource 설정 (autoscaling)
```
![디플로이먼트배포](https://user-images.githubusercontent.com/73939028/98314707-ebf67180-2019-11eb-9b49-12c9b3ab6718.JPG)

- deployment.yml로 서비스 배포
```
cd promotion
kubectl apply -f kubernetes/deployment.yml
```

## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 결제(pay)-->프로모션(promotion) 취소의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 프로모션 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml
feign:
  hystrix:
    enabled: true
    
hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```
![image](https://user-images.githubusercontent.com/73699193/98093705-a166df00-1ecb-11eb-83b5-f42e554f7ffd.png)

* siege 툴 사용법:
```
 siege가 생성되어 있지 않으면:
 kubectl run siege --image=apexacme/siege-nginx -n pp82
 siege 들어가기:
 kubectl exec -it pod/siege-5c7c46b788-x5qp5 -c siege -n pp82 -- /bin/bash
 siege 종료:
 Ctrl + C -> exit
```
* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 10명
- 10초 동안 실시

```
siege -c10 -t10S -v --content-type "application/json" 'http://pay:8080/payments POST {"orderId": "380", "process":"OrderedCancelled"}'
```
- 부하 발생하여 CB가 발동하여 요청 실패처리하였고, promotion에서 밀린 부하가 처리되면서 다시 결제취소를 받기 시작 

![서킷브레이킹1](https://user-images.githubusercontent.com/73939028/98321737-2adff380-2029-11eb-8ecb-64fe7086aa86.JPG)

- report

![서킷브레이킹2](https://user-images.githubusercontent.com/73939028/98321733-29aec680-2029-11eb-997e-80a965f1310b.JPG)

- CB 잘 적용됨을 확인


### 오토스케일 아웃

- 프로모션 시스템에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 20프로를 넘어서면 replica 를 10개까지 늘려준다:

```
# autocale out 설정
promotion > deployment.yml 설정
```
![오토스케일아웃1](https://user-images.githubusercontent.com/73939028/98312924-0a5a6e00-2016-11eb-8367-cac215cef0e1.JPG)

```
kubectl autoscale deploy promotion --min=1 --max=10 --cpu-percent=20 -n pp82
```
![오토스케일아웃2](https://user-images.githubusercontent.com/73939028/98312932-0d555e80-2016-11eb-8e63-737b954011fb.JPG)


-
- 부하테스터 siege 툴을 통한 워크로드를 1분 동안 걸어준다.
```
kubectl exec -it pod/siege-5c7c46b788-x5qp5 -c siege -n pp82 -- /bin/bash
siege -c10 -t60S -r10 -v --content-type "application/json" 'http://gateway:8080/promotions'
```

- 부하를 주고 확인하니 Availability가 100%로 유지되는 것을 확인 할 수 있었다.

![오토스케일아웃3](https://user-images.githubusercontent.com/73939028/98312940-121a1280-2016-11eb-8836-f68319479db4.JPG)

- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy promotion -w -n pp82
```
- 어느정도 시간이 흐른 후 스케일 아웃이 벌어지는 것을 확인할 수 있다. max=10 
- 부하를 줄이니 늘어난 스케일이 점점 줄어들었다.

![오토스케일아웃5](https://user-images.githubusercontent.com/73939028/98324382-6ed5f700-202f-11eb-91ac-0e4425ade9a8.JPG)




## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscale 이나 CB 설정을 제거함


- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c10 -t60S -r10 -v --content-type "application/json" 'http://promotion:8080/promotions GET'
```

- readiness 옵션이 없는 경우 배포 중 서비스 요청처리 실패
```
kubectl apply -f kubernetes/deployment_readiness.yml
```
![무정지재배포1](https://user-images.githubusercontent.com/73939028/98313516-5f4ab400-2017-11eb-9be9-6f8481ab680e.JPG)


- deployment.yml에 readiness 옵션을 추가 
```
kubectl apply -f kubernetes/deployment.yml
```
![무정지재배포2](https://user-images.githubusercontent.com/73939028/98313519-61147780-2017-11eb-850d-92bdcf772d68.JPG)

- readiness적용된 deployment.yml 적용

```
kubectl apply -f kubernetes/deployment.yml
```
- 새로운 버전의 이미지로 교체
```
az acr build --registry admin37 --image admin37.azurecr.io/promotion:v4 .
kubectl set image deploy promotion promotion=admin37.azurecr.io/promotion:v4 -n pp82
```
- 기존 버전과 새 버전의 store pod 공존 중

![무정지재배포3](https://user-images.githubusercontent.com/73939028/98313520-61ad0e00-2017-11eb-8bc8-1f02b71bc58a.JPG)

- Availability: 100.00 % 확인

![무정지재배포4](https://user-images.githubusercontent.com/73939028/98324570-f58ad400-202f-11eb-863f-e33130d57b9f.JPG)



## Config Map

- apllication.yml 설정

* default쪽

![컨피그맵_디폴트](https://user-images.githubusercontent.com/73939028/98259854-f555ee80-1fc5-11eb-8ca4-029259890e41.JPG)

* docker 쪽

![컨피그맵_도커](https://user-images.githubusercontent.com/73939028/98259857-f5ee8500-1fc5-11eb-9210-400545080e90.JPG)

- Deployment.yml 설정

![컨피그맵_디플로이먼트](https://user-images.githubusercontent.com/73939028/98259859-f6871b80-1fc5-11eb-9dee-67058504aaca.JPG)

- config map 생성 후 조회
```
kubectl create configmap apiurl --from-literal=url=http://promotion:8080 --from-literal=fluentd-server-ip=10.xxx.xxx.xxx -n pp82
```
![컨피그맵_생성조회](https://user-images.githubusercontent.com/73939028/98313627-9a4ce780-2017-11eb-98cc-464d75d077ae.JPG)

- 설정한 url로 프로모션 호출
```
http http://gateway:8080/promotions
```

![컨피그맵_url호출](https://user-images.githubusercontent.com/73939028/98313682-b6508900-2017-11eb-9fc5-0a3e365db2db.JPG)

- configmap 삭제 후 app 서비스 재시작
```
kubectl delete configmap apiurl -n pp82
kubectl get pod/pay-875b546bc-77rd2  -n pp82 -o yaml | kubectl replace --force -f-
```
![컨피그맵_삭제](https://user-images.githubusercontent.com/73939028/98313671-b0f33e80-2017-11eb-8b5b-4ee59195ca9a.JPG)

- configmap 삭제된 상태에서 주문 호출   
```
http PATCH http://gateway:8080/payments process="OrderCancelled"
```

![컨피그맵_오류](https://user-images.githubusercontent.com/73939028/98313631-9caf4180-2017-11eb-9dfe-71629e275939.JPG)



## Self-healing (Liveness Probe)

- promotion 서비스 정상 확인

![셀프힐링1](https://user-images.githubusercontent.com/73939028/98313982-6c1bd780-2018-11eb-9a3a-b30aa843d3d2.JPG)


- deployment.yml 에 Liveness Probe 옵션 추가
```
cd ~/pp92/promotion/kubernetes
vi deployment.yml

(아래 설정 변경)
livenessProbe:
	tcpSocket:
	  port: 8081
	initialDelaySeconds: 5
	periodSeconds: 5
```
![셀프힐링2](https://user-images.githubusercontent.com/73939028/98314144-ba30db00-2018-11eb-89da-bf8fa9509571.JPG)

- promotion pod에 liveness가 적용된 부분 확인

![셀프힐링3](https://user-images.githubusercontent.com/73939028/98313987-6d4d0480-2018-11eb-91fb-8c83bdefab7f.JPG)

- promotion 서비스의 liveness가 발동되어 4번 retry 시도 한 부분 확인

![셀프힐링4](https://user-images.githubusercontent.com/73939028/98313988-6d4d0480-2018-11eb-9612-8be2563da67d.JPG)

