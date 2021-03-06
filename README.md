# 영화 예매 경품 서비스

MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 3조 프로젝트의 개인 과제입니다.
조별 과제에 티켓 발권 시 경품을 자동으로 부여하고, 수령할 수 있는 시스템이 추가되었습니다. 

- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW



# 서비스 시나리오

기능적 요구사항

1. 고객이 영화 및 좌석을 선택하고 예매를 한다. 
1. 고객이 결제를 진행한다.
1. 예매 및 결제가 완료되면 티켓이 생성된다.
1. 영화관에서 나의 예매 정보로 티켓을 수령한다.
1. 티켓을 수령하면 경품이 랜덤하게 당첨된다.
1. 당첨된 경품을 수령한다.
1. 티켓 수령 전까지 고객이 예매를 취소할 수 있다. 
1. 예매가 취소되면 결제가 취소된다.
1. 고객이 예매 내역 및 상태를 조회할 수 있다.

비기능적 요구사항

1. 트랜잭션
   1. 결제가 되지 않은 예매 건은 아예 예매가 성립되지 않아야 한다. Sync 호출
1. 장애격리
   1. 티켓 수령 기능이 수행되지 않더라도 예매는 365일 24시간 받을 수 있어야 한다. Async (event-driven), Eventual Consistency
   1. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다. Circuit breaker, fallback
1. 성능
   1. 고객이 예매 내역을 my page(프론트엔드)에서 확인할 수 있어야 한다 CQRS
   1. 예매 상태가 바뀔때마다 mypage에서 확인 가능하여야 한다 Event driven

# 분석/설계

## Event Storming 결과

- MSAEz 로 모델링한 이벤트스토밍 결과: 
  <img width="833" alt="스크린샷 2021-02-19 오후 4 18 04" src="https://user-images.githubusercontent.com/25216200/109028943-ad3e1180-7705-11eb-8bc8-bccd591e4dc5.png">

## 헥사고날 아키텍처 다이어그램 도출

