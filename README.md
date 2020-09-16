# Intensive Lv2. 이상평

음식을 주문하고 요리하여 배달하는 현황을 확인 할 수 있는 CNA의 개발

# Table of contents

- [Restaurant](# )
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석설계)
  - [구현:](#구현)
    - [DDD 의 적용](#ddd-의-적용)
    - [동기식 호출 과 Fallback 처리](#동기식-호출과-Fallback-처리)
    - [비동기식 호출과 Saga Pattern](#비동기식-호출과-Saga-Pattern)
    - [Gateway](#Gateway)
    - [CQRS](#CQRS)
  - [운영](#운영)
    - [AWS를 활용한 코드 자동빌드 배포 환경구축](#AWS를-활용한-코드-자동빌드-배포-환경구축)
    - [서킷 브레이킹과 오토스케일](서킷-브레이킹과-오토스케일)
    - [무정지 배포](#무정지-배포)
    - [마이크로서비스 로깅 관리를 위한 PVC 설정](#마이크로서비스-로깅-관리를-위한-PVC-설정)
    - [SelfHealing](#SelfHealing)
  - [첨부](#첨부)

# 서비스 시나리오

음식을 주문하고, 요리현황 및 배달현황, 쿠폰발행 현황 을 조회

## 기능적 요구사항

1. 고객이 주문을 하면 주문정보를 바탕으로 요리가 시작된다.
1. 고객이 주문을 하면 신규쿠폰이 발행된다.
1. 요리가 완료되면 배달이 시작된다. 
1. 고객이 주문취소를 하게 되면 요리가 취소된다.
1. 고객이 주문취소를 하게 되면 쿠폰발행도 취소된다.
1. 고객 주문에 재고가 없을 경우 주문이 취소된다. 
1. 고객은 Mypage를 통해, 주문과 요리, 배달 그리고 쿠폰발행의 전체 상황을 조회할수 있다.

## 비기능적 요구사항
1. 장애격리
    1. 주문시스템이 과중되면 사용자를 잠시동안 받지 않고 잠시후에 주문하도록 유도한다.
    1. 주문 시스템이 죽을 경우 재기동 될 수 있도록 한다.
1. 운영
    1. 마이크로서비스별로 로그를 한 곳에 모아 볼 수 있도록 시스템을 구성한다.
    1. 마이크로서비스의 개발 및 빌드 배포가 한번에 이루어질 수 있도록 시스템을 구성한다.
    1. 서비스라우팅을 통해 한개의 접속점으로 서비스를 이용할 수 있도록 한다.
    1. 주문 시스템이 과중되면 Replica를 추가로 띄울 수 있도록 한다.

# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과 : http://labs.msaez.io/#/storming/ecU7zzeUBxdYwUDu64Yfh34lznp2/mine/ac871e565e4b8d2e4dbe3a770cc8bbea/-MHAwa2m7jURuMoDzgD4
![eventStorming](https://user-images.githubusercontent.com/69958878/93348727-8e437580-f871-11ea-9658-d9cddd14f91c.png)

### 이벤트 도출
1. 주문됨
1. 주문취소됨
1. 요리재고체크됨
1. 요리완료
1. 배달
1. 쿠폰발행됨
1. 쿠폰발행취소됨


### 어그리게잇으로 묶기

  * 고객의 주문(Order), 식당의 요리(Cook), 배달(Delivery), 쿠폰(Coupon) 은 그와 연결된 command와 event 들에 의하여 트랙잭션이 유지되어야 하는 단위로 묶어 줌.

### Policy 부착 

### Policy와 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Res)

### 기능적 요구사항 검증
 * 고객이 메뉴를 주문한다.(ok)
 * 주문된 주문정보를 레스토랑으로 전달한다.(ok)
 * 주문정보를 바탕으로 요리가 시작된다.(ok)
 * 요리가 완료되면 배달이 시작된다.(ok)
 * 고객은 본인의 주문을 취소할 수 있다.(ok)
 * 주문이 취소되면 요리를 취소한다.(ok)
 * 주문이 취소되면, 요리취소 내용을 고객에게 전달한다.(ok)
 * 고객이 주문 시 재고량을 체크한다.(ok)
 * 재고가 없을 경우 주문이 취소된다.(ok)
 * 고객이 주문 시 신규 쿠폰이 발행된다.
 * 고객이 주문 취소시 신규 쿠폰이 취소된다.
 * 고객은 Mypage를 통해, 주문과 요리, 배달의 전체 상황을 조회할수 있다.(ok)

</br>
</br>



# 구현:

분석/설계 단계에서 도출된 아키텍처에 따라, 각 BC별로 마이크로서비스들을 스프링부트 + JAVA로 구현하였다. 각 마이크로서비스들은 Kafka와 RestApi로 연동되며 스프링부트의 내부 H2 DB를 사용한다.


## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 주문- Order 마이크로서비스).

```
package myProject_LSP;
import org.springframework.beans.BeanUtils;
import javax.persistence.*;

@Entity
@Table(name="Coupon_table")
public class Coupon {


    private static int couponQty=10;
    private boolean couponFlowChk=true;
    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long orderId;
    private String status;
    private Long sendDate;
    private String couponKind;
    
    ....
}
```
- JPA를 활용한 Repository Pattern을 적용하여 이후 데이터소스 유형이 변경되어도 별도의 처리 없이 사용 가능한 Spring Data REST 의 RestRepository 를 적용하였다
```
package myProject_LSP;
import org.springframework.data.repository.PagingAndSortingRepository;
import java.util.Optional;

public interface CouponRepository extends PagingAndSortingRepository<Coupon, Long>{
    Optional<Coupon> findByOrderId(Long orderId);
}
```
</br>

## 동기식 호출

분석단계에서의 조건 중 하나로 주문->취소 간의 호출은 트랜잭션으로 처리. 호출 프로토콜은 Rest Repository의 REST 서비스를 FeignClient 를 이용하여 호출.
- 쿠폰(Coupon) 서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```

package myProject_LSP.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

@FeignClient(name="coupon", url="${api.url.coupon}")
public interface CouponService {

    @RequestMapping(method= RequestMethod.POST, path="/coupons")
    public void couponSend(@RequestBody Coupon coupon);

}
```

- 주문이 접수 될 경우 Coupon 현황에 신규발행 내역을 접수한다.
```
@PrePersist
public void onPrePersist(){
   CookCancelled cookCancelled = new CookCancelled();
   BeanUtils.copyProperties(this, cookCancelled);
   cookCancelled.setStatus("COOK : ORDER CANCELED");
   cookCancelled.publishAfterCommit();
@PostPersist
public void onPostPersist(){
   if("COUPON : COUPON SENDED".equals(this.getStatus())){
      //ORDER -> COUPON SEND 경우
      CouponSended couponSended = new CouponSended()
      BeanUtils.copyProperties(this, couponSended);
      couponSended.publishAfterCommit();
    }
}
```
- COUPON 서비스에서 정상적으로 COUPON이 발행되고 STATUS도 'COUPON SENDED'로 접수된다.

![coupon](https://user-images.githubusercontent.com/69958878/93352458-d4023d00-f875-11ea-9f79-b54867858aa9.png)

</br>

## 비동기식 호출과 Saga Pattern

비동기식 호출 : 신규쿠폰 발행, 신규쿠폰 취소는 비동기식으로 처리하여 시스템 상황에 따라 발행 및 취소가 블로킹 되지 않도록 처리 한다. 
Saga Patter : 
1. 고객이 음식 주문을 접수하게 되면, Coupon에 신규발행이 접수된다.
1. Coupon이 신규발행이 완료되면, Coupon서비스에서 Order서비스에 Coupon 상태를 전송한다.
1. Order서비스는 Coupon 상태를 업데이트한다. (Order 서비스에서 Coupon 성공 인지)

 
```
# 고객이 음식 주문 시, Coupon에 신규발행 호출 (REST API)
@PostPersist
  public void onPostPersist(){
      Ordered ordered = new Ordered();
      BeanUtils.copyProperties(this, ordered);
      if(!"ORDER : COOK CANCELED".equals(ordered.getStatus())){
          ordered.publishAfterCommit();
          /*수정*/
          myProject_LSP.external.Coupon coupon = new myProject_LSP.external.Coupon();
          coupon.setOrderId(this.getId());
          coupon.setStatus("ORDER : COUPON SEND");
          OrderApplication.applicationContext.getBean(myProject_LSP.external.CouponService.class).couponSend(coupon);
      }

  }
```
- 주문 정상 접수
![orderPost](https://user-images.githubusercontent.com/69958878/93352415-c5b42100-f875-11ea-835d-35fddb0594fb.png) 


- 쿠폰 정상 발행
![coupon](https://user-images.githubusercontent.com/69958878/93352458-d4023d00-f875-11ea-9f79-b54867858aa9.png)  


```
# Coupon이 신규발행이 완료되면 Order 서비스에 Coupon 완료 정보 전송 (PUB/SUB)
 @PostPersist
  public void onPostPersist(){
      if("COUPON : COUPON SENDED".equals(this.getStatus())){
          //ORDER -> COUPON SEND 경우
          CouponSended couponSended = new CouponSended();
          BeanUtils.copyProperties(this, couponSended);
          couponSended.publishAfterCommit();
      }
  }
  
```

```
# 주문서비스의 쿠폰상태 설정 (PUB/SUB)
  @StreamListener(KafkaProcessor.INPUT)
  public void wheneverCouponSended_CouponInfoUpdate(@Payload CouponSended couponSended){

      if(couponSended.isMe()){
          System.out.println("##### listener CouponInfoUpdate : " + couponSended.toJson());
          Optional<Order> orderOptional = orderRepository.findById(couponSended.getOrderId());
          Order order = orderOptional.get();
          if("COUPON : COUPON SENDED".equals(couponSended.getStatus())){
              order.setCouponStatus("ORDER : COUPON SENDED SUCCESS");
          }
          orderRepository.save(order);
      }
  }
```
- ORDER 서비스에서 쿠폰 상태 업데이트 확인 (ORDER 서비스에 쿠폰상태가 필요 없지만, SAGA 패턴 확인을 위함)
![orderPostResut](https://user-images.githubusercontent.com/69958878/93352443-cf3d8900-f875-11ea-9079-3ebce9c415d3.png)


## Gateway
하나의 접점으로 서비스를 관리할 수 있는 Gateway를 통한 서비스라우팅을 적용 한다. Loadbalancer를 이용한 각 서비스의 접근을 확인 함.

```
# Gateway 설정(https://github.com/ladyfirst15/lsp-cna-gateway/blob/master/target/classes/application.yml)
spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: order
          uri: http://order:8080
          predicates:
            - Path=/orders/**
        - id: cook
          uri: http://cook:8080
          predicates:
            - Path=/cooks/**,/cancellations/**
        - id: delivery
          uri: http://delivery:8080
          predicates:
            - Path=/deliveries/**
        - id: mypage
          uri: http://mypage:8080
          predicates:
            - Path= /mypages/**
        - id: coupon
          uri: http://coupon:8080
          predicates:
            - Path= /couponS/**
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

server:
  port: 8080
```
![gateway](https://user-images.githubusercontent.com/69958878/93355435-07929680-f879-11ea-8b9f-8844fb57565e.png)
![gateway-w](https://user-images.githubusercontent.com/69958878/93355443-095c5a00-f879-11ea-97b5-c98ebd046bcc.png)

</br>

## CQRS
기존 코드에 영향도 없이 mypage 용 materialized view 구성한다. 고객은 주문 접수, 요리 상태, 배송현황, 쿠폰발행 등을 한개의 페이지에서 확인 할 수 있게 됨.</br>

```
# 주문 내역 mypage에 insert
   @StreamListener(KafkaProcessor.INPUT)
    public void whenOrdered_then_CREATE_1 (@Payload Ordered ordered) {
        try {
            if (ordered.isMe()) {
                // view 객체 생성
                Mypage mypage = new Mypage();
                // view 객체에 이벤트의 Value 를 set 함
                mypage.setRestaurantId(ordered.getRestaurantId());
                mypage.setRestaurantMenuId(ordered.getRestaurantMenuId());
                mypage.setCustomerId(ordered.getCustomerId());
                mypage.setQty(ordered.getQty());
                mypage.setOrderId(ordered.getId());
                mypage.setOrderStatus(ordered.getStatus());
                // view 레파지 토리에 save
                mypageRepository.save(mypage);
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }
```
```
# 쿠폰발행 발행(CouponSend) mypage 업데이트
    @StreamListener(KafkaProcessor.INPUT)
    public void whenCouponSended_then_UPDATE_6(@Payload CouponSended couponSended) {
        try {
            if (couponSended.isMe()) {
                // view 객체 조회
                List<Mypage> mypageList = mypageRepository.findByOrderId(couponSended.getOrderId());
                for(Mypage mypage : mypageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    mypage.setCouponId(couponSended.getId());
                    mypage.setCouponStatus(couponSended.getStatus());
                    mypage.setCouponSendDate(couponSended.getSendDate());
                    mypage.setCouponKind(couponSended.getCouponKind());
                    // view 레파지 토리에 save
                    mypageRepository.save(mypage);
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }
```
- mypage에 COUPON 발행상태 업데이트 확인
![mypages](https://user-images.githubusercontent.com/69958878/93352476-d95f8780-f875-11ea-8e55-b7aadcac59a1.png)

```
# 쿠폰발행 발행취소(CouponSendCancel) mypage 업데이트
    @StreamListener(KafkaProcessor.INPUT)
    public void whenCouponSendCancelled_then_UPDATE_7(@Payload CouponSendCancelled couponSendCancelled) {
        try {
            if (couponSendCancelled.isMe()) {
                // view 객체 조회
                List<Mypage> mypageList = mypageRepository.findByOrderId(couponSendCancelled.getOrderId());
                for(Mypage mypage : mypageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    mypage.setCouponId(couponSendCancelled.getId());
                    mypage.setCouponStatus(couponSendCancelled.getStatus());
                    mypage.setCouponSendDate(couponSendCancelled.getSendDate());
                    mypage.setCouponKind(couponSendCancelled.getCouponKind());
                    // view 레파지 토리에 save
                    mypageRepository.save(mypage);
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }


 ```
 - mypage에 COUPON 회수(삭제)상태 업데이트 확인
![mypages_cancel](https://user-images.githubusercontent.com/69958878/93354138-a6b68e80-f877-11ea-87e0-656dc1f5bc9c.png)
![cqrs](https://user-images.githubusercontent.com/54210936/93281210-987c5a00-f806-11ea-835b-2cea09bf6466.png)

</br>
</br>


# 운영

## AWS를 활용한 코드 자동빌드 배포 환경구축

  * AWS codebuild를 설정하여 github이 업데이트 되면 자동으로 빌드 및 배포 작업이 이루어짐.
  * Github에 Codebuild를 위한 yml 파일을 업로드하고, codebuild와 연동 함
  * 각 마이크로서비스의 build 스펙
  ```
    https://github.com/ladyfirst15/lsp-cna-order/blob/master/buildspec.yml
    https://github.com/ladyfirst15/lsp-cna-cook/blob/master/buildspec.yml
    https://github.com/ladyfirst15/lsp-cna-delivery/blob/master/buildspec.yml
    https://github.com/ladyfirst15/lsp-cna-gateway/blob/master/buildspec.yml
    https://github.com/ladyfirst15/lsp-cna-mypage/blob/master/buildspec.yml
    https://github.com/ladyfirst15/lsp-cna-coupon/blob/master/buildspec.yml

  ```
  
</br>

## 서킷 브레이킹과 오토스케일

* 서킷 브레이킹 :
주문이 과도할 경우 CB 를 통하여 장애격리. 500 에러가 5번 발생하면 10분간 CB 처리하여 100% 접속 차단
```
# AWS codebuild에 설정(https://github.com/ladyfirst15/lsp-cna-coupon/blob/master/buildspec.yml)
 http:
   http1MaxPendingRequests: 1   # 연결을 기다리는 request 수를 1개로 제한 (Default 
   maxRequestsPerConnection: 1  # keep alive 기능 disable
 outlierDetection:
  consecutiveErrors: 1          # 5xx 에러가 5번 발생하면
  interval: 1s                  # 1초마다 스캔 하여
  baseEjectionTime: 10m         # 10분 동안 circuit breaking 처리   
  maxEjectionPercent: 100       # 100% 로 차단
```

* 오토스케일(HPA) :
CPU사용률 10% 초과 시 replica를 3개까지 확장해준다. 상용에서는 70%로 세팅하지만 여기에서는 기능적용 확인을 위해 수치를 조절.
```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: skcchpa-coupon
  namespace: teamc
  spec:
    scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: $_PROJECT_NAME                # order (주문) 서비스 HPA 설정
    minReplicas: 1                      # 최소 1개
    maxReplicas: 3                      # 최대 3개
    targetCPUUtilizationPercentage: 10  # cpu사용율 10프로 초과 시 
```    
* 부하테스트(Siege)를 활용한 부하 적용 후 서킷브레이킹 / 오토스케일 내역을 확인한다.
1. 부하테스트 전, minReplicas=1로 1개의 pod만이 떠있는 것을 확인
![auto_1](https://user-images.githubusercontent.com/69958878/93374959-6d3f4c80-f892-11ea-8d55-88f169967738.png)
2. 부하테스트하여, 오토스케일링 된 pod 내역 확인
![503](https://user-images.githubusercontent.com/69958878/93374920-5dc00380-f892-11ea-9a46-e32c9a464361.jpg)
![auto_result](https://user-images.githubusercontent.com/69958878/93374971-716b6a00-f892-11ea-937d-eb1c72b586a4.png)




## 무정지 배포

* 무정지 배포를 위해 ECR 이미지를 업데이트 하고 이미지 체인지를 시도 함. Github에 소스가 업데이트 되면 자동으로 AWS CodeBuild에서 컴파일 하여 이미지를 ECR에 올리고 EKS에 반영.
  이후 아래 옵션에 따라 무정지 배포 적용 된다.
  

```
# AWS codebuild에 설정(https://github.com/ladyfirst15/final-cna-coupon/blob/master/buildspec.yml)
  spec:
    replicas: 3
    minReadySeconds: 10   # 최소 대기 시간 10초
    strategy:
      type: RollingUpdate
      rollingUpdate:
      maxSurge: 1         # 1개씩 업데이트 진행
      maxUnavailable: 0   # 업데이트 프로세스 중에 사용할 수 없는 최대 파드의 수

```

- 새버전으로의 배포 시작(V1로 배포)
![ZeroDownTime  console - pod change status](https://user-images.githubusercontent.com/54210936/93277970-4c2d1c00-f7fe-11ea-87ce-82cdd77e84ac.jpg)

- siege를 이용한 부하 적용. Availability가 100% 미만으로 떨어짐. 쿠버네티스가 새로 올려진 서비스를 Ready 상태로 인식하여 서비스 유입을 진행 하였음. Readiness Probe 설정하여 조치 필요.
![무중단_100](https://user-images.githubusercontent.com/69958878/93379258-8b0fb000-f898-11ea-98e4-9aeb99049184.png)

- 새버전 배포 확인(V1 적용)
![ZeroDownTime  console - pod describe](https://user-images.githubusercontent.com/54210936/93278015-6d8e0800-f7fe-11ea-82d1-dc80b96b601c.jpg)


- Readiness Probe 설정을 통한 ZeroDownTime 설정.
```
  readinessProbe:
    tcpSocket:
      port: 8080
      initialDelaySeconds: 180      # 서비스 어플 기동 후 180초 뒤 시작
      periodSeconds: 120            # 120초 주기로 readinessProbe 실행 
```
![ZeroDownTime  SEIGE_STATUS_read](https://user-images.githubusercontent.com/54210936/93278989-1473a380-f801-11ea-8140-f7edbc2c9b6f.jpg)


</br>

## 마이크로서비스 로깅 관리를 위한 PVC 설정
AWS의 EFS에 파일시스템을 생성(EFS-teamc (fs-28564909))하고 서브넷과 클러스터(TeamC-final)를 연결하고 PVC를 설정해준다. 각 마이크로 서비스의 로그파일이 EFS에 정상적으로 생성되고 기록됨을 확인 함.
```
#AWS의 각 codebuild에 설정(https://github.com/ladyfirst15/lsp-cna-coupon/blob/master/buildspec.yml)
volumeMounts:  
- mountPath: "/mnt/aws"    # COUPON 서비스 로그파일 생성 경로
  name: volume                 
volumes:                                # 로그 파일 생성을 위한 EFS, PVC 설정 정보
- name: volume
  persistentVolumeClaim:
  claimName: aws-efs  
```
![pvc](https://user-images.githubusercontent.com/69958878/93348772-9c919180-f871-11ea-8a1a-f8ae375437a4.png)
</br>

## SelfHealing
운영 안정성의 확보를 위해 마이크로서비스가 아웃된 뒤에 다시 프로세스가 올라오는 환경을 구축한다. 프로세스가 죽었을 때 다시 기동됨을 확인함.
```
#AWS의 각 codebuild에 설정(https://github.com/ladyfirst15/lsp-cna-coupon/blob/master/buildspec.yml)
livenessProbe:
  exec:
    command:
    - cat
    - /mnt/aws/logs/coupon-application.log
  initialDelaySeconds: 20      # 서비스 어플 기동 후 20초 뒤 시작
  periodSeconds: 30            # 30초 주기로 livenesProbe 실행 
```
- /mnt/aws/logs/coupon-application.log 가 삭제 될 때, coupon 프로세스가 kill된 후에, 자동으로 SelfHealing되는 과정 확인
![liveness_delete](https://user-images.githubusercontent.com/69958878/93369584-89d78680-f88a-11ea-8d04-d8b189abe407.png)
![liveness](https://user-images.githubusercontent.com/69958878/93369578-880dc300-f88a-11ea-93b3-0df5822e973e.png)
</br>
</br>



# 첨부
팀프로젝트 구성을 위해 사용한 계정 정보 및 클러스터 명, Github 주소 등의 내용 공유 
* AWS 계정 명 : admin14
```
Region : ap-northeast-1
EFS : EFS-teamc (fs-96929df7)
EKS : TeamC-final
ECR : order / delivery / cook / mypage / gateway
Codebuild : order / delivery / cook / mypage / gateway
```
* Github :</br>
```
https://github.com/ladyfirst15/lsp-cna-gateway
https://github.com/ladyfirst15/lsp-cna-order
https://github.com/ladyfirst15/lsp-cna-delivery
https://github.com/ladyfirst15/lsp-cna-cook
https://github.com/ladyfirst15/lsp-cna-mypage
```
