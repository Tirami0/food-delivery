# 서비스 시나리오
![image](https://user-images.githubusercontent.com/118098096/205941104-93752bcc-f25e-41d6-a50d-05e76719242f.png)

기능적 요구사항
1. 고객이 메뉴를 선택하여 주문한다.
2. 고객이 선택한 메뉴에 대해 결제한다.
3. 주문이 되면 주문 내역이 입점상점주인에게 주문정보가 전달된다
4. 상점주는 주문을 수락하거나 거절할 수 있다
5. 상점주는 요리시작때와 완료 시점에 시스템에 상태를 입력한다
6. 고객은 아직 요리가 시작되지 않은 주문은 취소할 수 있다
7. 요리가 완료되면 고객의 지역 인근의 라이더들에 의해 배송건 조회가 가능하다
8. 라이더가 해당 요리를 Pick한 후, 앱을 통해 통보한다.
9. 고객이 주문상태를 중간중간 조회한다
10. 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다
11. 고객이 요리를 배달 받으면 배송확인 버튼을 탭하여, 모든 거래가 완료된다
12. (추가사항)고객이 상점의 배달된 요리 평가점수를 등록한다.
13. (추가사항)상점에서 요리 평가 점수 조회가 가능하다.
![image](https://user-images.githubusercontent.com/118098096/205945787-f8027e69-0c0f-4d48-b483-96a3a7d706b2.png)

비기능적 요구사항

1. 장애격리
    1. 상점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency
    2. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 Circuit breaker, fallback
2. 성능
    1. 고객이 자주 상점관리에서 확인할 수 있는 배달상태를 주문시스템(프론트엔드)에서 확인할 수 있어야 한다 CQRS
    2. 배달상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다 Event driven



# 체크포인트
1. Saga (Pub/Sub)
2. CQRS
3. Compensation/Correlation
4. Request/Response
5. Circuit Breaker
6. Gateway/Ingress

# Saga(Pub/Sub)
foodCookings.java 파일에 orderPlaced 이벤트 발생시 구현부
```
    public static void updateStatus(OrderPlaced orderPlaced){

        FoodCooking foodCooking = new FoodCooking();
        foodCooking.setOrderId(orderPlaced.getOrderId());
        foodCooking.setAddress(orderPlaced.getAddress());
        foodCooking.setFoodId(orderPlaced.getFoodId());
        foodCooking.setStatus("ordered");
        repository().save(foodCooking);        
    }
```

주문 

```
itpod /workspace/mall2/order (main) $ http :8081/orders orderId=1 foodId="짜장면" address="인천"
HTTP/1.1 201 
Connection: keep-alive
Content-Type: application/json
Date: Wed, 07 Dec 2022 14:16:52 GMT
Keep-Alive: timeout=60
Location: http://localhost:8081/orders/1
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/1"
        },
        "self": {
            "href": "http://localhost:8081/orders/1"
        }
    },
    "address": "인천",
    "foodId": "짜장면",
    "payed": null,
    "rate": null,
    "status": null
}
```
store에 foodCooking 정보
```
gitpod /workspace/mall2/store (main) $ http :8084/foodCookings
HTTP/1.1 200 
Connection: keep-alive
Content-Type: application/hal+json
Date: Wed, 07 Dec 2022 14:17:58 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_embedded": {
        "foodCookings": [
            {
                "_links": {
                    "foodCooking": {
                        "href": "http://localhost:8084/foodCookings/1"
                    },
                    "self": {
                        "href": "http://localhost:8084/foodCookings/1"
                    }
                },
                "address": "인천",
                "foodId": "짜장면",
                "rate": null,
                "status": "ordered"
            }
        ]
    },
    "_links": {
        "profile": {
            "href": "http://localhost:8084/profile/foodCookings"
        },
        "self": {
            "href": "http://localhost:8084/foodCookings"
        }
    },
    "page": {
        "number": 0,
        "size": 20,
        "totalElements": 1,
        "totalPages": 1
    }
}
```   

    

# CQRS

customer myPage의 CQRS 설정


![image](https://user-images.githubusercontent.com/118098096/206208962-50309fef-c94f-40bf-9170-cb98f5fd2c88.png)

소스 구현
```
    @StreamListener(KafkaProcessor.INPUT)
    public void whenOrderPlaced_then_CREATE_1 (@Payload OrderPlaced orderPlaced) {
        try {

            if (!orderPlaced.validate()) return;

            // view 객체 생성
            MyPage myPage = new MyPage();
            // view 객체에 이벤트의 Value 를 set 함
            myPage.setOrderId(orderPlaced.getOrderId());
            myPage.setAddress(orderPlaced.getAddress());
            myPage.setFoodId(orderPlaced.getFoodId());
            myPage.setStatus("주문됨");
            // view 레파지 토리에 save
            myPageRepository.save(myPage);

        }catch (Exception e){
            e.printStackTrace();
        }
    }
```    
주문 후 현황
```
gitpod /workspace/mall2/customer (main) $ http :8083/myPages
HTTP/1.1 200 
Connection: keep-alive
Content-Type: application/hal+json
Date: Wed, 07 Dec 2022 14:17:00 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_embedded": {
        "myPages": [
            {
                "_links": {
                    "myPage": {
                        "href": "http://localhost:8083/myPages/1"
                    },
                    "self": {
                        "href": "http://localhost:8083/myPages/1"
                    }
                },
                "address": "인천",
                "foodId": "짜장면",
                "payed": null,
                "rate": null,
                "status": "주문됨"
            }
        ]
    },
    "_links": {
        "profile": {
            "href": "http://localhost:8083/profile/myPages"
        },
        "search": {
            "href": "http://localhost:8083/myPages/search"
        },
        "self": {
            "href": "http://localhost:8083/myPages"
        }
    },
    "page": {
        "number": 0,
        "size": 20,
        "totalElements": 1,
        "totalPages": 1
    }
}
```


# Compensation/Correaltion
주문 후 취소시 FoodCooking 상태 변경
구현
```
    public static void updateStatus(OrderCanceld orderCanceld){

            repository().findById(orderCanceld.getOrderId()).ifPresent(foodCooking->{            
            foodCooking.setStatus("canceled"); // do something
            repository().save(foodCooking);

           });

```

주문
```
gitpod /workspace/mall2/order (main) $ http :8081/orders  foodId="짬뽕" address="인천"
HTTP/1.1 201 
Connection: keep-alive
Content-Type: application/json
Date: Wed, 07 Dec 2022 16:40:11 GMT
Keep-Alive: timeout=60
Location: http://localhost:8081/orders/1
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/1"
        },
        "self": {
            "href": "http://localhost:8081/orders/1"
        }
    },
    "address": "인천",
    "foodId": "짬뽕",
    "payed": null,
    "rate": null,
    "status": null
}
```
취소
```
gitpod /workspace/mall2/order (main) $ http DELETE :8081/orders/1
HTTP/1.1 204 
Connection: keep-alive
Date: Wed, 07 Dec 2022 16:41:56 GMT
Keep-Alive: timeout=60
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
```
foodCookings 확인
```

gitpod /workspace/mall2/store (main) $ http :8084/foodCookings
HTTP/1.1 200 
Connection: keep-alive
Content-Type: application/hal+json
Date: Wed, 07 Dec 2022 16:42:01 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_embedded": {
        "foodCookings": [
            {
                "_links": {
                    "foodCooking": {
                        "href": "http://localhost:8084/foodCookings/1"
                    },
                    "self": {
                        "href": "http://localhost:8084/foodCookings/1"
                    }
                },
                "address": "인천",
                "foodId": "짬뽕",
                "rate": null,
                "status": "canceled"
            }
        ]
    },
    "_links": {
        "profile": {
            "href": "http://localhost:8084/profile/foodCookings"
        },
        "self": {
            "href": "http://localhost:8084/foodCookings"
        }
    },
    "page": {
        "number": 0,
        "size": 20,
        "totalElements": 1,
        "totalPages": 1
    }
}
```

# Request/Response
주문 수락/거절 controller 방식 설정

![image](https://user-images.githubusercontent.com/118098096/206246970-9b607981-e39e-4767-8db4-e809111165e5.png)


소스 구현 : accept 변수 값에 따라 이벤트 발생 및 상태 
```
    public void accept(AcceptCommand acceptCommand){

        if(acceptCommand.getAccept())
        {
            OrderAccepted orderAccepted = new OrderAccepted(this);
            orderAccepted.publishAfterCommit();
            setStatus("접수됨");
        }
        else
        {
            OrderRejected orderRejected = new OrderRejected(this);
            orderRejected.publishAfterCommit();
            setStatus("취소됨");
        }

    }
```

주문 
```
gitpod /workspace/mall3/store (main) $ http PUT localhost:8084/foodCookings/1/accept accept=true
HTTP/1.1 200 
Connection: keep-alive
Content-Type: application/json;charset=UTF-8
Date: Wed, 07 Dec 2022 17:12:03 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked

{
    "address": null,
    "foodId": "짬뽕",
    "orderId": 1,
    "rate": null,
    "status": "접수됨",
    "stock": null
}
```
주문 거절
```
gitpod /workspace/mall3/store (main) $ http PUT localhost:8084/foodCookings/1/accept accept=false
HTTP/1.1 200 
Connection: keep-alive
Content-Type: application/json;charset=UTF-8
Date: Wed, 07 Dec 2022 17:12:11 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked

{
    "address": null,
    "foodId": "짬뽕",
    "orderId": 1,
    "rate": null,
    "status": "취소됨",
    "stock": null
}
```

# Circuit Breaker
store 의 application.yml 파일에 임계치 설정
```
server:
  port: 8080
feign:
  hystrix:
    enabled: true
hystrix:
  command:
    default:
      execution.isolation.thread.timeoutInMilliseconds: 500
spring:
  application:
    name: store
---

```
foodCooking.java에 delay 설정
```
    @PostLoad
    public void makeDelay(){
         try {
             Thread.currentThread().sleep((long) (400 + Math.random() * 220));
         } catch (InterruptedException e) {
             e.printStackTrace();
         }

    }
```

siege 설치
```
gitpod /workspace/mall4/store (main) $ sudo apt update -y
Hit:1 https://download.docker.com/linux/ubuntu focal InRelease
Hit:3 http://archive.ubuntu.com/ubuntu focal InRelease                                                        
Hit:4 http://security.ubuntu.com/ubuntu focal-security InRelease                                 
Hit:5 http://ppa.launchpad.net/git-core/ppa/ubuntu focal InRelease
Hit:6 http://archive.ubuntu.com/ubuntu focal-updates InRelease
Hit:7 http://archive.ubuntu.com/ubuntu focal-backports InRelease           
Hit:8 http://ppa.launchpad.net/ondrej/apache2/ubuntu focal InRelease       
Hit:2 https://apt.llvm.org/focal llvm-toolchain-focal-15 InRelease   
Hit:9 http://ppa.launchpad.net/ondrej/nginx-mainline/ubuntu focal InRelease
Hit:10 http://ppa.launchpad.net/ondrej/php/ubuntu focal InRelease
Reading package lists... Done
Building dependency tree       
Reading state information... Done
90 packages can be upgraded. Run 'apt list --upgradable' to see them.
gitpod /workspace/mall4/store (main) $ sudo apt install siege -y
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following NEW packages will be installed:
  siege
0 upgraded, 1 newly installed, 0 to remove and 90 not upgraded.
Need to get 98.3 kB of archives.
After this operation, 278 kB of additional disk space will be used.
Get:1 http://archive.ubuntu.com/ubuntu focal/main amd64 siege amd64 4.0.4-1build1 [98.3 kB]
Fetched 98.3 kB in 0s (265 kB/s) 
debconf: delaying package configuration, since apt-utils is not installed
Selecting previously unselected package siege.
(Reading database ... 36156 files and directories currently installed.)
Preparing to unpack .../siege_4.0.4-1build1_amd64.deb ...
Unpacking siege (4.0.4-1build1) ...
Setting up siege (4.0.4-1build1) ...
Processing triggers for man-db (2.9.1-1) ...
```
부하툴 사용
```
gitpod /workspace/mall4/order (main) $ siege -c2 -t10S  -v --content-type "application/json" 'http://localhost:8081/orders POST {"foodId":"짬뽕", "address":"인천"}'
** SIEGE 4.0.4
** Preparing 2 concurrent users for battle.
The server is now under siege...
HTTP/1.1 201     0.01 secs:     267 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.01 secs:     267 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.00 secs:     267 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.00 secs:     267 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.00 secs:     267 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.01 secs:     267 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.01 secs:     267 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.00 secs:     267 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.00 secs:     267 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.02 secs:     267 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.02 secs:     267 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.01 secs:     267 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.01 secs:     267 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.01 secs:     267 bytes ==> POST http://localhost:8081/orders
```

# Gateway/Ingress
application.yml
```
server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: order
          uri: http://localhost:8081
          predicates:
            - Path=/orders/**, 
        - id: rider
          uri: http://localhost:8082
          predicates:
            - Path=/deliveries/**, /deliveryInfos/**
        - id: customer
          uri: http://localhost:8083
          predicates:
            - Path=, /myPages/**
        - id: store
          uri: http://localhost:8084
          predicates:
            - Path=/foodCookings/**, /shopPages/**
        - id: frontend
          uri: http://localhost:8080
          predicates:
            - Path=/**
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
orders(8081) Gateway(8988) 사용
```
gitpod /workspace/mall2/order (main) $ http :8081/orders
HTTP/1.1 200 
Connection: keep-alive
Content-Type: application/hal+json
Date: Tue, 06 Dec 2022 15:09:27 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_embedded": {
        "orders": []
    },
    "_links": {
        "profile": {
            "href": "http://localhost:8081/profile/orders"
        },
        "self": {
            "href": "http://localhost:8081/orders"
        }
    },
    "page": {
        "number": 0,
        "size": 20,
        "totalElements": 0,
        "totalPages": 0
    }
}


gitpod /workspace/mall2/order (main) $ http :8088/orders
HTTP/1.1 200 OK
Content-Type: application/hal+json
Date: Tue, 06 Dec 2022 15:32:31 GMT
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
transfer-encoding: chunked

{
    "_embedded": {
        "orders": []
    },
    "_links": {
        "profile": {
            "href": "http://localhost:8081/profile/orders"
        },
        "self": {
            "href": "http://localhost:8081/orders"
        }
    },
    "page": {
        "number": 0,
        "size": 20,
        "totalElements": 0,
        "totalPages": 0
    }
}


gitpod /workspace/mall2/order (main) $ 
```


# 추가 요구사항 : 고객이 상점의 배달된 요리 평가점수를 등록한다.
![image](https://user-images.githubusercontent.com/118098096/206206980-b0971836-02f6-4752-bec1-2660463abd3a.png)

# 추가 요구사항 : 상점에서 요리 평가 점수 조회가 가능하다.
![image](https://user-images.githubusercontent.com/118098096/206207322-95fe5f88-d00e-48f4-9c1c-6726d380a9e6.png)