![헥사고널](https://user-images.githubusercontent.com/25216200/109097475-bdd0a500-7762-11eb-80ba-cde18cfda3a6.png)

# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 
(각자의 포트넘버는 gateway 외에 8081 ~ 8085 이다)

```
cd book
mvn spring-boot:run

cd payment
mvn spring-boot:run

cd ticket
mvn spring-boot:run

cd mypage
mvn srping-boot:run

cd gateway
mvn spring-boot:run

cd gift
mvn spring-boot:run
```

## 동기식 호출

분석단계에서의 조건 중 하나로 예매(book)->결제(pay) 간의 호출과 추가적으로 구현 된 티켓 발권(ticket)->경품당첨(gift) 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다.
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.

- 아래는 경품 서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현

```
# GiftService.java

package movie.external;

@FeignClient(name="gift", url="http://localhost:8085")
public interface GiftService {

    @RequestMapping(method= RequestMethod.POST, path="/gifts")
    public void apply(@RequestBody Gift gift);
}
```

- 티켓 발권 직후(@PostPersist) 경품이 당첨되어 부여되도록 요청 처리

```
# Ticket.java (Entity)

    @PostUpdate
    public void onPostUpdate(){

        if("Printed".equals(status)){
             Printed printed = new Printed();
             BeanUtils.copyProperties(this, printed);
             printed.setStatus("Printed");
             printed.publishAfterCommit();
            
            movie.external.Gift gift = new movie.external.Gift();
            
            // mappings goes here

            gift.setBookingId(printed.getBookingId());
            Random random = new Random();
            Integer randomValue = random.nextInt(3);
            switch (randomValue) {  
                case 0: 
                    
                    gift.setName("Americano");
                    gift.setGiftCode("G000");
                    break;
                case 1:     
                    gift.setName("CafeLatte");
                    gift.setGiftCode("G001");
                    break; 
                case 2:
                    gift.setName("CafeMocha");
                    gift.setGiftCode("G002");
                    break;
                case 3:
                    gift.setName("Cappuccino");
                    gift.setGiftCode("G003");
                    break;    
                default:
                    gift.setName("Americano");
                    gift.setGiftCode("G000");
            };
            gift.setStatus("GiftApplied");
            TicketApplication.applicationContext.getBean(movie.external.GiftService.class)
            .apply(gift);
            
        }
    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 예매도 못받는다는 것을 확인


- 경품 (gift) 서비스를 잠시 내려놓음 (ctrl+c)

1. 티켓 출력과 동시에 경품처리

![gift_Fail](https://user-images.githubusercontent.com/25216200/109085810-bc48b200-774d-11eb-96d1-657b06aaff94.png)


2. 경품 서비스 재기동
```
cd ../gift
mvn spring-boot:run
```

3. 다시 티켓 출력 처리

![gift_restart](https://user-images.githubusercontent.com/25216200/109085877-d8e4ea00-774d-11eb-8daa-c311d4a59ca4.png)


## 비동기식 호출

경품을 수령하면 Book 시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리한다.

- 이를 위하여 경품 수령 후에 곧바로 경품을 수령했다는 도메인 이벤트를 카프카로 송출한다(Publish)

```
package movie;

@Entity
@Table(name="Gift_table")
public class Gift {

 ...
    @PostUpdate
    public void onPostUpdate(){
        if("Taken".equals(status)){
            Taken taken = new Taken();
            BeanUtils.copyProperties(this, taken);
            taken.setStatus("PritedAndTakenGift");
            taken.publishAfterCommit();
        }
    }
	
	'''

}
```

- Book 서비스에서는 Taken 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
package movie;

...

@Service
public class PolicyHandler{

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverTaken_(@Payload Taken taken){
        if(taken.isMe()){

            System.out.println("======================================");
            System.out.println("**** listener  : " + taken.toJson());
            System.out.println("======================================");
            bookRepository.findById(taken.getBookingId()).ifPresent((book)->{
                book.setStatus("PritedAndTakenGift");
                bookRepository.save(book);
            });

        }
    };

}

```
- 또한, Ticket 시스템은 예매/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, Ticket 시스템이 유지보수로 인해 잠시 내려간 상태라도 예매 받는데 문제가 없다:

- Ticket 서비스를 잠시 내려놓음 (ctrl+c)

1. 예매처리
![book0](https://user-images.githubusercontent.com/25216200/109092251-890c2000-7759-11eb-9fc9-6c255fb4da3a.png)
```
{"eventType":"Paid","timestamp":"20210225105652","id":null,"bookingId":2,"status":"Paid","me":true}
{"eventType":"Booked","timestamp":"20210225105651","id":2,"qty":2,"seat":"1D,2D","movieName":"“batman”","status":"Registered","totalPrice":10000,"name":null,"me":true}
```



2. 예매상태 확인

![book1](https://user-images.githubusercontent.com/25216200/109092015-2c106a00-7759-11eb-92cc-d8262a1e63b4.png)

3. Ticket 서비스 기동
```
cd ../ticket
mvn spring-boot:run
```

4. 예매상태 확인

![ticket](https://user-images.githubusercontent.com/25216200/109092076-46e2de80-7759-11eb-8b7a-5f5125802947.png)


## Gateway

- Gateway의 application.yaml에 모든 서비스들이 8088 포트를 사용할 수 있도록 한다.


```
# gateway.application.yaml
spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: book
          uri: http://localhost:8081
          predicates:
            - Path=/books/** 
        - id: payment
          uri: http://localhost:8082
          predicates:
            - Path=/payments/** 
        - id: mypage
          uri: http://localhost:8083
          predicates:
            - Path= /mypages/**
        - id: ticket
          uri: http://localhost:8084
          predicates:
            - Path=/tickets/**
        - id: gift
          uri: http://localhost:8085
          predicates:
            - Path=/gifts/**  
          
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

- 8088 포트를 사용하여 API를 발생시킨다.

```
# book 서비스의 예매처리
http POST http://localhost:8088/books qty=2 movieName="Avengers" seat="3A,3B" totalPrice=10000

# ticket 서비스의 출력 및 경품 당첨 처리
http PATCH http://localhost:8088/tickets/2 status="Printed"

# 예매 상태 확인
http http://localhost:8088/books/3

# 경품 상태 확인
http http://localhost:8088/gifts/2

```
![avg](https://user-images.githubusercontent.com/25216200/109093624-020c7700-775c-11eb-86ea-4dc099e09c8d.png)

![avg_print](https://user-images.githubusercontent.com/25216200/109093861-692a2b80-775c-11eb-8d49-85a1224d9c4c.png)

![avg_gift](https://user-images.githubusercontent.com/25216200/109093960-9971ca00-775c-11eb-9cdd-57f747f5dcb9.png)

![avg_gift2](https://user-images.githubusercontent.com/25216200/109094591-93c8b400-775d-11eb-9586-ee7273ce5158.png)
## Mypage

- 고객은 예매 상태를 Mypage에서 확인할 수 있다.

- REST API 의 테스트

```
# book 서비스의 예매처리
http POST http://localhost:8088/books qty=2 movieName="Spiderman" seat="10A,10B" totalPrice=10000

# ticket 서비스의 출력처리 및 경품 부여
http PATCH http://localhost:8088/tickets/1 status="Printed"

# gift 서비스의 경품 수령
http PATCH http://localhost:8088/gifts/1 status="Taken"

# Mypage에서 상태 확인
http http://localhost:8088/mypages/1

```

![mypage1](https://user-images.githubusercontent.com/25216200/109096523-04250480-7761-11eb-8959-b74cd6b906d7.png)
![mypage2](https://user-images.githubusercontent.com/25216200/109096554-1a32c500-7761-11eb-8312-14e106e96263.png)

## Polyglot

```
# Book - pom.xml

		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>


# Ticket - pom.xml

		<dependency>
			<groupId>org.hsqldb</groupId>
			<artifactId>hsqldb</artifactId>
			<scope>runtime</scope>
		</dependency>

# Gift - pom.xml

		<dependency>
			<groupId>org.hsqldb</groupId>
			<artifactId>hsqldb</artifactId>
			<scope>runtime</scope>
		</dependency>

```



# 운영

## Pipeline

각 구현체들은 Amazon ECR(Elastic Container Registry)에 구성되었고, 사용한 CI/CD 플랫폼은 AWS Codebuild며, pipeline build script 는 각 프로젝트 폴더 이하에 buildspec.yml 에 포함되었다. 

```
# gift/buildspec.yaml
version: 0.2

env:
  variables:
    _PROJECT_NAME: "gift"
    _PROJECT_DIR: "gift"

phases:
  install:
    runtime-versions:
      java: openjdk8
      docker: 18
    commands:
      - echo install kubectl
      # - curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
      # - chmod +x ./kubectl
      # - mv ./kubectl /usr/local/bin/kubectl
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - echo $_PROJECT_NAME
      - echo $AWS_ACCOUNT_ID
      - echo $AWS_DEFAULT_REGION
      - echo $CODEBUILD_RESOLVED_SOURCE_VERSION
      - echo start command
      - $(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - cd $_PROJECT_DIR
      - mvn package -Dmaven.test.skip=true
      - docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/skccuser14-$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION  .
  post_build:
    commands:
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/skccuser14-$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - echo connect kubectl
      - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
      - kubectl config set-credentials admin --token="$KUBE_TOKEN"
      - kubectl config set-context default --cluster=k8s --user=admin
      - kubectl config use-context default
      - |
        cat <<EOF | kubectl apply -f -
        apiVersion: v1
        kind: Service
        metadata:
          name: $_PROJECT_NAME
          namespace: movie
          labels:
            app: $_PROJECT_NAME
        spec:
          ports:
            - port: 8080
              targetPort: 8080
          selector:
            app: $_PROJECT_NAME
        EOF
      - |
        cat  <<EOF | kubectl apply -f -
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: $_PROJECT_NAME
          namespace: movie
          labels:
            app: $_PROJECT_NAME
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: $_PROJECT_NAME
          template:
            metadata:
              labels:
                app: $_PROJECT_NAME
            spec:
              containers:
                - name: $_PROJECT_NAME
                  image: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/skccuser14-$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
                  ports:
                    - containerPort: 8080
        EOF
cache:
  paths:
    - "/root/.m2/**/*"
```

- 서비스 이미지
![repository](https://user-images.githubusercontent.com/25216200/109039291-f4c99b00-770f-11eb-834e-a4fed13e1434.png)

- Pipeline


![Codebild](https://user-images.githubusercontent.com/25216200/109038903-91d80400-770f-11eb-8cae-237a082b0b99.png)
## Zero-downtime deploy(Readiness Probe)

- buildspec.yaml 파일에 Readiness Probe 추가

```
readinessProbe:
  httpGet:
    path: /abc
    port: 8080
  initialDelaySeconds: 10
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 10

```
![Readiness](https://user-images.githubusercontent.com/25216200/109042980-28a6bf80-7714-11eb-977f-9464995946ec.png)



## Self-healing(Liveness Probe)

- buildspec.yaml 파일에 Liveness Probe 추가

```
  livenessProbe:
    httpGet:
      path: /abc
      port: 8080
    initialDelaySeconds: 120
    timeoutSeconds: 2
    periodSeconds: 5
    failureThreshold: 5

```
![livenessProbe](https://user-images.githubusercontent.com/25216200/109046697-6279c500-7718-11eb-891c-e2fe4d5ec2c7.png)

## Config Map

- buildspec.yaml에 env 추가 (Gift 서비스)


```
# deployment.yaml

  env:
    - name: NAME
      valueFrom:
	configMapKeyRef:
	  name: moviecm
	  key: text1

```

- 경품 부여와 함께 환경변수로 설정한 NAME(msg 변수에)이 들어가도록 코드를 변경

```
@Id
@GeneratedValue(strategy=GenerationType.AUTO)

...

private String msg = System.getenv("NAME");

```
- moviecm.yaml 작성

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: moviecm
  namespace: movie
data:
  text1: Congratulation Gift

```

- moviecm.yaml 적용

```
kubectl create -f moviecm.yaml

```

- gift pod에 들어가서 환경변수 확인

![configmap](https://user-images.githubusercontent.com/25216200/109103067-7ef41c80-776d-11eb-8e3c-f72fd80f92ee.png)


- 경품 부여 동시에 name(msg에 환경변수 적용 

![configmap2](https://user-images.githubusercontent.com/25216200/109104611-a21fcb80-776f-11eb-87ce-80df32e1e4d5.png)




## Circuit Breaker

- 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 영화 티켓 발권 (ticket) --> 경품( gift ) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정: 요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정

```
# application.yml in ticket service

feign:
  hystrix:
    enabled: true
    
hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

- 피호출 서비스(결제: gift) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게

```
# (gift) Gift.java (Entity)

    @PrePersist
    public void onPrePersist(){
    
        try {
            Thread.currentThread().sleep((long) (400 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
	...
    }
```

- 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:

* 동시사용자 50명
* 60초 동안 실시



```
$ siege -c50 -t60S -r10 -v  --content-type "application/json" 'http://ticket:8080/tickets POST {"status":"Printed"}'



** SIEGE 4.0.5
** Preparing 50 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     1.17 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     1.26 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     1.25 secs:     282 bytes ==> POST http://book:8080/books

* 요청이 과도하여 CB를 동작함 요청을 차단

HTTP/1.1 201     1.51 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.38 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 500     2.65 secs:     247 bytes ==> POST http://book:8080/books
HTTP/1.1 500     1.98 secs:     247 bytes ==> POST http://book:8080/books
HTTP/1.1 500     1.87 secs:     247 bytes ==> POST http://book:8080/books
HTTP/1.1 500     1.86 secs:     247 bytes ==> POST http://book:8080/books
HTTP/1.1 500     1.77 secs:     247 bytes ==> POST http://book:8080/books
HTTP/1.1 500     1.78 secs:     247 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.37 secs:     282 bytes ==> POST http://book:8080/books

* 요청을 어느정도 돌려보내고나니, 기존에 밀린 일들이 처리되었고, 회로를 닫아 요청을 다시 받기 시작

HTTP/1.1 201     2.43 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.49 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.57 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.53 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.45 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.55 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     0.51 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.38 secs:     282 bytes ==> POST http://book:8080/books

* 다시 요청이 쌓이기 시작하여 건당 처리시간이 610 밀리를 살짝 넘기기 시작 => 회로 열기 => 요청 실패처리

HTTP/1.1 500     2.65 secs:     247 bytes ==> POST http://book:8080/books
HTTP/1.1 500     1.98 secs:     247 bytes ==> POST http://book:8080/books
HTTP/1.1 500     1.86 secs:     247 bytes ==> POST http://book:8080/books
HTTP/1.1 500     1.78 secs:     247 bytes ==> POST http://book:8080/books

* 생각보다 빨리 상태 호전됨 - (건당 (쓰레드당) 처리시간이 610 밀리 미만으로 회복) => 요청 수락

HTTP/1.1 201     2.37 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.51 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.40 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.64 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.64 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.64 secs:     282 bytes ==> POST http://book:8080/books

* 이후 이러한 패턴이 계속 반복되면서 시스템은 도미노 현상이나 자원 소모의 폭주 없이 잘 운영됨

HTTP/1.1 201     2.38 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.34 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.13 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 500     2.54 secs:     247 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.33 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 500     2.54 secs:     247 bytes ==> POST http://book:8080/books
HTTP/1.1 500     1.74 secs:     247 bytes ==> POST http://book:8080/books
HTTP/1.1 500     1.73 secs:     247 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.24 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.31 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.36 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 500     2.39 secs:     247 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.38 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 500     1.80 secs:     247 bytes ==> POST http://book:8080/books
HTTP/1.1 201     2.38 secs:     282 bytes ==> POST http://book:8080/books
HTTP/1.1 201     4.49 secs:     282 bytes ==> POST http://book:8080/books


:
:

Transactions:                   1030 hits
Availability:                  62.05 %
Elapsed time:                  59.83 secs
Data transferred:               0.43 MB
Response time:                  2.85 secs
Transaction rate:              17.22 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                   48.99
Successful transactions:        1030
Failed transactions:             630
Longest transaction:            5.20
Shortest transaction:           0.01

```

![cb](https://user-images.githubusercontent.com/25216200/109111539-73f4b880-777c-11eb-9982-41c40f2f283c.png)



- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 62% 가 성공하였고, 38%가 실패했다는 것은 고객 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.

- Availability 가 높아진 것을 확인 (siege)

## Autoscale (HPA)

앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다.

- 결제서비스에 대한 deplyment.yml 파일에 해당 내용을 추가한다.

```
  resources:
    requests:
      cpu: "300m"
    limits:
      cpu: "500m"
```

- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:

```
kubectl autoscale deploy payment --min=1 --max=10 --cpu-percent=15
```

- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.

```
siege -c50 -t120S -r10 --content-type "application/json" 'http://book:8080/books POST {"qty": "3"}'
```

- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:

```
kubectl get deploy payment -w
```

- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:

```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
payment   1/1     1            1           81s
payment   1/4     1            1           3m51s
payment   1/8     4            1           4m6s
payment   1/8     8            1           4m6s
payment   1/9     8            1           4m21s
payment   2/9     9            2           5m13s
payment   3/9     9            3           5m18s
payment   4/9     9            4           5m20s
payment   5/9     9            5           5m28s
payment   6/9     9            6           5m29s
payment   7/9     9            7           5m29s
payment   8/9     9            8           5m31s
payment   9/9     9            9           5m42s
```

- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다.

```
Transactions:                    976 hits
Availability:                  89.95 %
Elapsed time:                 119.45 secs
Data transferred:               0.29 MB
Response time:                  0.61 secs
Transaction rate:               8.17 trans/sec
Throughput:                     0.00 MB/sec
Concurrency:                    4.95
Successful transactions:         976
Failed transactions:             109
Longest transaction:            0.79
Shortest transaction:           0.41
```

